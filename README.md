# K3s On-Prem Platform

Đồ án xây dựng một nền tảng Kubernetes on-premises quy mô nhỏ bằng K3s và Ansible, phục vụ triển khai microservices theo quy trình CI/CD hoàn chỉnh. Hệ thống tập trung vào khả năng tái lập hạ tầng, phân tách vai trò máy chủ, dự phòng workload/ingress và vận hành qua một endpoint ổn định.

Repository này quản lý platform. Source code và vòng đời release của ứng dụng kiểm chứng nằm tại [`Kien-devops/app1`](https://github.com/Kien-devops/app1).

## Mục tiêu đồ án

- Xây dựng cluster K3s có control plane và worker tách biệt.
- Tự động hóa cấu hình hạ tầng bằng Ansible theo hướng idempotent.
- Cung cấp một API endpoint và application edge ổn định qua HAProxy.
- Phân tán ingress controller và application replica trên hai worker.
- Triển khai ứng dụng bằng immutable image gắn với Git commit SHA.
- Kiểm chứng toàn bộ data path từ Internet đến Kubernetes Service và Pod.

## Kiến trúc hệ thống

```text
                         Internet
                            |
                    Cloudflare Tunnel
                            |
                            v
              server-tang3 - 192.168.30.45
       Ansible / HAProxy / Monitoring / GitHub Runner
                  |                         |
          Kubernetes API :6443       Application :80/:443
                  |                         |
                  v                  +------+------+
       server-tang2                  |             |
       192.168.30.44                 v             v
  K3s control plane + SQLite   server-tang1   server-tang4
                              192.168.30.200  192.168.30.35
                              K3s worker      K3s worker
                              ServiceLB       ServiceLB
                              Traefik         Traefik
                              app1 replica    app1 replica
```

| Host | Vai trò trong đồ án |
| --- | --- |
| `server-tang2` | K3s server, Kubernetes API, scheduler/controller và SQLite datastore |
| `server-tang1` | K3s worker, ServiceLB, Traefik và application workloads |
| `server-tang4` | K3s worker, ServiceLB, Traefik, application workloads và platform smoke workloads |
| `server-tang3` | Automation controller, HAProxy, Cloudflare Tunnel, monitoring và self-hosted GitHub Actions runner |

Tang3 không tham gia cluster K3s. Tailscale chỉ phục vụ management; node traffic, Flannel và HAProxy backend sử dụng mạng LAN.

## Thiết kế kỹ thuật

### Cluster topology

Control plane chạy độc lập trên tang2 với `NoSchedule` taint. Tang1 và tang4 thuộc hai inventory group `workers` và `ingress_workers`; cả hai được gắn labels cho Traefik node selection và K3s ServiceLB pool.

Cluster sử dụng Flannel VXLAN, Pod CIDR `10.42.0.0/16` và Service CIDR `10.43.0.0/16`. Application Services giữ kiểu `ClusterIP`, không public trực tiếp từ namespace ứng dụng.

### Ingress availability

Traefik chạy hai replica với hard topology spread theo `kubernetes.io/hostname`. Rolling update sử dụng `maxSurge: 0` và `maxUnavailable: 1` để duy trì một replica trên mỗi failure domain sau khi thay ReplicaSet.

ServiceLB bind cổng `80/443` trên cả tang1 và tang4. HAProxy health-check hai backend và loại node lỗi khỏi vòng cân bằng tải.

```text
Client
  -> Cloudflare Tunnel trên tang3
  -> HAProxy tang3
  -> ServiceLB tang1 hoặc tang4
  -> Traefik
  -> Ingress
  -> ClusterIP Service
  -> Application Pod
```

### Application placement

Mỗi Deployment của `app1` chạy hai replica và sử dụng topology spread theo hostname. Pipeline coi release là không đạt nếu hai replica của cùng Deployment nằm trên một node. Mô hình mục tiêu là một Pod trên tang1 và một Pod trên tang4 để workload không phụ thuộc vào một worker duy nhất.

Smoke test và manifest `public-app` của platform vẫn được pin vào tang4 vì chúng dùng để kiểm chứng placement cố định, không đại diện cho chiến lược scheduling của `app1`.

## Automation và delivery

| Phạm vi | Repository chịu trách nhiệm |
| --- | --- |
| Inventory, K3s, HAProxy, Traefik, ServiceLB, validation | `k3s-onprem` |
| Source, test, image, manifests, rollout, application acceptance | `app1` |

Luồng triển khai hạ tầng:

```text
Inventory
  -> preflight
  -> HAProxy API endpoint
  -> K3s control plane
  -> rolling worker convergence
  -> Traefik / ServiceLB
  -> HAProxy ingress backends
  -> smoke test
  -> platform validation
```

Luồng phát hành ứng dụng:

```text
Git push
  -> component tests
  -> container build
  -> security scan
  -> GHCR image:<full-git-sha>
  -> self-hosted deploy runner trên tang3
  -> Kubernetes rolling deployment
  -> placement validation
  -> route acceptance test
```

Runner chỉ sử dụng namespace-scoped kubeconfig. Pipeline ứng dụng không quản lý cluster-level resources và không sử dụng admin kubeconfig.

## Availability model

| Failure domain | Hành vi hệ thống |
| --- | --- |
| Mất tang1 hoặc tang4 | HAProxy loại backend lỗi; ingress và `app1` tiếp tục trên worker còn lại với mức redundancy giảm |
| Mất tang2 | Workload đang chạy có thể tiếp tục tạm thời, nhưng API, scheduling và reconciliation không còn khả dụng |
| Mất tang3 | Cluster nội bộ vẫn chạy; public edge, stable API endpoint, monitoring và CD runner mất |
| Thay đổi DHCP | Inventory, certificate endpoint và HAProxy backend có nguy cơ drift |

Hai ingress worker cung cấp HA ở tầng worker nhưng không biến hệ thống thành Kubernetes HA hoàn chỉnh. Tang2 và tang3 vẫn là các single failure domain. Thiết kế HA đầy đủ cần nhiều K3s server dùng embedded etcd hoặc external datastore, đồng thời bổ sung edge/load balancer dự phòng.

## Security baseline

- Kubernetes API `:6443` không được expose ra Internet.
- Control plane không nhận application workload.
- Application pipeline sử dụng namespace-scoped RBAC.
- Image được deploy theo immutable Git SHA, không phụ thuộc tag `latest`.
- Secret, token, kubeconfig, private key và `server.txt` không được commit.
- HAProxy image, K3s, Ansible và collection dependencies được pin version.
- DHCP reservation được duy trì cho `.44`, `.200`, `.45` và `.35`.

## Technology baseline

| Thành phần | Phiên bản / lựa chọn |
| --- | --- |
| K3s | `v1.36.4+k3s1` |
| k3s-ansible | `k3s.orchestration` `1.2.2` |
| ansible-core | `2.21.4` |
| HAProxy | `haproxy:3.2.24-alpine3.24` |
| Datastore | SQLite |
| CNI | Flannel VXLAN |
| Ingress | Traefik bundled với K3s |
| Registry | GitHub Container Registry |
| CI/CD | GitHub Actions |

## Trạng thái kiểm chứng

Runtime acceptance gần nhất xác nhận:

- tang1, tang2 và tang4 ở trạng thái `Ready`;
- Traefik có hai replica, phân tán một Pod trên mỗi worker;
- ServiceLB có một Pod trên mỗi worker;
- HAProxy đánh dấu cả hai HTTP/HTTPS backend `UP`;
- Kubernetes API trực tiếp và qua HAProxy đều trả `pong`;
- các route frontend, Auth, User và Product đều trả HTTP `200` qua từng worker và HAProxy;
- public endpoint `https://app1.onprem.site` hoạt động qua Cloudflare Tunnel.

Automation yêu cầu SSH identity và `become` credential hợp lệ trên các host đích. Credential thuộc phạm vi vận hành, không được lưu trong repository.

## Cấu trúc repository

```text
inventories/production/     Inventory, group vars và host vars
playbooks/                  Cluster lifecycle và validation
roles/                      Ansible roles của platform
manifests/                  Traefik, smoke test và public app
docs/                       Architecture, deployment và operations runbooks
collections/                Pinned Ansible collections
```

## Tài liệu kỹ thuật

- [Architecture](docs/architecture.md)
- [Deployment and recovery](docs/deployment.md)
- [Application CI/CD](docs/application-cicd.md)
- [`app1` runtime design](docs/app1-microservices.md)
- [Troubleshooting](docs/troubleshooting.md)
