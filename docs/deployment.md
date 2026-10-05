# Rebuild và triển khai K3s homelab

Tài liệu này mô tả quy trình rebuild cluster mới từ `server-tang3`. Cluster cũ có SQLite trên control plane đã mất và không được recover/rejoin.

## 1. Source of truth

| Host | LAN IP | Connection | Vai trò mới |
| --- | --- | --- | --- |
| `server-tang3` | `192.168.30.45` | local user `monitor` | Ansible, HAProxy, tunnel, monitoring |
| `server-tang4` | `192.168.30.35` | SSH user `node` | K3s server, SQLite, schedulable workload node |

LAN interfaces:

```text
server-tang3 -> wlp2s0
server-tang4 -> wlx58044f3fedb4
```

Tailscale chỉ dùng SSH/management. Không đặt Tailscale IP vào K3s config hoặc HAProxy backend.

## 2. Bảo vệ workload không liên quan

Trước recovery, ghi lại trên tang3 và tang4:

```bash
docker ps -a || true
docker volume ls || true
docker network ls || true
systemctl --type=service --state=running
```

Yêu cầu bảo vệ:

- `power-prometheus` và `power-grafana` trên tang3 phải tiếp tục chạy;
- không xóa Docker containers/volumes/networks ngoài K3s;
- không xóa Tailscale, SSH, `/home` hoặc network configuration;
- không format disk, chạy `fsck` hoặc reinstall OS;
- không flush firewall mù quáng.

Recovery playbook từ chối cleanup nếu Docker, CRI-O hoặc standalone containerd active trên tang4.

## 3. Chuẩn bị controller

Chạy trên tang3:

```bash
cd /home/monitor/k3s-onprem
python3 -m venv /home/monitor/.venvs/k3s-ansible
source /home/monitor/.venvs/k3s-ansible/bin/activate
python -m pip install -r requirements.txt
ansible-galaxy collection install -r collections/requirements.yml
```

Repo pin Ansible/K3s versions; recovery không tự upgrade ngoài phạm vi.

## 4. Discovery và preflight

```bash
ansible-inventory --graph
ansible-inventory --host server-tang3
ansible-inventory --host server-tang4
ansible all -m ping
ansible-playbook playbooks/preflight.yml
```

Inventory graph mong đợi:

```text
control_plane -> server-tang4
load_balancers -> server-tang3
server -> control_plane
workers/agent -> empty
k3s_cluster -> server-tang4
```

Preflight fail nếu inventory LAN IP/interface sai, RAM không đủ hoặc monitoring tang3 không chạy.

## 5. Syntax check

```bash
ansible-playbook playbooks/rebuild-single-node.yml --syntax-check
ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/app.yml --syntax-check
```

Không chạy cleanup nếu syntax check hoặc Ansible connectivity fail.

## 6. One-time cleanup agent cũ

Xác nhận cluster cũ và SQLite cũ thực sự đã mất, sau đó chạy:

```bash
ansible-playbook playbooks/rebuild-single-node.yml \
  -e confirm_k3s_rebuild=true
```

Playbook:

1. Xác minh target đúng `server-tang4`/`192.168.30.35`.
2. Refuse nếu `k3s.service` đã tồn tại.
3. Refuse nếu có unrelated container runtime active.
4. Yêu cầu official `/usr/local/bin/k3s-agent-uninstall.sh`.
5. Chạy uninstall script.
6. Chỉ xóa known K3s leftovers còn lại.
7. Chỉ xóa `cni0`, `flannel.1`, `kube-ipvs0` nếu chúng tồn tại.
8. Reload systemd và xác minh `k3s agent` process đã mất.

Sau cleanup:

```bash
ansible server-tang4 -b -m shell -a \
  "systemctl status k3s-agent --no-pager || true; ip link show"
```

## 7. Dựng cluster mới

```bash
ansible-playbook playbooks/site.yml
```

### Phase 1: Preflight

Xác minh tang3/tang4 và áp dụng baseline tối thiểu cho K3s.

### Phase 2: HAProxy API

