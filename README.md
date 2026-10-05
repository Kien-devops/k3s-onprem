# K3s On-Prem Homelab bằng Ansible

Repository tự động hóa cluster K3s hai node với control plane và workload tách biệt. Tang3 cung cấp automation và stable edge endpoint nhưng không phải thành viên K3s.

## Topology

```text
Client / kubectl / Internet
          |
          | API :6443, application :80/:443
          v
server-tang3 - 192.168.30.45
  Ansible + HAProxy + cloudflared + monitoring + GitHub runner
          |
          +-- :6443 --------> server-tang2 - 192.168.30.200
          |                     K3s server/control plane + SQLite
          |
          `-- :80/:443 -----> server-tang4 - 192.168.30.35
                                K3s agent/worker + ServiceLB + Traefik + apps
```

| Inventory group | Host | LAN IP | Trách nhiệm |
| --- | --- | --- | --- |
| `control_plane` | `server-tang2` | `192.168.30.200` | K3s API, scheduler/controller và SQLite datastore |
| `workers` | `server-tang4` | `192.168.30.35` | Traefik, ServiceLB và application Pods |
| `load_balancers` | `server-tang3` | `192.168.30.45` | Ansible, HAProxy, tunnel, monitoring và CD runner |

Control plane có `NoSchedule` taint. Tang4 có ServiceLB labels `enablelb=true` và `lbpool=ingress`, nên platform/application workloads chỉ chạy trên worker.

Đây không phải Kubernetes HA. Nếu tang2 mất, API/SQLite mất và cluster không thể reconcile; các Pod đang chạy trên tang4 có thể tiếp tục tạm thời nhưng không thể coi là hệ thống khỏe. Nếu tang4 mất, application/Ingress mất. Nếu tang3 mất, cluster vẫn chạy nội bộ nhưng stable API, public edge, monitoring và CD runner mất.

## Data paths

```text
kubectl -> tang3:6443 -> HAProxy -> tang2:6443 -> K3s API -> SQLite

Client -> tang3:80/:443 -> HAProxy -> tang4:80/:443
       -> ServiceLB -> Traefik -> Ingress -> ClusterIP Service -> Pod
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
  group_vars/{all,control_plane,workers,load_balancers}.yml
  host_vars/{server-tang2,server-tang3,server-tang4}.yml
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
- `k3s_worker`: converge K3s agent tang4, chờ node Ready và xác minh ServiceLB labels.
- `k3s_addons`: chờ Helm controller, xác minh Traefik và ServiceLB chỉ chạy trên tang4.
- `haproxy`: render từ inventory, validate config trước khi recreate container.
- `validation`: kiểm tra đúng hai node, taint/labels/placement, API direct/HAProxy, ingress HTTP/HTTPS và protected containers.

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

Migration một lần từ cluster single-node cũ trên tang4 yêu cầu backup app và confirmation flag:

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

curl -ksS https://192.168.30.200:6443/ping
curl -ksS https://192.168.30.45:6443/ping
curl -H 'Host: demo.apps.k3s.home.arpa' http://192.168.30.45/
```

Kết quả đúng: tang2 và tang4 đều Ready; control plane có `NoSchedule`; Traefik, ServiceLB, smoke workload và `app1` nằm trên tang4; direct API và HAProxy API đều trả `pong`.

## `app1` và CI/CD

Ứng dụng nằm tại [`Kien-devops/app1`](https://github.com/Kien-devops/app1). Pipeline test bốn component, build/scan/push exact-SHA images lên GHCR, sau đó self-hosted runner tang3 deploy bằng namespace-scoped kubeconfig.

```text
GitHub Actions -> GHCR:<git-sha> -> runner tang3
  -> HAProxy API tang3:6443 -> K3s API tang2
  -> Deployments/ClusterIP/Ingress -> Pods tang4
```

Chi tiết: [docs/app1-microservices.md](docs/app1-microservices.md) và [docs/application-cicd.md](docs/application-cicd.md).

## Security và giới hạn

- Không commit `server.txt`, password, token, kubeconfig hoặc private key.
- Tailscale chỉ dùng management; K3s node IP/Flannel/HAProxy dùng LAN.
- Không expose `:6443` ra Internet.
- App pipeline không dùng admin kubeconfig và không quản lý cluster-level resources.
- SQLite một control plane không cung cấp HA; cần backup datastore và rebuild procedure.
- Nên đặt DHCP reservation cho `.200`, `.45` và `.35`.

Xem [deployment](docs/deployment.md), [architecture](docs/architecture.md) và [troubleshooting](docs/troubleshooting.md) để vận hành chi tiết.
