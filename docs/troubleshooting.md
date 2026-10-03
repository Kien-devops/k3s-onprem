# Troubleshooting K3s Homelab theo từng lớp

Nguyên tắc chính: tìm hop đầu tiên không hoạt động, xác định root cause tại hop đó, rồi áp dụng thay đổi nhỏ nhất. Không reset K3s, sửa APT, recreate toàn bộ cluster hoặc đổi HAProxy khi chưa chứng minh lỗi nằm ở lớp tương ứng.

## 1. Quy trình chẩn đoán

1. Xác định expected behavior: API, smoke app hay demo app cần trả gì.
2. Reproduce bằng một lệnh cụ thể và ghi status/error.
3. Đi theo data path từ ngoài vào trong.
4. Ở mỗi lớp, kiểm tra listener/state, config và log/event.
5. Dừng tại lớp đầu tiên fail; các lớp phía sau chưa thể kết luận.
6. Sửa nguyên nhân nhỏ nhất.
7. Test lại từ lớp lỗi, sau đó test end-to-end.
8. Chạy `playbooks/validate.yml` để phát hiện regression.

Hai data path cần phân biệt:

```text
Kubernetes API:
kubectl -> HAProxy tang3:6443 -> K3s API tang2:6443

Application:
Client -> HAProxy tang3:80/443 -> worker ports -> ServiceLB
       -> Traefik -> Ingress -> Service -> Pod
```

## 2. Thu thập trạng thái ban đầu

Trên `server-tang3`:

```bash
cd /home/monitor/k3s-onprem
source /home/monitor/.venvs/k3s-ansible/bin/activate

ansible all -m ping
ansible-inventory --graph
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
kubectl --kubeconfig /home/monitor/.kube/config get pods -A -o wide
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Không sửa gì ở bước này. Mục tiêu là xác định lỗi thuộc Ansible connectivity, API path, ingress path hay workload.

## 3. Application path: Client đến Pod

Chẩn đoán đúng thứ tự sau:

```text
Client
  -> HAProxy
  -> tang4 ports 80/443
  -> ServiceLB
  -> Traefik
  -> Ingress
  -> ClusterIP Service
  -> Pod