Render/validate:

```text
192.168.30.45:6443 -> 192.168.30.35:6443
```

Backend có thể tạm thời Down trước khi K3s server được cài; frontend listener vẫn phải tồn tại.

### Phase 3: K3s server

Upstream role cài K3s server `v1.36.4+k3s1` trên tang4 với:

- LAN node IP/interface;
- TLS SAN tang3;
- SQLite;
- secrets encryption;
- ServiceLB labels;
- không có `NoSchedule` taint.

Kubeconfig gốc được fetch từ `/etc/rancher/k3s/k3s.yaml`, rồi endpoint trên tang3 được đổi thành:

```text
https://192.168.30.45:6443
```

### Phase 4: Addons

Ansible apply Traefik `HelmChartConfig`, đợi rollout và xác minh Traefik/ServiceLB trên tang4.

### Phase 5: HAProxy application listeners

```text
192.168.30.45:80  -> 192.168.30.35:80
192.168.30.45:443 -> 192.168.30.35:443
```

Template được kiểm tra bằng `haproxy -c` trước khi recreate container.

### Phase 6: Smoke workload

Static manifest tạo namespace `lab-demo`, Deployment, Service và Ingress `demo.apps.k3s.home.arpa`.

### Phase 7: Validation

Kiểm tra single node, scheduling, API direct/proxy, Traefik/ServiceLB, smoke HTTP/HTTPS và protected containers tang3.

## 8. Validation thủ công

```bash
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
kubectl --kubeconfig /home/monitor/.kube/config get nodes --show-labels
kubectl --kubeconfig /home/monitor/.kube/config get pods -A -o wide
kubectl --kubeconfig /home/monitor/.kube/config get svc -A
kubectl --kubeconfig /home/monitor/.kube/config get ingress -A
kubectl --kubeconfig /home/monitor/.kube/config get endpoints -A
```

Expected node set:

```text
server-tang4   Ready   control-plane
```

Không được còn stale cluster node khác.

API proof:

```bash
curl -k https://192.168.30.35:6443/ping
curl -k https://192.168.30.45:6443/ping
```

Ingress proof:

```bash
curl -H 'Host: demo.apps.k3s.home.arpa' http://192.168.30.45/
curl -k -H 'Host: demo.apps.k3s.home.arpa' https://192.168.30.45/
```

## 9. Demo application

Sau khi platform validation pass:

```bash
ansible-playbook playbooks/app.yml
curl -H 'Host: nginx.onprem.site' http://192.168.30.45/
```

Hai Nginx replicas phải Ready trên tang4. `nodeSelector: server-tang4` vẫn đúng dù node đồng thời là control plane.

## 10. Restore app1 CI/CD

Cluster rebuild làm mất namespace/RBAC/token cũ. Trước khi pipeline app1 deploy lại:

1. Bootstrap namespace-scoped `ci-deployer` bằng admin kubeconfig.
2. Tạo lại `/home/monitor/.kube/microservices-demo-deployer.config` mode `0600`.
3. Deploy exact known-good Git SHA hoặc chạy lại pipeline `app1`.
4. Xác minh `/`, Auth, User và Product health routes.

Không dùng admin kubeconfig trong application runner.

## 11. Idempotency

Chạy lại:

```bash
ansible-playbook playbooks/site.yml
ansible-playbook playbooks/validate.yml
```

HAProxy không được recreate nếu config/container không drift. K3s không được reinstall nếu binary, version, config và service đã đúng. Read-only command tasks phải báo `ok`.

Recovery playbook là one-time destructive workflow; sau khi tang4 đã chạy `k3s.service`, guard sẽ từ chối chạy lại.

## 12. Điều kiện dừng

Dừng thay vì bypass khi:

- host identity/IP/interface không đúng inventory;
- phát hiện unrelated runtime/workload trên tang4;
- official agent uninstall script không tồn tại;
- K3s fail do disk/filesystem I/O;
- monitoring tang3 mất;
- bước tiếp theo cần xóa dữ liệu ngoài danh sách K3s đã review.
