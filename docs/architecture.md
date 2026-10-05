# Kiến trúc K3s single-node homelab

## 1. Mục tiêu thiết kế

Cluster được rebuild sau khi control plane/SQLite cũ bị mất. Thiết kế mới ưu tiên đơn giản, reproducible và bảo vệ workload không liên quan:

- tang3 chỉ làm Ansible controller, HAProxy, Cloudflare Tunnel và monitoring;
- tang4 chạy K3s server, SQLite, Traefik, ServiceLB và application Pods;
- không còn K3s agent/worker riêng;
- LAN là data path; Tailscale chỉ dùng management/SSH;
- inventory và Ansible là desired state.

## 2. Physical và logical topology

```text
                           Clients
                              |
             +----------------+----------------+
             |                                 |
        API TCP 6443                     HTTP/S 80/443
             |                                 |
             v                                 v
  +-----------------------------------------------------+
  | server-tang3 - 192.168.30.45                       |
  | Ansible + HAProxy + cloudflared + monitoring       |
  +--------------------------+--------------------------+
                             |
                             | TCP passthrough
                             v
  +-----------------------------------------------------+
  | server-tang4 - 192.168.30.35                       |
  | K3s server + SQLite + schedulable workload node    |
  | Flannel + CoreDNS + metrics-server                 |
  | Traefik + ServiceLB + application Pods             |
  +-----------------------------------------------------+
```

Inventory model:

```text
control_plane  -> server-tang4 -> 192.168.30.35
load_balancers -> server-tang3 -> 192.168.30.45
workers        -> empty
server         -> alias of control_plane
agent          -> alias of empty workers group
k3s_cluster    -> control_plane only
```

Tang4 chỉ nằm trong `server`, không nằm trong `agent`. Điều này ngăn Ansible cố cài đồng thời `k3s.service` và `k3s-agent.service`.

## 3. Kubernetes server configuration

K3s server trên tang4 dùng:

```yaml
node-name: server-tang4
node-ip: 192.168.30.35
flannel-iface: wlx58044f3fedb4
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
tls-san:
  - 192.168.30.45
secrets-encryption: true
write-kubeconfig-mode: "0600"
node-label:
  - svccontroller.k3s.cattle.io/enablelb=true
  - svccontroller.k3s.cattle.io/lbpool=ingress
```

Không có `NoSchedule` taint. Control plane phải schedulable vì đây là node duy nhất.

SQLite nằm cục bộ trên tang4. Không restore hoặc reuse datastore/token của cluster cũ.

## 4. API path

```text
kubectl trên tang3
  | kubeconfig: https://192.168.30.45:6443
  v
HAProxy tang3 :6443
  | backend derived từ control_plane inventory
  v
K3s API tang4 :6443
  |
  v
SQLite tang4
```

HAProxy không terminate TLS. `192.168.30.45` nằm trong K3s TLS SAN nên client có thể dùng stable endpoint.

Direct backend test và stable endpoint test là hai proof khác nhau:

```bash
curl -k https://192.168.30.35:6443/ping
curl -k https://192.168.30.45:6443/ping
```

Cả hai phải trả `pong`.

## 5. Application data path

```text
Client / Cloudflare Tunnel
  | Host header + path
  v
HAProxy tang3 :80/:443
  |
  v
ServiceLB host ports tang4 :80/:443
  |
  v
Traefik LoadBalancer Service
  |
  v
Traefik Ingress Controller
  | match host/path
  v
ClusterIP Service
  | select Ready endpoints
  v
Application Pod tang4
```

HAProxy chỉ biết tang4. Nó không cần backend riêng cho mỗi application. Traefik watch Ingress resources và phân loại request theo hostname/path.

## 6. ServiceLB placement

Tang4 mang labels:

```text
svccontroller.k3s.cattle.io/enablelb=true
svccontroller.k3s.cattle.io/lbpool=ingress
```

Traefik LoadBalancer Service mang label pool `ingress` từ `HelmChartConfig`. ServiceLB vì vậy tạo `svclb-traefik` trên tang4 và bind host ports `80/443`.

Placement validation không dựa vào Kubernetes role label `worker`; nó dựa vào abstraction `k3s_workload_node_name` và node name thực tế.

## 7. Automation sequence

```text
preflight
  -> HAProxy API backend = tang4
  -> K3s server tang4
  -> Traefik / ServiceLB configuration
  -> HAProxy application listeners
  -> smoke workload
  -> complete validation
```

One-time recovery đi trước sequence này:

```text
explicit approval guard
  -> verify host identity
  -> verify no unrelated runtime
  -> official k3s-agent uninstall
  -> remove known K3s leftovers/interfaces
  -> verify old agent process is gone
```

## 8. Resource layers

| Lớp | Source of truth | Lifecycle |
| --- | --- | --- |
| Host/platform | `inventories/`, `roles/`, `playbooks/` | `site.yml` |
| Traefik configuration | `manifests/traefik/` | `k3s_addons` role |
| Platform smoke proof | `manifests/smoke-test/` | `site.yml` |
| Demo Nginx | `manifests/app/` | `app.yml` |
| App1 microservices | repository `Kien-devops/app1` | GitHub Actions CD |

Platform repo không deploy app1 release. App repo không cài K3s hoặc sửa HAProxy.

## 9. Network ranges

- LAN/node network: `192.168.30.0/24`.
- Pod CIDR: `10.42.0.0/16`.
- Service CIDR: `10.43.0.0/16`.
- Tailscale: management-only, không nằm trong K3s/HAProxy data path.

HAProxy chỉ dùng LAN IP. Kubernetes networking xử lý Pod và Service CIDRs bên trong tang4.

## 10. Failure boundaries

| Failure | Impact |
| --- | --- |
| tang4 mất | API, SQLite, scheduling, ingress và applications đều mất |
| tang3 mất | Stable API/app entry point, tunnel và monitoring mất; K3s vẫn chạy local trên tang4 |
| DHCP tang4 đổi IP | HAProxy backend và node config drift cho tới khi inventory được cập nhật |
| DHCP tang3 đổi IP | Kubeconfig/TLS endpoint và client routing cần reconcile |
| tang4 disk lỗi | SQLite và toàn bộ cluster state có nguy cơ mất |

HAProxy cung cấp stable endpoint, không tạo control-plane HA. Single-node SQLite là single point of failure có chủ đích trong homelab.
