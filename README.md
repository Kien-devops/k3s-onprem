# K3s On-Prem Homelab bằng Ansible

Repository tự động hóa cluster K3s ba node với control plane và worker tách biệt. Tang3 cung cấp automation và stable edge endpoint nhưng không phải thành viên K3s.

## Topology

```text
Client / kubectl / Internet
          |
          | API :6443, application :80/:443
          v
server-tang3 - 192.168.30.45
  Ansible + HAProxy + cloudflared + monitoring + GitHub runner
          |
          +-- :6443 --------> server-tang2 - 192.168.30.44
          |                     K3s server/control plane + SQLite
          |
          `-- :80/:443 --+--> server-tang4 - 192.168.30.35
                         |      K3s worker + ServiceLB + Traefik + app1 replica
                         |
                         `--> server-tang1 - 192.168.30.200
                                K3s worker + ServiceLB + Traefik + app1 replica
```

| Inventory group | Host | LAN IP | Trách nhiệm |
| --- | --- | --- | --- |
| `control_plane` | `server-tang2` | `192.168.30.44` | K3s API, scheduler/controller và SQLite datastore |
| `workers`, `ingress_workers` | `server-tang4` | `192.168.30.35` | Traefik, ServiceLB, bundled workloads và một replica của mỗi workload `app1` |
| `workers`, `ingress_workers` | `server-tang1` | `192.168.30.200` | Traefik, ServiceLB và một replica của mỗi workload `app1` |
| `load_balancers` | `server-tang3` | `192.168.30.45` | Ansible, HAProxy, tunnel, monitoring và CD runner |

Control plane có `NoSchedule` taint. Cả tang1 và tang4 thuộc nhóm `ingress_workers`, có ServiceLB labels `enablelb=true`, `lbpool=ingress` và chạy một replica Traefik. Smoke test và manifest `public-app` trong repository này vẫn được pin vào tang4. Riêng `app1` do repository ứng dụng quản lý dùng hai replica cho mỗi Deployment, bắt buộc một replica trên mỗi worker bằng topology spread.

Đây chưa phải Kubernetes HA toàn phần. Nếu tang2 mất, API/SQLite mất và cluster không thể reconcile; các Pod đang chạy có thể tiếp tục tạm thời nhưng không thể coi là hệ thống khỏe. Nếu một worker mất, HAProxy loại backend lỗi và `app1` tiếp tục qua worker còn lại, nhưng replica thứ hai sẽ `Pending` cho đến khi failure domain phục hồi. Nếu tang3 mất, cluster vẫn chạy nội bộ nhưng stable API, public edge, monitoring và CD runner mất.

## Data paths

```text
kubectl -> tang3:6443 -> HAProxy -> tang2:6443 -> K3s API -> SQLite

Client -> tang3:80/:443 -> HAProxy -> tang1:80/:443 hoặc tang4:80/:443
       -> ServiceLB -> Traefik -> Ingress -> ClusterIP Service
       -> Pod (`public-app` trên tang4; `app1` trải đều tang1 và tang4)
```

HAProxy dùng TCP passthrough. K3s API certificate có TLS SAN `192.168.30.45`; backend IP được derive từ inventory, không hard-code trong template.

## Thành phần được pin

| Thành phần | Giá trị |
| --- | --- |
| K3s | `v1.36.4+k3s1` |
| k3s-ansible | `k3s.orchestration` `1.2.2` |
| ansible-core | `2.21.4` |
| HAProxy | `haproxy:3.2.24-alpine3.24` |
| Datastore | SQLite trên tang2 |
| CNI | Flannel VXLAN |
| Ingress | Traefik bundled với K3s |
| Pod/Service CIDR | `10.42.0.0/16` / `10.43.0.0/16` |

## Cấu trúc repository

```text
inventories/production/
  hosts.yml
  group_vars/{all,control_plane,workers,ingress_workers,load_balancers}.yml
  host_vars/{server-tang1,server-tang2,server-tang3,server-tang4}.yml
playbooks/
  migrate-to-dedicated-control-plane.yml
  preflight.yml
  cluster.yml
  addons.yml
  site.yml
  smoke-test.yml
  validate.yml
  app.yml
roles/
  preflight/
  k3s_control_plane/
  k3s_worker/
  k3s_addons/
  haproxy/
  validation/
manifests/{traefik,smoke-test,app}/
docs/
```

## Vai trò automation

