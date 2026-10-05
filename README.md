# K3s On-Prem Homelab bằng Ansible

Repository này tự động hóa một cluster K3s single-node trên hai máy Ubuntu trong cùng LAN. Đây là homelab phục vụ học tập và chạy workload nhỏ, không phải kiến trúc Kubernetes High Availability.

## Topology hiện tại

```text
LAN client / kubectl / Internet
              |
              | API :6443, application :80/:443
              v
  +-----------------------------------------+
  | server-tang3 - 192.168.30.45           |
  |                                         |
  | Ansible controller                      |
  | HAProxy container                       |
  | Cloudflare Tunnel                       |
  | Prometheus / Grafana hiện hữu           |
  +-------------------+---------------------+
                      |
                      | TCP passthrough
                      v
  +-----------------------------------------+
  | server-tang4 - 192.168.30.35           |
  |                                         |
  | K3s server / control plane              |
  | SQLite datastore                        |
  | Schedulable workload node               |
  | Traefik + ServiceLB                     |
  | Application Pods                        |
  +-----------------------------------------+
```

`server-tang2` không còn là thành viên cluster. Cluster cũ dùng SQLite trên tang2 đã bị coi là mất hoàn toàn; tang4 được làm sạch agent state cũ và dựng thành K3s server mới.

| Inventory group | Host | LAN IP | Trách nhiệm |
| --- | --- | --- | --- |
| `control_plane` | `server-tang4` | `192.168.30.35` | K3s API, SQLite, Traefik, ServiceLB và workloads |
| `load_balancers` | `server-tang3` | `192.168.30.45` | Ansible, HAProxy và monitoring |
| `workers` | rỗng | - | Dành cho mở rộng tương lai, không chạy role agent |

Tang4 không có control-plane `NoSchedule` taint. Nó mang hai label ServiceLB:

```text
svccontroller.k3s.cattle.io/enablelb=true
svccontroller.k3s.cattle.io/lbpool=ingress
```

## Data paths

Kubernetes API:

```text
kubectl
  -> 192.168.30.45:6443
  -> HAProxy tang3
  -> 192.168.30.35:6443
  -> K3s API tang4
  -> SQLite tang4
```

Application traffic:

```text
Client
  -> 192.168.30.45:80/:443
  -> HAProxy tang3
  -> 192.168.30.35:80/:443
  -> ServiceLB
  -> Traefik
  -> Ingress
  -> ClusterIP Service
  -> Pod tang4
```

HAProxy chạy TCP passthrough. Nó không đọc Ingress và không terminate Kubernetes API TLS. K3s certificate có TLS SAN `192.168.30.45` để kubeconfig dùng stable endpoint qua HAProxy.

## Failure boundaries

Đây không phải HA Kubernetes:

- Nếu tang4 mất, control plane, SQLite datastore, Traefik và toàn bộ application workload đều mất.
- Nếu tang3 mất, K3s vẫn chạy trên tang4 nhưng stable API endpoint, public application entry point, Cloudflare Tunnel và monitoring trên tang3 mất.
- Nếu DHCP đổi LAN IP, inventory phải được cập nhật rồi chạy lại Ansible.

## Công nghệ được pin

| Thành phần | Phiên bản / chế độ |
| --- | --- |
| K3s | `v1.36.4+k3s1` |
| k3s-ansible | `k3s.orchestration` `1.2.2` |
| ansible-core | `2.21.4` |
| HAProxy | `haproxy:3.2.24-alpine3.24` |
| CNI | Flannel VXLAN |
| Datastore | SQLite trên tang4 |
| Ingress | Traefik bundled với K3s |
| Service exposure | K3s ServiceLB trên tang4 |
| Pod CIDR | `10.42.0.0/16` |
| Service CIDR | `10.43.0.0/16` |

## Cấu trúc repository

```text
.
|-- ansible.cfg
|-- requirements.txt
|-- collections/requirements.yml
|-- inventories/production/
|   |-- hosts.yml
|   |-- group_vars/
|   |   |-- all.yml
|   |   |-- control_plane.yml
|   |   `-- load_balancers.yml
|   `-- host_vars/
|       |-- server-tang3.yml
|       `-- server-tang4.yml
|-- playbooks/
|   |-- rebuild-single-node.yml
|   |-- preflight.yml
|   |-- cluster.yml
|   |-- addons.yml
|   |-- site.yml
|   |-- smoke-test.yml
|   |-- validate.yml
|   `-- app.yml
|-- roles/
|   |-- preflight/
|   |-- k3s_control_plane/
|   |-- haproxy/
|   |-- k3s_addons/
|   `-- validation/
|-- manifests/
|   |-- traefik/
|   |-- smoke-test/
|   `-- app/
`-- docs/
    |-- architecture.md
    |-- deployment.md
    |-- troubleshooting.md
    |-- application-cicd.md
    `-- app1-microservices.md
```

## Inventory và variables

`inventories/production/hosts.yml` là source of truth cho host membership và LAN IP. Tailscale chỉ dùng cho management/SSH, không dùng làm `node-ip`, Flannel interface hoặc HAProxy backend.

Các giá trị derived từ inventory:

- `api_endpoint`: IP tang3 từ nhóm `load_balancers`;
- `k3s_control_plane_name`: host đầu tiên trong `control_plane`;
- `k3s_workload_node_name`: cùng host control plane trong topology single-node;
- `haproxy_api_backend`: LAN IP của `control_plane`;
- `haproxy_ingress_backend`: LAN IP của workload node.

Không hard-code backend IP trong HAProxy template.

## Vai trò các role

### `preflight`

