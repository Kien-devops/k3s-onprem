# Troubleshooting theo data path

Không reset/reinstall trước khi tìm hop đầu tiên bị lỗi.

## Snapshot ban đầu

```bash
cd /home/monitor/k3s-onprem
source /home/monitor/.venvs/k3s-ansible/bin/activate
ansible all -m ping
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
kubectl --kubeconfig /home/monitor/.kube/config get pods -A -o wide
docker ps --filter name=k3s-haproxy --filter name=power-prometheus --filter name=power-grafana
```

## Kubernetes API

```text
kubectl -> tang3:6443 -> HAProxy -> tang2:6443 -> k3s.service
```

```bash
# tang3
curl -ksS https://192.168.30.45:6443/ping
curl -ksS https://192.168.30.200:6443/ping
docker logs --tail 100 k3s-haproxy
grep -n '192.168.30.200:6443' /home/monitor/k3s-haproxy/haproxy.cfg

# tang2
sudo systemctl status k3s --no-pager
sudo journalctl -u k3s --since '-15 min' --no-pager
sudo k3s kubectl get --raw=/readyz
```

Direct tang2 `pong` nhưng tang3 fail nghĩa là lỗi HAProxy/listener/route. Cả hai fail nghĩa là kiểm tra `k3s.service`, LAN và firewall tang2.

## Worker join

```bash
# tang4
sudo systemctl status k3s-agent --no-pager
sudo journalctl -u k3s-agent --since '-15 min' --no-pager

# tang3
kubectl --kubeconfig /home/monitor/.kube/config get node server-tang4 -o wide
kubectl --kubeconfig /home/monitor/.kube/config describe node server-tang4
```

Agent join qua stable endpoint `192.168.30.45:6443`. Kiểm tra HAProxy, token runtime, certificate time và hostname; không copy token vào Git/shell history.

## Application ingress

```text
Client -> tang3:80/443 -> tang4:80/443 -> ServiceLB
       -> Traefik -> Ingress -> ClusterIP -> Pod
```

```bash
curl -v -H 'Host: app1.onprem.site' http://192.168.30.45/
nc -vz 192.168.30.35 80
nc -vz 192.168.30.35 443
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system get svc,pods -o wide
kubectl --kubeconfig /home/monitor/.kube/config get ingress -A
kubectl --kubeconfig /home/monitor/.kube/config -n microservices-demo get pods,svc,endpointslice -o wide
```

| Kết quả | Lớp cần kiểm tra |
| --- | --- |
| timeout/refused tang3 | HAProxy listener, route, firewall |
| HAProxy backend down | tang4 agent, ServiceLB hoặc port conflict |
| Traefik `404` | Host/path/Ingress rule |
| `502/503` | Service selector, EndpointSlice, readiness |
| `ImagePullBackOff` | GHCR visibility, image SHA, DNS/outbound |
| Pod `Pending` | nodeSelector, control-plane taint, worker capacity |

## Placement sai

```bash
kubectl --kubeconfig /home/monitor/.kube/config get nodes --show-labels
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system get pods -o wide
```

Mong đợi:

- tang2 có control-plane `NoSchedule` taint;
- tang4 có `enablelb=true`, `lbpool=ingress`;
- Traefik, ServiceLB, smoke và app Pods nằm trên tang4.

Chạy `ansible-playbook playbooks/site.yml` để reconcile; không xóa taint nhằm che lỗi worker.

## Migration guard fail

`migrate-to-dedicated-control-plane.yml` chỉ dùng một lần. Nó dừng nếu identity/IP/interface sai, backup app không tồn tại, tang2 có runtime/K3s hoặc tang4 không còn là K3s server cũ. Đọc assertion và xác minh state; không sửa guard để ép chạy.

## Protected services trên tang3

```bash
docker ps --filter name=k3s-haproxy
docker ps --filter name=power-prometheus
docker ps --filter name=power-grafana
systemctl status cloudflared --no-pager
systemctl status actions.runner.Kien-devops-app1.server-tang3-k3s-deploy --no-pager
```

Không chạy `docker system prune`, không xóa volume/network và không recreate monitoring trong migration K3s.

## Kết thúc sự cố

```bash
ansible-playbook playbooks/validate.yml
ansible-playbook playbooks/site.yml
```

Chỉ đóng sự cố khi hai node Ready, direct/HAProxy API trả `pong`, HTTP/HTTPS ingress pass, app public healthy và lần converge tiếp theo idempotent.
