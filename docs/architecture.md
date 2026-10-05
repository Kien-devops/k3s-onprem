# Kiến trúc K3s on-prem

## Thành phần

```text
server-tang3 (192.168.30.45)
  Ansible / HAProxy / cloudflared / Prometheus / Grafana / GitHub runner
        | API 6443                     | App 80/443
        v                              v
server-tang2 (192.168.30.200)    server-tang4 (192.168.30.35)
  K3s server + SQLite              K3s agent + ServiceLB + Traefik + Pods
```

Tang3 không chạy K3s. Tailscale phục vụ SSH management, không tham gia node IP, Flannel hoặc HAProxy backend.

## Control plane

Tang2 chạy `k3s.service`, SQLite và Kubernetes API. Cấu hình chính:

```yaml
node-name: server-tang2
node-ip: 192.168.30.200
flannel-iface: wlx58044f3fffa6
tls-san:
  - 192.168.30.45
node-taint:
  - node-role.kubernetes.io/control-plane=true:NoSchedule
```

`NoSchedule` giữ workload trên tang4. Node token chỉ được đọc runtime với `no_log` để upstream agent role join tang4; token không được lưu trong Git.

## Worker và ingress

Tang4 chạy `k3s-agent.service`, không giữ datastore. Labels:

```text
svccontroller.k3s.cattle.io/enablelb=true
svccontroller.k3s.cattle.io/lbpool=ingress
```

Traefik Service dùng pool `ingress`, vì vậy ServiceLB bind `80/443` trên tang4. Application Services giữ type `ClusterIP`.

## Data paths

```text
kubectl/runner -> tang3:6443 -> HAProxy -> tang2:6443 -> K3s API

Internet -> Cloudflare Tunnel tang3 -> HAProxy tang3:80
         -> ServiceLB tang4 -> Traefik -> Ingress -> Service -> Pod
```

HAProxy không terminate TLS và không route theo hostname/path; Traefik thực hiện application routing.

## Failure domains

- Tang2 lỗi: API và SQLite mất; không thể schedule/reconcile/rollout.
- Tang4 lỗi: Ingress và toàn bộ application Pods mất.
- Tang3 lỗi: K3s nội bộ vẫn tồn tại nhưng stable API, public edge, monitoring và CD runner mất.
- LAN/DHCP lỗi: inventory, kubeconfig và HAProxy backend có thể drift.

Đây là tách role để dễ hiểu và vận hành, không phải HA. HA thật cần nhiều K3s server với embedded etcd hoặc external datastore, nhiều workers và redundant edge.
