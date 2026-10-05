# Troubleshooting K3s homelab theo từng lớp

Nguyên tắc: tìm hop đầu tiên bị lỗi, đọc trạng thái và log tại hop đó, rồi áp dụng thay đổi nhỏ nhất. Không reset K3s hoặc thay HAProxy trước khi có bằng chứng.

## 1. Hai data path cần phân biệt

```text
Kubernetes API:
kubectl tang3 -> HAProxy tang3:6443 -> K3s API tang4:6443

Application:
Client -> HAProxy tang3:80/443 -> ServiceLB tang4:80/443
       -> Traefik -> Ingress -> ClusterIP Service -> Pod
```

Tang3 không chạy K3s. Tang4 là một K3s server duy nhất, schedulable, không chạy `k3s-agent.service`.

## 2. Thu thập trạng thái ban đầu

Trên tang3:

```bash
cd /home/monitor/k3s-onprem
source /home/monitor/.venvs/k3s-ansible/bin/activate

ansible-inventory --graph
ansible all -m ping
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
kubectl --kubeconfig /home/monitor/.kube/config get pods -A -o wide
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Giá trị mong đợi:

- inventory chỉ có `server-tang4` trong nhóm K3s và `server-tang3` trong nhóm load balancer;
- cluster có đúng một node `server-tang4`, trạng thái `Ready`;
- `power-grafana`, `power-prometheus` và `k3s-haproxy` trên tang3 vẫn chạy.

## 3. Kubernetes API không truy cập được

Kiểm tra từ ngoài vào trong:

```bash
# tang3: endpoint dùng bởi kubeconfig
kubectl --kubeconfig /home/monitor/.kube/config config view --minify
curl -ksS https://192.168.30.45:6443/ping

# tang3: HAProxy
sudo ss -lntp | grep ':6443'
docker logs --tail 100 k3s-haproxy
docker exec k3s-haproxy haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
grep -n '192.168.30.35:6443' /home/monitor/k3s-haproxy/haproxy.cfg

# kiểm tra backend trực tiếp
curl -ksS https://192.168.30.35:6443/ping
```

Kết quả `/ping` mong đợi là `pong`. Nếu backend trực tiếp khỏe nhưng endpoint tang3 lỗi, root cause nằm ở HAProxy/listener/firewall trên tang3. Nếu cả hai lỗi, kiểm tra K3s trên tang4:

```bash
sudo systemctl is-active k3s
sudo systemctl status k3s --no-pager
sudo journalctl -u k3s --since '-15 min' --no-pager
sudo ss -lntp | grep ':6443'
sudo k3s kubectl get --raw=/readyz
```

Không dùng log của `k3s-agent`: sau recovery, service hợp lệ trên tang4 là `k3s.service`.

## 4. Application/Ingress không truy cập được

Test không phụ thuộc DNS:

```bash
curl -v -H 'Host: nginx.onprem.site' http://192.168.30.45/
curl -v -H 'Host: app1.onprem.site' http://192.168.30.45/
```

Sau đó đi theo từng lớp.

### HAProxy trên tang3

```bash
docker ps --filter name=k3s-haproxy
docker logs --tail 100 k3s-haproxy
sudo ss -lntp | grep -E ':(80|443|6443)\s'
sed -n '1,220p' /home/monitor/k3s-haproxy/haproxy.cfg
```

Backend mong đợi:

```text
:6443 -> 192.168.30.35:6443
:80   -> 192.168.30.35:80
:443  -> 192.168.30.35:443
```

### ServiceLB và Traefik trên tang4

```bash
nc -vz 192.168.30.35 80
nc -vz 192.168.30.35 443

