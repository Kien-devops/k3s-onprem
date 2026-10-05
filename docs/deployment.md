# Deployment và migration runbook

## Desired state

| Host | IP | Service |
| --- | --- | --- |
| tang2 | `192.168.30.44` | `k3s.service` |
| tang4 | `192.168.30.35` | `k3s-agent.service` |
| tang1 | `192.168.30.200` | `k3s-agent.service` (compute worker) |
| tang3 | `192.168.30.45` | HAProxy container, Ansible, monitoring, runner |

## Chuẩn bị

Trên tang3:

```bash
cd /home/monitor/k3s-onprem
source /home/monitor/.venvs/k3s-ansible/bin/activate
ansible-inventory --graph
ansible all -m ping
ansible-playbook --syntax-check playbooks/migrate-to-dedicated-control-plane.yml
ansible-playbook --syntax-check playbooks/site.yml
```

Backup application trước migration:

```bash
kubectl --kubeconfig /home/monitor/.kube/config \
  get all,ingress,networkpolicy,serviceaccount,role,rolebinding \
  -n microservices-demo -o yaml \
  > /home/monitor/app1-pre-migration-backup.yaml
chmod 600 /home/monitor/app1-pre-migration-backup.yaml
```

## Migration một lần

SQLite của K3s single-node cũ trên tang4 không thể biến thành một multi-server control plane. Với app stateless, quy trình sạch là dựng cluster mới:

```bash
ansible-playbook playbooks/migrate-to-dedicated-control-plane.yml \
  -e confirm_k3s_topology_migration=true
ansible-playbook playbooks/site.yml
```

Migration playbook:

1. Xác minh tang2 `.200`, interface LAN và trạng thái chưa có K3s/runtime.
2. Xác minh backup app trên tang3.
3. Xác minh tang4 đang chạy đúng K3s server cũ.
4. Gọi official `k3s-uninstall.sh` trên tang4.
5. Chỉ xóa K3s/CNI leftovers đã biết.

`site.yml` sau đó converge theo thứ tự:

```text
preflight
 -> HAProxy API backend tang2
 -> K3s server tang2
 -> K3s agents tang4 và tang1
 -> hai replica Traefik và ServiceLB trên tang4/tang1
 -> HAProxy ingress backends tang4/tang1
 -> smoke workload
 -> validation
```

Worker convergence dùng `serial: 1` và thứ tự hostname để cập nhật tang1 trước tang4. Cách này tránh restart đồng thời hai K3s agent và giữ backend ingress hiện hữu trên tang4 trong lúc tang1 được đưa vào pool.

## Restore app1

Bootstrap lại namespace-scoped deploy identity nếu cluster mới:

```bash
cd /home/monitor/actions-runner-app1/_work/app1/app1
sh scripts/bootstrap-rbac.sh
```

Release bình thường phải đi qua push `main`, tạo exact-SHA images và deploy bằng self-hosted runner. Không dùng `latest` hoặc admin kubeconfig trong CI.

## Validation

```bash
ansible-playbook playbooks/validate.yml
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
kubectl --kubeconfig /home/monitor/.kube/config get pods -A -o wide
curl -ksS https://192.168.30.44:6443/ping
curl -ksS https://192.168.30.45:6443/ping
curl -H 'Host: app1.onprem.site' http://192.168.30.45/api/auth/health
curl --fail https://app1.onprem.site/api/auth/health
```

Chạy `site.yml` lần hai. Idempotency đạt khi `changed=0`, `failed=0` (ngoại trừ thay đổi có chủ đích từ external controllers).

## Rollback boundary

Sau khi official uninstall chạy trên tang4, rollback tại chỗ về SQLite cũ không được bảo đảm. Recovery source là Git, exact-SHA images và backup manifest. Với dữ liệu stateful trong tương lai, phải có PVC/database backup riêng trước migration.
