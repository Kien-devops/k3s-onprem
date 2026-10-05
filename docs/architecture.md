# Kiến trúc K3s on-prem

## Thành phần

```text
server-tang3 (192.168.30.45)
  Ansible / HAProxy / cloudflared / Prometheus / Grafana / GitHub runner
        | API 6443                     | App 80/443
        v                              v
server-tang2 (192.168.30.44)     server-tang4 (192.168.30.35)
  K3s server + SQLite              K3s agent + ServiceLB + Traefik + pinned Pods
                                      |
                                server-tang1 (192.168.30.200)
                                  K3s compute agent
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

Tang4 và tang1 chạy `k3s-agent.service`, không giữ datastore. Chỉ tang4 mang các ingress labels:

```text
svccontroller.k3s.cattle.io/enablelb=true
svccontroller.k3s.cattle.io/lbpool=ingress
```

Traefik Deployment được pin vào tang4 và Service dùng pool `ingress`, vì vậy ServiceLB chỉ bind `80/443` trên tang4. Tang1 là compute worker cho workload không có nodeSelector; Application Services giữ type `ClusterIP`.

## Data paths

```text
kubectl/runner -> tang3:6443 -> HAProxy -> tang2:6443 -> K3s API

Internet -> Cloudflare Tunnel tang3 -> HAProxy tang3:80
         -> ServiceLB tang4 -> Traefik -> Ingress -> Service -> Pod
```

HAProxy không terminate TLS và không route theo hostname/path; Traefik thực hiện application routing.

## Failure domains

- Tang2 lỗi: API và SQLite mất; không thể schedule/reconcile/rollout.
- Tang4 lỗi: Ingress và các application Pods đang được pin vào tang4 mất; tang1 không tự thay thế ingress.
- Tang1 lỗi: giảm compute capacity nhưng ingress trên tang4 vẫn hoạt động.
- Tang3 lỗi: K3s nội bộ vẫn tồn tại nhưng stable API, public edge, monitoring và CD runner mất.
- LAN/DHCP lỗi: inventory, kubeconfig và HAProxy backend có thể drift.

Đây là tách role để dễ hiểu và vận hành, không phải HA. Hai worker chưa loại bỏ single point of failure tại control plane, ingress worker và edge; HA thật cần nhiều K3s server với embedded etcd hoặc external datastore, ingress trên nhiều worker và redundant edge.