kubectl --kubeconfig /home/monitor/.kube/config -n kube-system get svc traefik -o wide
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system get pods -o wide
kubectl --kubeconfig /home/monitor/.kube/config get node server-tang4 --show-labels
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system rollout status deployment/traefik --timeout=180s
```

Tang4 phải có `enablelb=true` và `lbpool=ingress`; ServiceLB của Traefik phải chạy trên chính tang4.

### Ingress, Service và Pod

```bash
kubectl --kubeconfig /home/monitor/.kube/config get ingress -A
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get service,endpointslice,pod -o wide
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get events --sort-by=.lastTimestamp
```

Diễn giải nhanh:

| Kết quả | Kiểm tra tiếp |
| --- | --- |
| timeout/refused tại tang3 | listener HAProxy, route, firewall |
| HAProxy backend down | port tương ứng trên tang4 và K3s/ServiceLB |
| Traefik `404` | `Host`, path và Ingress rule |
| `502/503` | Service selector, EndpointSlice, readiness |
| `ImagePullBackOff` | image/tag, GHCR visibility, DNS/outbound Internet |
| Pod `Pending` | node selector, taints và tài nguyên tang4 |

Không đổi Service sang NodePort để che lỗi selector; kiến trúc chủ đích dùng ClusterIP sau Traefik.

## 5. Ansible không kết nối được

```bash
ansible-inventory --graph
ansible all -m ping
ansible server-tang4 -m setup -a 'filter=ansible_all_ipv4_addresses'
ansible server-tang3 -m setup -a 'filter=ansible_all_ipv4_addresses'
```

Kiểm tra lần lượt IP inventory, LAN route, SSH daemon, user/key, host-key fingerprint và sudo. Không tắt host-key checking để bỏ qua một thay đổi định danh máy chưa được xác minh.

## 6. Recovery playbook từ chối chạy

`playbooks/rebuild-single-node.yml` cố ý fail-closed. Nó chỉ chạy khi:

- có `-e confirm_k3s_rebuild=true`;
- target là `server-tang4`, IP `192.168.30.35`, interface LAN đúng;
- chưa có `k3s.service` hoạt động;
- không có Docker, CRI-O hoặc standalone containerd active;
- tồn tại official `/usr/local/bin/k3s-agent-uninstall.sh`.

Nếu guard fail, đọc message và kiểm tra thực tế. Không sửa guard chỉ để ép playbook chạy. Nếu phát hiện workload ngoài K3s, phải lập phương án backup/migration riêng trước cleanup.

Sau khi recovery thành công:

```bash
sudo systemctl is-active k3s
sudo systemctl is-enabled k3s
sudo systemctl is-active k3s-agent || true
sudo k3s kubectl get nodes -o wide
```

Mong đợi `k3s=active/enabled`, `k3s-agent=inactive/not-found` và chỉ có node `server-tang4`.

## 7. Bảo vệ monitoring trên tang3

Repository chỉ quản lý HAProxy container; không sở hữu lifecycle của Prometheus/Grafana và các container khác trên tang3. Trước và sau thay đổi:

```bash
docker ps --filter name=power-grafana
docker ps --filter name=power-prometheus
docker ps --filter name=k3s-haproxy
```

Không chạy `docker system prune`, không xóa volume/network và không recreate monitoring stack trong quá trình recovery K3s.

## 8. IP hoặc hostname thay đổi

Inventory hiện cố định:

- tang3: `192.168.30.45`;
- tang4: `192.168.30.35`.

Nếu DHCP thay đổi, xác minh IP thực trên host trước, sau đó cập nhật inventory và converge lại. Kubeconfig của tang3 vẫn trỏ tới HAProxy `192.168.30.45:6443`; public/hosts DNS cũng phải giữ hostname ứng dụng trỏ tới tang3.

## 9. Xác nhận sau khi sửa

```bash
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
kubectl --kubeconfig /home/monitor/.kube/config get pods -A -o wide
curl -ksS https://192.168.30.45:6443/ping
curl -H 'Host: nginx.onprem.site' http://192.168.30.45/
ansible-playbook playbooks/validate.yml
```

Cuối cùng chạy lại playbook liên quan để kiểm tra idempotency. Chỉ kết luận hoàn tất khi direct backend, HAProxy path, Kubernetes state và HTTP smoke test đều có bằng chứng thành công.