```

### Lớp 1 - Client, DNS/hosts và LAN route

Test không phụ thuộc DNS:

```bash
curl -v -H 'Host: nginx.onprem.site' http://192.168.30.45/
```

Trên Windows:

```powershell
findstr /C:"nginx.onprem.site" "$env:SystemRoot\System32\drivers\etc\hosts"
ping -n 1 192.168.30.45
ping -n 1 nginx.onprem.site
curl.exe -v http://nginx.onprem.site/
```

Hosts file cần có:

```text
192.168.30.45 nginx.onprem.site
```

Diễn giải:

- Request theo IP kèm `Host` thành công nhưng theo hostname thất bại: lỗi name resolution/hosts file.
- Cả IP và hostname đều timeout: kiểm tra route/firewall/LAN trước.
- Hostname resolve đúng IP nhưng TCP timeout: hosts file đã đúng; lỗi không nằm ở DNS.
- HTTP trả response nhưng sai app: kiểm tra `Host` header và Ingress rule.

### Lớp 2 - HAProxy trên tang3

Trên tang3:

```bash
docker ps --filter name=k3s-haproxy
docker logs --tail 100 k3s-haproxy
sudo ss -lntp | grep -E ':(80|443|6443)\s'
docker exec k3s-haproxy haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
```

Kiểm tra backend trong file managed:

```bash
sed -n '1,220p' /home/monitor/k3s-haproxy/haproxy.cfg
```

Giá trị mong đợi:

```text
:6443 -> 192.168.30.44:6443
:80   -> 192.168.30.35:80
:443  -> 192.168.30.35:443
```

Nếu HAProxy không listen, kiểm tra container state và log. Nếu backend báo down, tiếp tục kiểm tra tang4 ports trước khi sửa HAProxy. Không thêm backend riêng cho Nginx; application hostname do Traefik xử lý.

### Lớp 3 - Worker host ports trên tang4

Từ tang3, kiểm tra TCP tới worker:

```bash
nc -vz 192.168.30.35 80
nc -vz 192.168.30.35 443
```

Trên tang4:

```bash
sudo ss -lntp | grep -E ':(80|443)\s'
sudo systemctl is-active k3s-agent
sudo journalctl -u k3s-agent --since '-15 min' --no-pager
```

Nếu HAProxy listener khỏe nhưng tang4 không nhận `80/443`, lỗi thường nằm ở ServiceLB placement, agent, host firewall hoặc port conflict. Chưa cần kiểm tra application Pod ở giai đoạn này.

### Lớp 4 - ServiceLB

Chạy từ tang3 với kubeconfig:

```bash
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system get svc traefik -o wide
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system get pods -l svccontroller.k3s.cattle.io/svcname=traefik -o wide
kubectl --kubeconfig /home/monitor/.kube/config get node server-tang4 --show-labels
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system describe svc traefik
```

Mong đợi:

- Traefik Service là `LoadBalancer`;
- ServiceLB Pod ở trạng thái `Running` trên `server-tang4`;
- tang4 có `enablelb=true` và `lbpool=ingress`;
- Service có label pool `ingress` do `HelmChartConfig` tạo.

Kiểm tra cluster configuration:

```bash
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system get helmchartconfig traefik -o yaml
```

Nếu ServiceLB Pod nằm sai node hoặc không tồn tại, đối chiếu node labels, Service labels và event trước. Có thể reconcile đúng phạm vi bằng:

```bash
ansible-playbook playbooks/addons.yml
```

### Lớp 5 - Traefik Ingress Controller

```bash
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system get deployment traefik
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system get pods -l app.kubernetes.io/name=traefik -o wide
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system rollout status deployment/traefik --timeout=180s
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system logs deployment/traefik --tail=100
```

Traefik Pod phải `Running` và nằm trên tang4. Một HTTP `404` từ Traefik thường cho thấy network path đến Traefik đã hoạt động nhưng không có Ingress rule khớp `Host`/path. Timeout thường nằm ở lớp trước Traefik.

### Lớp 6 - Ingress rule

Demo app:

```bash
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get ingress nginx -o wide
kubectl --kubeconfig /home/monitor/.kube/config -n public-app describe ingress nginx
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get ingress nginx -o yaml
```

Mong đợi:

```text
ingressClassName: traefik
host: nginx.onprem.site
path: /
backend service: nginx, port http
```

Kiểm tra client gửi đúng `Host` header. HAProxy chạy TCP mode nên giữ nguyên HTTP header; Traefik cần header này để match rule.

Nếu Ingress không tồn tại nhưng Deployment/Service có, chạy lại standalone app playbook:

```bash
ansible-playbook playbooks/app.yml
```

### Lớp 7 - ClusterIP Service và endpoints

```bash
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get service nginx -o wide
kubectl --kubeconfig /home/monitor/.kube/config -n public-app describe service nginx
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get endpointslices -l kubernetes.io/service-name=nginx -o wide
```

Mong đợi:

- Service type là `ClusterIP`;
- selector là `app=nginx`;
- port `80` trỏ tới target port tên `http`;
- EndpointSlice chứa IP của hai Ready Pods.

Nếu Service có `ENDPOINTS <none>`, so sánh selector với Pod labels:

```bash
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get service nginx -o jsonpath='{.spec.selector}'
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get pods --show-labels
```

Không đổi Service sang NodePort để che lỗi selector. Kiến trúc chủ đích dùng ClusterIP phía sau Traefik.

### Lớp 8 - Deployment và Pod

```bash
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get deployment nginx
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get pods -l app=nginx -o wide
kubectl --kubeconfig /home/monitor/.kube/config -n public-app describe deployment nginx
kubectl --kubeconfig /home/monitor/.kube/config -n public-app describe pod -l app=nginx
kubectl --kubeconfig /home/monitor/.kube/config -n public-app logs -l app=nginx --tail=100
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get events --sort-by=.lastTimestamp
```

Mong đợi hai Pod `1/1 Running`, `Ready=True`, đều ở `server-tang4`.

Một số trạng thái thường gặp:

| Trạng thái | Hướng kiểm tra |
| --- | --- |
| `Pending` | nodeSelector, taints, scheduler events, tài nguyên worker |
| `ImagePullBackOff` | DNS/internet từ tang4, registry tag/rate limit |
| `CrashLoopBackOff` | container logs, command/config, filesystem mount |
| `Running` nhưng `0/1` | readiness probe và application response |
| Một replica Ready | describe Pod lỗi và kiểm tra capacity tang4 |

## 4. Phân biệt lỗi bằng HTTP response

| Kết quả client | Khả năng cao |
| --- | --- |
| DNS không resolve | Windows hosts/LAN DNS |
| Connection refused tới tang3 | HAProxy container/listener chưa chạy hoặc port conflict |
| Timeout tới tang3 | route/firewall/LAN |
| HAProxy backend down / 503 | tang4 port hoặc ServiceLB không sẵn sàng |
| Traefik `404 page not found` | `Host`/path không match Ingress |
| `502/503` sau khi match Ingress | Service không có Ready endpoint hoặc Pod lỗi |
| `200 Welcome to nginx!` | Toàn bộ application path hoạt động |

Status code chỉ là chỉ dấu. Luôn xác nhận bằng state/log ở đúng lớp.

## 5. Kubernetes API path

### Lớp 1 - Kubeconfig

```bash
kubectl --kubeconfig /home/monitor/.kube/config config view --minify
```

Server phải là:

```text
https://192.168.30.45:6443
```

Kubeconfig là admin credential. Không in nội dung đầy đủ vào log công khai hoặc commit Git.

### Lớp 2 - HAProxy API frontend

Trên tang3:

```bash
curl -vk https://192.168.30.45:6443/ping
docker logs --tail 100 k3s-haproxy
sudo ss -lntp | grep ':6443'
```

Kết quả `/ping` mong đợi là `pong`.

### Lớp 3 - K3s server trên tang2

Trên tang2:

```bash
sudo systemctl is-active k3s
sudo systemctl status k3s --no-pager
sudo journalctl -u k3s --since '-15 min' --no-pager
sudo ss -lntp | grep ':6443'
sudo k3s kubectl get --raw=/readyz
```

Diễn giải:

- Tang2 API local khỏe nhưng tang3 `/ping` fail: HAProxy backend, LAN route hoặc firewall.
- Tang2 service không active: đọc K3s journal trước khi chạy lại automation.
- Log chứa I/O/ext4 error: dừng; không cố repair filesystem tự động.

### Lớp 4 - Worker join

Trên tang4:

```bash
sudo systemctl is-active k3s-agent
sudo systemctl status k3s-agent --no-pager
sudo journalctl -u k3s-agent --since '-15 min' --no-pager
```

Từ tang3:

```bash
kubectl --kubeconfig /home/monitor/.kube/config get node server-tang4 -o wide
kubectl --kubeconfig /home/monitor/.kube/config describe node server-tang4
```

Nếu agent không join, kiểm tra endpoint `192.168.30.45:6443`, certificate time/TLS, token workflow, hostname và LAN interface. Không tự copy token vào Git hoặc shell history.

## 6. Ansible không kết nối được host

```bash
ansible-inventory --graph
ansible all -m ping
ansible server-tang2 -m setup -a 'filter=ansible_all_ipv4_addresses'
ansible server-tang4 -m setup -a 'filter=ansible_all_ipv4_addresses'
```

Kiểm tra theo thứ tự:

1. DHCP IP hiện tại có khớp `inventories/production/hosts.yml` không.
2. Client/controller có route tới LAN không.
3. SSH daemon có chạy không.
4. User `node` và key `/home/monitor/.ssh/k3s_ansible_ed25519` có đúng không.
5. Host key trong `known_hosts` có hợp lệ không; repo bật `host_key_checking=True`.
6. Remote user có sudo phù hợp không.

Không tắt host key checking chỉ để bỏ qua lỗi nhận dạng host; xác minh fingerprint mới nếu máy/IP thực sự thay đổi.

## 7. Public image không pull được

```bash
kubectl --kubeconfig /home/monitor/.kube/config -n public-app describe pod -l app=nginx
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get events --sort-by=.lastTimestamp
```

Trên tang4, kiểm tra DNS và outbound internet. Phân biệt:

- `not found`: image/tag sai;
- `i/o timeout`: DNS, route, proxy hoặc firewall;
- `toomanyrequests`: anonymous registry rate limit;
- TLS/x509 error: system time hoặc trust chain.

Demo dùng `imagePullPolicy: Always`, nên mỗi lần tạo Pod mới cần registry khả dụng ngay cả khi image từng được pull.

## 8. Protected Docker workloads hoặc monitoring mất

Tang2:

```bash
docker ps --filter name=code-frontend-1
docker ps --filter name=code-backend-1
```

Tang3:

```bash
docker ps --filter name=power-grafana
docker ps --filter name=power-prometheus
docker ps --filter name=k3s-haproxy
```

Preflight/validation cố ý fail nếu các protected containers không tồn tại. Repo không sở hữu lifecycle của Docker apps và monitoring; điều tra stack gốc thay vì sửa role K3s để tự khởi động chúng.

## 9. DHCP IP thay đổi

Dấu hiệu thường gặp:

- Ansible timeout hoặc preflight báo inventory IP không tồn tại trên host;
- HAProxy backend down;
- kubeconfig trỏ endpoint cũ;
- Windows hosts trỏ sai tang3.

Quy trình:

1. Discovery LAN IP thật trên cả ba máy.
2. Cập nhật `ansible_host` trong `inventories/production/hosts.yml`.
3. Chạy `ansible-inventory --graph` và `ansible all -m ping`.
4. Chạy `ansible-playbook playbooks/site.yml` để converge config derived từ inventory.
5. Nếu IP tang3 đổi, cập nhật LAN DNS/Windows hosts.
6. Chạy validation end-to-end.

## 10. Tang2 có lỗi APT hoặc filesystem

APT/dpkg không khỏe và disk tang2 có lịch sử lỗi vật lý/read/ext4. Đây là known limitation đã chấp nhận cho disposable lab.

Không thực hiện qua repo:

- purge package để làm sạch;
- unattended APT repair;
- `fsck` trên mounted filesystem;
- reset/reinstall K3s khi chưa có bằng chứng K3s hỏng;
- storage recovery có nguy cơ phá `code-frontend-1` hoặc `code-backend-1`.

Chỉ xem disk là blocker khi K3s installation/startup thực tế thất bại do I/O error hoặc filesystem corruption. Khi đó dừng thay đổi và thay SSD theo quy trình bảo vệ workload hiện hữu.

## 11. Xác nhận sau khi sửa

Sau một fix có phạm vi nhỏ, test từ trong ra ngoài:

```bash
# Kubernetes state
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
kubectl --kubeconfig /home/monitor/.kube/config get pods -A -o wide

# Demo application path
curl -H 'Host: nginx.onprem.site' http://192.168.30.45/

# Full repository validation
ansible-playbook playbooks/validate.yml
```

Cuối cùng chạy lại playbook liên quan để xác nhận idempotency. Một fix chỉ hoàn tất khi test trực tiếp thành công, validation không regression và lần reconcile tiếp theo không tiếp tục tạo thay đổi ngoài dự kiến.
