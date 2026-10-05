# Ứng dụng microservices `app1` trên K3s

Tài liệu này mô tả ứng dụng thực tế đang chạy trên cluster K3s do repository `k3s-onprem` quản lý. Source code và lifecycle release của ứng dụng nằm tại repository [`Kien-devops/app1`](https://github.com/Kien-devops/app1).

## 1. Ứng dụng làm gì?

`app1` là ứng dụng demo dùng để kiểm chứng đầy đủ chuỗi microservices và CI/CD trên homelab:

- frontend hiển thị trạng thái các service;
- Auth service mô phỏng session và login;
- User service trả user profile mẫu;
- Product service trả danh mục sản phẩm mẫu;
- mỗi component có một container image độc lập;
- GitHub Actions tự động test, scan, publish và deploy.

```text
Source code
  -> automated tests
  -> container build
  -> vulnerability scan
  -> GHCR
  -> K3s rolling deployment
  -> internal và public validation
```

Auth API chỉ là demo, không phải hệ thống danh tính production.

## 2. Phân chia trách nhiệm giữa hai repository

### `k3s-onprem`

Repository platform quản lý:

- inventory của ba host thuộc K3s platform;
- K3s control plane/SQLite trên tang2, ingress worker trên tang4 và compute worker trên tang1;
- Kubernetes API endpoint qua HAProxy;
- Traefik và ServiceLB;
- host-level validation và smoke workload;
- network path cần thiết để application hoạt động.

### `app1`

Repository ứng dụng quản lý:

- frontend và ba backend service;
- Dockerfile và test;
- bốn GHCR images;
- namespace `microservices-demo`;
- Deployments, Services, Ingress và NetworkPolicies;
- namespace-scoped RBAC cho CD;
- GitHub Actions workflow;
- rollout và application acceptance test.

Ranh giới này giúp application release không chạy lại Ansible, không sửa HAProxy và không tác động workload khác.

## 3. Runtime topology

| Thành phần | Vị trí / cấu hình |
| --- | --- |
| K3s control plane | `server-tang2` - `192.168.30.44` |
| Application worker | `server-tang4` - `192.168.30.35` |
| General compute worker | `server-tang1` - `192.168.30.200` |
| Edge, HAProxy và CD runner | `server-tang3` - `192.168.30.45` |
| Namespace | `microservices-demo` |
| Ingress hostname | `app1.onprem.site` |
| Ingress class | `traefik` |
| Container registry | `ghcr.io/kien-devops/app1` |
| Public endpoint | `https://app1.onprem.site` |

```text
Internet client
    |
    | HTTPS
    v
Cloudflare Edge
    |
    | outbound Cloudflare Tunnel
    v
cloudflared server-tang3
    |
    v
HAProxy server-tang3 :80
    |
    v
ServiceLB :80 trên server-tang4
    |
    v
Traefik Ingress Controller
    |
    +-- /                  -> Service frontend
    +-- /api/auth/*        -> Service auth-service
    +-- /api/users/*       -> Service user-service
    `-- /api/products/*    -> Service product-service
```

HAProxy không có backend riêng cho `app1`. Nó chuyển application traffic tới tang4; Traefik đọc hostname/path và chọn Service tương ứng.

## 4. Kubernetes workloads

| Deployment | Replicas | Port | Service | Chức năng |
| --- | ---: | ---: | --- | --- |
| `frontend` | 2 | `8080` | `ClusterIP` | Dashboard HTML/CSS/JavaScript |
| `auth-service` | 1 | `8080` | `ClusterIP` | Demo auth/session API |
| `user-service` | 1 | `8080` | `ClusterIP` | User profile API |
| `product-service` | 1 | `8080` | `ClusterIP` | Product catalogue API |

Các Deployment dùng `nodeSelector` để chạy trên `server-tang4`. Frontend có hai replicas để chứng minh Service load balancing và zero-unavailable rolling update; đây không phải node-level high availability vì cả hai Pod vẫn ở cùng worker.

Image reference có dạng:

```text
ghcr.io/kien-devops/app1/frontend:<full-git-sha>
ghcr.io/kien-devops/app1/auth-service:<full-git-sha>
ghcr.io/kien-devops/app1/user-service:<full-git-sha>
ghcr.io/kien-devops/app1/product-service:<full-git-sha>
```

Không deploy tag `latest`. Git SHA là liên kết giữa source, build artifact và workload đang chạy.

## 5. Application routing

| Path | Kubernetes Service | Endpoint tiêu biểu |
| --- | --- | --- |
| `/` | `frontend` | Dashboard |
| `/api/auth` | `auth-service` | `/api/auth/health`, `/api/auth/session` |
| `/api/users` | `user-service` | `/api/users/health`, `/api/users/profile` |
| `/api/products` | `product-service` | `/api/products/health`, `/api/products` |

Browser gọi relative path `/api/...`, vì vậy frontend và APIs có cùng origin. Backend không cần NodePort và ứng dụng không cần CORS giữa nhiều domain.

## 6. Health, logs và metrics

Mỗi component cung cấp `/health` cho readiness/liveness probe. Kubernetes chỉ đưa Pod vào EndpointSlice sau khi readiness probe thành công.

Các backend còn cung cấp `/metrics` theo Prometheus text format và ghi JSON log gồm:

- timestamp;
- service;
- method và path;
- HTTP status;
- latency;
- request ID.

NetworkPolicy chưa cho Prometheus scrape trực tiếp. Khi tích hợp monitoring phải bổ sung allow rule có chủ đích tới application port `8080`.

## 7. Security boundary

Application workloads áp dụng:

- non-root UID/GID;
- `allowPrivilegeEscalation: false`;
- `readOnlyRootFilesystem: true`;
- drop toàn bộ Linux capabilities;
- `seccompProfile: RuntimeDefault`;
- không mount ServiceAccount token vào application Pod;
- CPU/memory requests và limits;
- default-deny ingress/egress NetworkPolicy.

Traefik trong trusted namespace `kube-system` được phép kết nối tới port `8080`. Các Service vẫn là `ClusterIP` và không trực tiếp public.

CD dùng kubeconfig riêng:

```text
/home/monitor/.kube/microservices-demo-deployer.config
```

Identity này có quyền quản lý workload cần thiết trong `microservices-demo`, nhưng không được:

- đọc Secrets;
- tạo hoặc sửa RoleBinding;
- quản lý Namespace;
- truy cập Pods trong namespace `default`;
- thay đổi cluster-level resource.

Admin kubeconfig `/home/monitor/.kube/config` không được dùng trong application pipeline.

## 8. CI/CD release flow

Workflow nằm tại `.github/workflows/ci-cd.yml` trong repository `app1`.

```text
Pull request
  -> 4 test jobs
  -> 4 build jobs
  -> Trivy scan
  -> không deploy

Push main
  -> 4 test jobs
  -> 4 build + Trivy jobs
  -> push exact-SHA images lên GHCR
  -> self-hosted runner server-tang3-k3s-deploy
  -> render exact-SHA manifests
  -> kubectl apply
  -> rollout status
  -> HTTP acceptance test
```

GitHub-hosted runner xử lý test và Docker build. Self-hosted runner chỉ chạy deploy job từ protected `main`, giúp giảm phạm vi rủi ro trên server-tang3.

Runner truy cập Kubernetes API qua:

```text
self-hosted runner
  -> https://192.168.30.45:6443
  -> HAProxy tang3
  -> K3s API tang2
```

## 9. Kiểm tra trạng thái application

Chạy trên `server-tang3` với admin kubeconfig cho hoạt động read-only:

```bash
kubectl --kubeconfig /home/monitor/.kube/config \
  get deployments,pods,services,endpointslices,ingress \
  -n microservices-demo -o wide
```

Kết quả khỏe mạnh:

- frontend `2/2` Available;
- ba backend `1/1` Available;
- Pod `Running` và Ready trên `server-tang4`;
- frontend có hai ready endpoints;
- mỗi backend có một ready endpoint;
- Ingress host là `app1.onprem.site`.

Kiểm tra internal data path:

```bash
for path in / /api/auth/health /api/users/health /api/products/health; do
  curl --fail --show-error --silent \
    -H 'Host: app1.onprem.site' \
    "http://192.168.30.45${path}" >/dev/null
done
```

Kiểm tra public data path:

```bash
curl --fail https://app1.onprem.site/
curl --fail https://app1.onprem.site/api/auth/health
curl --fail https://app1.onprem.site/api/users/health
curl --fail https://app1.onprem.site/api/products/health
```

## 10. Chẩn đoán theo lớp

```text
Public DNS / Cloudflare Tunnel
  -> HAProxy :80
  -> ServiceLB / Traefik
  -> Ingress host/path
  -> ClusterIP Service
  -> EndpointSlice
  -> Pod readiness
  -> application logs
```

```bash
kubectl --kubeconfig /home/monitor/.kube/config \
  describe ingress microservices-demo -n microservices-demo

kubectl --kubeconfig /home/monitor/.kube/config \
  get endpointslices -n microservices-demo

kubectl --kubeconfig /home/monitor/.kube/config \
  get events -n microservices-demo --sort-by=.lastTimestamp

kubectl --kubeconfig /home/monitor/.kube/config \
  logs -n microservices-demo deployment/auth-service --tail=100
```

| Hiện tượng | Lớp kiểm tra đầu tiên |
| --- | --- |
| Public `404`, internal `200` | Cloudflare hostname/tunnel |
| Internal `404` | Ingress hostname hoặc path |
| `502/503` | Service selector, EndpointSlice, readiness |
| `ImagePullBackOff` | GHCR visibility và exact SHA |
| Rollout timeout | Pod events, probes và application logs |
| Deploy job không chạy | Runner service, labels và GitHub Environment |
| Deploy bị `Forbidden` | Namespace-scoped RBAC; không dùng admin để bypass |

## 11. Update và rollback

Release bình thường được thực hiện bằng cách merge hoặc push vào `main` của repository `app1`. `k3s-onprem` không deploy application release.

Rollback repeatable nhất:

1. Xác định known-good commit của `app1`.
2. Revert commit lỗi hoặc chạy lại release bằng known-good Git SHA.
3. Chờ toàn bộ Deployment rollout.
4. Kiểm tra cả internal và public route.

Rollback khẩn cấp một Deployment:

```bash
kubectl --kubeconfig /home/monitor/.kube/microservices-demo-deployer.config \
  rollout undo deployment/frontend -n microservices-demo
```

Nếu thay đổi ảnh hưởng API contract của nhiều service, phải rollback đồng bộ cả release thay vì undo riêng lẻ từng Deployment.

## 12. Giới hạn hiện tại

- Một control plane, một ingress worker và một HAProxy vẫn là các failure domain đơn lẻ; compute worker tang1 không làm kiến trúc này thành HA.
- Hai frontend Pod không bảo vệ khỏi sự cố mất `server-tang4`.
- Auth, User và Product dùng dữ liệu demo, chưa có database.
- Chưa có Horizontal Pod Autoscaler hoặc PodDisruptionBudget.
- Chưa có centralized tracing và alert rules riêng cho application.
- Deploy kubeconfig dùng long-lived ServiceAccount token và cần rotate định kỳ.

Đây là thiết kế phù hợp cho homelab và học CI/CD. Production thực tế cần nhiều control plane/worker, secret manager, short-lived identity, database HA, backup/restore, WAF/rate limiting và observability hoàn chỉnh.