- `preflight`: xác minh IP/interface/RAM, bảo vệ Prometheus và Grafana trên tang3, chuẩn bị kernel/swap cho K3s nodes.
- `k3s_control_plane`: converge K3s server tang2, lấy node token bằng `no_log`, tạo kubeconfig dùng HAProxy endpoint.
- `k3s_worker`: converge K3s agents tang4/tang1, chờ từng node Ready và xác minh ingress/ServiceLB labels.
- `k3s_addons`: chạy hai replica Traefik, bắt buộc trải đều và xác minh ServiceLB trên cả hai ingress worker.
- `haproxy`: render từ inventory, validate config trước khi recreate container.
- `validation`: kiểm tra đúng ba node, taint/labels/placement, API direct/HAProxy, ingress HTTP/HTTPS và protected containers.

## Setup và deploy

Trên tang3:

```bash
cd /home/monitor/k3s-onprem
source /home/monitor/.venvs/k3s-ansible/bin/activate

ansible-galaxy collection install -r collections/requirements.yml
pip install -r requirements.txt

ansible-inventory --graph
ansible all -m ping
ansible-playbook --syntax-check playbooks/site.yml
ansible-playbook playbooks/site.yml
```

Playbook migration được giữ lại để audit quá trình chuyển từ cluster single-node cũ trên tang4. Không chạy lại trên topology hiện tại: playbook chứa các precondition của trạng thái trước migration và sẽ fail-closed khi tang2 đã chạy K3s.

Lệnh lịch sử đã dùng cho lần migration đó:

```bash
ansible-playbook playbooks/migrate-to-dedicated-control-plane.yml \
  -e confirm_k3s_topology_migration=true
```

Playbook migration fail-closed: xác minh đúng tang2/tang4, tang2 chưa có runtime/K3s, backup app tồn tại và tang4 đang ở trạng thái K3s server mong đợi trước khi gọi official `k3s-uninstall.sh`.

## Kiểm tra

```bash
ansible-playbook playbooks/validate.yml
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
kubectl --kubeconfig /home/monitor/.kube/config get pods -A -o wide

curl -ksS https://192.168.30.44:6443/ping
curl -ksS https://192.168.30.45:6443/ping
curl -H 'Host: demo.apps.k3s.home.arpa' http://192.168.30.45/
```

Kết quả đúng của hạ tầng: tang1, tang2 và tang4 đều Ready; control plane có `NoSchedule`; mỗi worker có một Traefik Pod và một ServiceLB Pod; smoke workload vẫn nằm trên tang4; direct API và HAProxy API đều trả `pong`. Placement của `app1` được kiểm tra riêng trong pipeline ứng dụng và bằng lệnh `kubectl` bên dưới.

## `app1` và CI/CD

Ứng dụng nằm tại [`Kien-devops/app1`](https://github.com/Kien-devops/app1). Pipeline test bốn component, build/scan/push exact-SHA images lên GHCR, sau đó self-hosted runner tang3 deploy bằng namespace-scoped kubeconfig.

```text
GitHub Actions -> GHCR:<git-sha> -> runner tang3
  -> HAProxy API tang3:6443 -> K3s API tang2
  -> Deployments/ClusterIP/Ingress -> Pods tang1 + tang4
```

Mỗi Deployment (`frontend`, `auth-service`, `user-service`, `product-service`) chạy hai replica. Trạng thái đạt yêu cầu là mỗi Deployment có một Pod trên tang1 và một Pod trên tang4; pipeline phải fail nếu cả hai replica cùng nằm trên một node.

```bash
kubectl --kubeconfig /home/monitor/.kube/config \
  -n microservices-demo get pods -o wide
```

Ingress chịu được lỗi một worker vì HAProxy health-check cả tang1 và tang4. Tang3 vẫn là single edge/HAProxy failure domain và tang2 vẫn là single control-plane/datastore failure domain.

Chi tiết: [docs/app1-microservices.md](docs/app1-microservices.md) và [docs/application-cicd.md](docs/application-cicd.md).

## Security và giới hạn

- Không commit `server.txt`, password, token, kubeconfig hoặc private key.
- Tailscale chỉ dùng management; K3s node IP/Flannel/HAProxy dùng LAN.
- Không expose `:6443` ra Internet.
- App pipeline không dùng admin kubeconfig và không quản lý cluster-level resources.
- SQLite một control plane không cung cấp HA; cần backup datastore và rebuild procedure.
- Nên đặt DHCP reservation cho `.44`, `.200`, `.45` và `.35`.

Xem [deployment](docs/deployment.md), [architecture](docs/architecture.md) và [troubleshooting](docs/troubleshooting.md) để vận hành chi tiết.