- Xác minh inventory LAN IP và interface thật sự tồn tại.
- Kiểm tra RAM tối thiểu, swap và kernel modules trên K3s node.
- Fail-closed nếu `power-grafana` hoặc `power-prometheus` trên tang3 không chạy.
- Không sửa APT, filesystem, Tailscale hoặc Docker workload ngoài phạm vi.

### `k3s_control_plane`

- Kiểm tra binary, version, config và `k3s.service` trước khi converge.
- Chỉ gọi upstream server role khi có drift.
- Fetch kubeconfig từ tang4 và đổi endpoint thành `https://192.168.30.45:6443`.
- Không cài agent role và không dùng token của cluster cũ.

### `haproxy`

- Render config từ inventory.
- Validate bằng `haproxy -c` trước khi thay container.
- Chỉ recreate container khi config đổi hoặc container không chạy.
- Giữ monitoring containers và Docker resources không liên quan.

### `k3s_addons`

- Cấu hình Traefik Service dùng ServiceLB pool `ingress`.
- Xác minh Traefik và `svclb-traefik` chỉ chạy trên tang4.

### `validation`

- Xác minh cluster chỉ có một node `server-tang4`, Ready và schedulable.
- Xác minh ServiceLB labels, Traefik, ServiceLB và smoke Pod placement.
- Test K3s API trực tiếp và qua HAProxy.
- Test HTTP/HTTPS ingress qua HAProxy.
- Xác minh HAProxy, Prometheus và Grafana vẫn chạy trên tang3.

## Trình tự rebuild

Chạy trên `server-tang3`:

```bash
cd /home/monitor/k3s-onprem
source /home/monitor/.venvs/k3s-ansible/bin/activate

ansible-inventory --graph
ansible all -m ping
ansible-playbook playbooks/site.yml --syntax-check
```

One-time cleanup của agent thuộc cluster đã mất:

```bash
ansible-playbook playbooks/rebuild-single-node.yml \
  -e confirm_k3s_rebuild=true
```

Playbook recovery được guard rõ ràng, gọi official `k3s-agent-uninstall.sh`, chỉ xóa các K3s leftovers đã biết và từ chối chạy nếu phát hiện Docker, CRI-O hoặc standalone containerd đang active trên tang4.

Sau cleanup:

```bash
ansible-playbook playbooks/site.yml
ansible-playbook playbooks/validate.yml
```

Luồng `site.yml`:

```text
Preflight tang3 + tang4
  -> HAProxy API tang3:6443 -> tang4:6443
  -> K3s server tang4
  -> Traefik / ServiceLB
  -> HAProxy application :80/:443 -> tang4
  -> smoke workload
  -> full validation
```

Chi tiết tại [docs/deployment.md](docs/deployment.md).

## Smoke và demo workloads

Smoke workload trong namespace `lab-demo` là proof của platform data path và được deploy trong `site.yml`.

```bash
curl -H 'Host: demo.apps.k3s.home.arpa' http://192.168.30.45/
```

Demo Nginx trong namespace `public-app` có lifecycle riêng:

```bash
ansible-playbook playbooks/app.yml
curl -H 'Host: nginx.onprem.site' http://192.168.30.45/
```

Cả hai manifest vẫn dùng `nodeSelector: server-tang4`, phù hợp vì tang4 là schedulable control-plane/workload node.

## Ứng dụng `app1`

Ứng dụng microservices nằm ở repository [`Kien-devops/app1`](https://github.com/Kien-devops/app1). Repository này sở hữu source, image, namespace-scoped manifests và CI/CD; `k3s-onprem` chỉ cung cấp platform.

```text
GitHub Actions
  -> GHCR exact-SHA images
  -> self-hosted runner tang3
  -> HAProxy API endpoint
  -> K3s API tang4
  -> Pods tang4
```

Sau khi rebuild cluster, RBAC/kubeconfig của app phải được bootstrap lại trước khi redeploy. Xem [docs/app1-microservices.md](docs/app1-microservices.md).

## Validation nhanh

```bash
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
kubectl --kubeconfig /home/monitor/.kube/config get pods -A -o wide
kubectl --kubeconfig /home/monitor/.kube/config get svc -A
kubectl --kubeconfig /home/monitor/.kube/config get ingress -A

curl -k https://192.168.30.35:6443/ping
curl -k https://192.168.30.45:6443/ping
curl -H 'Host: demo.apps.k3s.home.arpa' http://192.168.30.45/
curl -k -H 'Host: demo.apps.k3s.home.arpa' https://192.168.30.45/
```

Kết quả tối thiểu để coi recovery thành công:

- chỉ có `server-tang4`, trạng thái `Ready`;
- direct API và HAProxy API đều trả `pong`;
- CoreDNS, metrics-server, Traefik và ServiceLB Running;
- smoke request HTTP/HTTPS đi qua HAProxy đến đúng Pod;
- lần chạy `site.yml` thứ hai không tạo drift ngoài dự kiến.

## Nguyên tắc an toàn

- Không dùng hoặc reconnect SQLite/token của cluster cũ.
- Không xóa `/home`, SSH, Tailscale, network config hoặc Docker resources tang3.
- Không flush firewall mù quáng.
- Không dùng Tailscale IP trong K3s data path.
- Không mô tả topology này là production-ready hoặc HA.
- `server.txt`, kubeconfig, token và private keys không được commit.

Đọc thêm:

- [Kiến trúc và failure boundaries](docs/architecture.md)
- [Quy trình deployment/rebuild](docs/deployment.md)
- [Troubleshooting theo data path](docs/troubleshooting.md)
- [Ứng dụng app1 trên cluster](docs/app1-microservices.md)
