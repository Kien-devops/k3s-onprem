# Kiến trúc K3s on-prem

## Thành phần

```text
server-tang3 (192.168.30.45)
  Ansible / HAProxy / cloudflared / Prometheus / Grafana / GitHub runner
        | API 6443                     | App 80/443
        v                              +------------------------+
server-tang2 (192.168.30.44)           v                        v
  K3s server + SQLite        server-tang4               server-tang1
                             K3s agent                   K3s agent
                             ServiceLB + Traefik         ServiceLB + Traefik
                             app1 replica                app1 replica
```

Tang3 không chạy K3s. Tailscale phục vụ SSH management, không tham gia node IP, Flannel hoặc HAProxy backend.

## Control plane

Tang2 chạy `k3s.service`, SQLite và Kubernetes API. Cấu hình chính:

```yaml
node-name: server-tang2
node-ip: 192.168.30.44
flannel-iface: wlx347de4475af5
tls-san:
  - 192.168.30.45
node-taint:
  - node-role.kubernetes.io/control-plane=true:NoSchedule
```

`NoSchedule` giữ workload khỏi control plane. Node token chỉ được đọc runtime với `no_log` để upstream agent role join tang4 và tang1; token không được lưu trong Git.

## Worker và ingress

Tang4 và tang1 chạy `k3s-agent.service`, không giữ datastore. Cả hai thuộc inventory group `ingress_workers` và mang các labels:

```text
onprem.site/ingress=true
svccontroller.k3s.cattle.io/enablelb=true
svccontroller.k3s.cattle.io/lbpool=ingress
```

Traefik chạy hai replica với hard topology spread theo hostname, một Pod trên mỗi worker. Service dùng pool `ingress`, vì vậy ServiceLB bind `80/443` trên cả tang1 và tang4. HAProxy health-check và cân bằng hai backend; Application Services vẫn giữ type `ClusterIP`.

## Data paths

```text
kubectl/runner -> tang3:6443 -> HAProxy -> tang2:6443 -> K3s API

Internet -> Cloudflare Tunnel tang3 -> HAProxy tang3:80
         -> ServiceLB tang1 hoặc tang4 -> Traefik -> Ingress -> Service -> Pod
```

HAProxy không terminate TLS và không route theo hostname/path; Traefik thực hiện application routing.

## Failure domains

- Tang2 lỗi: API và SQLite mất; không thể schedule/reconcile/rollout.
- Tang4 lỗi: HAProxy loại backend tang4; ingress và `app1` tiếp tục trên tang1. Bundled smoke/`public-app` được pin tang4 sẽ mất.
- Tang1 lỗi: HAProxy loại backend tang1; ingress và `app1` tiếp tục trên tang4.
- Tang3 lỗi: K3s nội bộ vẫn tồn tại nhưng stable API, public edge, monitoring và CD runner mất.
- LAN/DHCP lỗi: inventory, kubeconfig và HAProxy backend có thể drift.

Hai ingress worker loại bỏ failure domain đơn tại tầng worker ingress, nhưng đây chưa phải HA toàn cluster. Tang2 vẫn là single control plane/SQLite và tang3 vẫn là single edge/HAProxy; HA đầy đủ cần nhiều K3s server với embedded etcd hoặc external datastore và edge/load balancer dự phòng.
