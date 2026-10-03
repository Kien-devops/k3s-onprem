# Kiến trúc K3s On-Prem Homelab

## 1. Mục tiêu thiết kế

Kiến trúc này ưu tiên ba mục tiêu: dễ quan sát luồng traffic, tự động hóa lặp lại bằng Ansible và không làm hỏng các Docker workload đã tồn tại. Đây là môi trường học tập disposable (có thể dựng lại), không phải production platform.

Các quyết định chính:

- Một K3s server trên tang2 giữ Kubernetes API và SQLite datastore.
- Một K3s agent trên tang4 chạy Traefik, ServiceLB và application Pods.
- HAProxy trên tang3 là entry point chung từ LAN cho cả API và application.
- Ansible cũng chạy trên tang3 để không cần thêm một controller riêng.
- Control plane bị taint để workload thông thường không chiếm tài nguyên tang2.
- Toàn bộ Kubernetes node traffic dùng LAN IP, không dùng Tailscale IP.

## 2. Sơ đồ vật lý và trách nhiệm

```text
                           LAN 192.168.30.0/24
                                    |
                                    v
                 +--------------------------------------+
                 | server-tang3                         |
                 | 192.168.30.45                        |
                 |                                      |
                 | - Ansible Controller                 |
                 | - HAProxy Docker container           |
                 | - Prometheus/Grafana hiện hữu        |
                 +------------------+-------------------+
                                    |
                      +-------------+-------------+
                      |                           |
                 API TCP 6443              App TCP 80/443
                      |                           |
                      v                           v
          +------------------------+  +------------------------+
          | server-tang2           |  | server-tang4           |
          | 192.168.30.44          |  | 192.168.30.35          |
          |                        |  |                        |
          | K3s server             |  | K3s agent              |
          | kube-apiserver         |  | ServiceLB              |
          | SQLite                 |  | Traefik                |
          | Docker apps hiện hữu   |  | Application Pods       |
          +------------------------+  +------------------------+
                      \                 /
                       \_ Flannel VXLAN/
```

| Thành phần | Repo quản lý? | Ghi chú |
| --- | --- | --- |
| K3s server và agent | Có | Việc cài đặt giao cho collection `k3s.orchestration` đã pin version |
| HAProxy container/config | Có | Validate config trước khi recreate container |
| Traefik configuration | Có | Dùng Traefik được bundle cùng K3s |
| ServiceLB placement | Có | Node labels và Service pool giới hạn ở tang4 |
| Smoke workload | Có | Được `site.yml` dùng để chứng minh đường ingress |
| Demo application | Có, lifecycle riêng | Chỉ được deploy bởi `playbooks/app.yml` |
| Docker apps trên tang2 | Không | Chỉ xác minh chúng còn chạy |
| Prometheus/Grafana trên tang3 | Không | Preflight và validation bảo vệ, không reconfigure |
| DHCP, switch/router, Tailscale | Không | Hạ tầng mạng bên ngoài repo |

## 3. Mô hình Ansible

### 3.1. Inventory là WHO

`inventories/production/hosts.yml` ánh xạ hostname logic sang LAN IP và nhóm trách nhiệm:

```text
control_plane  -> server-tang2 -> 192.168.30.44
workers        -> server-tang4 -> 192.168.30.35
load_balancers -> server-tang3 -> 192.168.30.45
```

Nhóm `server` và `agent` là alias phục vụ collection upstream. Nhóm `k3s_cluster` gộp control plane và workers cho các task baseline chung.

`group_vars` mô tả biến dùng chung theo vai trò; `host_vars` chứa khác biệt từng máy như SSH user và LAN interface. Cách tổ chức này tránh lặp IP trong role/template: HAProxy backend và API endpoint đều được suy ra từ inventory groups.

### 3.2. Role là HOW

Mỗi role có một trách nhiệm hẹp:

```text
preflight         -> kiểm tra điều kiện và bảo vệ workload hiện hữu
k3s_control_plane -> hội tụ K3s server, token và kubeconfig
k3s_worker        -> hội tụ agent, join cluster và kiểm tra labels
haproxy           -> render/validate config và quản lý container
k3s_addons        -> cấu hình placement cho Traefik/ServiceLB
validation        -> chứng minh trạng thái Kubernetes và traffic ngoài cluster
```

Hai role K3s là wrapper quanh upstream roles. Wrapper quyết định khi nào cần gọi upstream dựa trên version, config, unit và service state. Điều này vừa tái sử dụng installer chuẩn, vừa giữ các quy tắc riêng của homelab trong code cục bộ.

### 3.3. Playbook là WHAT/WHEN

`playbooks/site.yml` là orchestration chính:

```text
1. preflight trên cả ba máy
2. HAProxy API endpoint trên tang3
3. K3s control plane trên tang2
4. K3s worker trên tang4
5. Traefik/ServiceLB addons từ tang2
6. HAProxy ingress listeners trên tang3
7. smoke workload và HTTP test
8. validation toàn hệ thống
```

Thứ tự này là dependency graph: worker cần API endpoint và token; addon placement chỉ được kiểm tra sau khi worker Ready; smoke test chỉ chạy sau khi ingress listener tồn tại.

`playbooks/app.yml` không được import vào `site.yml`. Demo app là user workload, không phải điều kiện để cluster được xem là dựng thành công.

### 3.4. Manifest là WHAT RUNS

Ansible tạo hoặc hội tụ host-level state. Kubernetes manifests mô tả desired state bên trong cluster:

- `manifests/traefik/helmchartconfig.yml`: cluster configuration.
- `manifests/smoke-test/nginx.yml`: validation workload.
- `manifests/app/*.yml`: demo user workload.

Tách các lớp này ngăn việc thay đổi một demo app bị nhầm với thay đổi K3s/HAProxy infrastructure.

## 4. Luồng Kubernetes API

```text
kubectl / K3s agent
       |
       | TLS tới 192.168.30.45:6443
       v
HAProxy frontend k3s_api trên tang3
       |
       | TCP pass-through
       v
HAProxy backend 192.168.30.44:6443
       |
       v
K3s API server trên tang2
       |
       v
SQLite datastore trên tang2
```

Kubeconfig trên tang3 được fetch từ `/etc/rancher/k3s/k3s.yaml`, sau đó endpoint loopback được đổi thành `https://192.168.30.45:6443`. K3s certificate có `tls-san` cho IP tang3 nên client có thể xác minh API qua endpoint này.

Tang4 cũng join qua endpoint HAProxy. Node token được đọc từ tang2 với `no_log` và chỉ dùng trong quá trình hội tụ agent.

### Endpoint ổn định không phải High Availability

HAProxy tách địa chỉ client khỏi địa chỉ trực tiếp của API server. Tuy nhiên backend chỉ có một tang2:

- tang2 hỏng: API mất vì không có control plane khác;
- tang3 hỏng: endpoint ngoài mất dù process K3s trên tang2 có thể còn chạy;
- SQLite trên tang2 không được replicate.

Muốn HA thật cần nhiều control-plane node, datastore quorum như embedded etcd, và load-balancer endpoint cũng phải có cơ chế dự phòng. Các thành phần đó nằm ngoài phạm vi lab này.

## 5. Scheduling và placement

### 5.1. Bảo vệ control plane

Tang2 có taint:

```text
node-role.kubernetes.io/control-plane=true:NoSchedule
```

Pod không có toleration tương ứng sẽ không được schedule lên tang2. Điều này giữ control plane tập trung vào API và system services, đồng thời bảo vệ Docker workloads hiện hữu khỏi cạnh tranh không cần thiết.

### 5.2. Đặt ServiceLB trên worker

Tang4 có hai labels:

```text
svccontroller.k3s.cattle.io/enablelb=true
svccontroller.k3s.cattle.io/lbpool=ingress
```

`HelmChartConfig` thêm label `svccontroller.k3s.cattle.io/lbpool=ingress` vào Traefik Service. ServiceLB controller ghép Service với node cùng pool, nên Pod ServiceLB chiếm host ports `80/443` trên tang4 thay vì tang2.

### 5.3. Đặt application Pod trên worker

Cả smoke Deployment và demo Deployment đều dùng:

```yaml
nodeSelector:
  kubernetes.io/hostname: server-tang4
```

Demo Deployment có hai replicas. Scheduler vẫn tạo cả hai trên cùng một worker vì cluster chỉ có một worker phù hợp. Hai replicas giúp quan sát Service load balancing và rolling behavior, nhưng không cung cấp node-level HA.

## 6. Luồng application traffic

### 6.1. Vai trò từng lớp

| Lớp | Trách nhiệm | Không làm gì |
| --- | --- | --- |
| HAProxy | Nhận TCP `80/443`, chuyển tới tang4 | Không đọc Kubernetes Ingress, không biết Pod |
| ServiceLB | Expose Traefik LoadBalancer Service trên host ports tang4 | Không route theo hostname |
| Traefik | Watch Ingress và route theo hostname/path | Không quản lý lifecycle của app Pod |
| Ingress | Khai báo rule `host/path -> Service` | Không phải process nhận traffic |
| ClusterIP Service | Cấp virtual IP và chọn Ready Pod bằng label | Không expose trực tiếp ra LAN |
| Deployment | Duy trì số lượng Pod và rollout | Không cung cấp stable network endpoint |
| Pod | Chạy Nginx container và trả response | IP Pod có thể thay đổi |

Ingress là dữ liệu cấu hình trong Kubernetes; Traefik mới là controller biến dữ liệu đó thành routing thực tế.

### 6.2. Luồng 11 bước của demo app

```bash
curl -H 'Host: nginx.apps.k3s.home.arpa' http://192.168.30.45/
```

1. Client mở TCP connection tới `192.168.30.45:80`.
2. HTTP request mang hostname `nginx.apps.k3s.home.arpa`.
3. HAProxy frontend `application_http` nhận connection.
4. HAProxy backend chuyển connection tới `192.168.30.35:80`.
5. ServiceLB Pod trên tang4 nhận host port `80`.
6. Traffic vào Traefik LoadBalancer Service.
7. Traefik nhận HTTP request và đọc `Host`/path.
8. Traefik match Ingress `public-app/nginx`.
9. Ingress trỏ tới ClusterIP Service `public-app/nginx`.
10. Service chọn một Ready Pod có label `app=nginx`.
11. Nginx trả response về client theo connection ban đầu.

```text
Client
  |
  | nginx.apps.k3s.home.arpa
  v
HAProxy tang3 :80/:443
  |
  v
ServiceLB tang4 :80/:443
  |
  v
Traefik
  |
  | Ingress host/path rule
  v
ClusterIP Service
  |
  | selector app=nginx
  +--------------+
  |              |
  v              v
Nginx Pod A    Nginx Pod B
```

HAProxy không thay đổi khi thêm application hostname. App mới chỉ cần Deployment, Service và Ingress phù hợp; Traefik tự watch object mới và cập nhật routing.

## 7. Mạng bên trong cluster

Ba loại địa chỉ không nên nhầm lẫn:

- Node/LAN IP `192.168.30.0/24`: kết nối vật lý giữa ba server và client.
- Pod CIDR `10.42.0.0/16`: IP động của Pod qua Flannel VXLAN.
- Service CIDR `10.43.0.0/16`: virtual IP ổn định của Kubernetes Service.

HAProxy chỉ dùng LAN IP. Nó không route trực tiếp tới Pod CIDR hay Service CIDR. Từ Traefik trở vào, Kubernetes networking xử lý Service và Pod endpoints.

Tailscale có thể dùng để quản trị SSH, nhưng không nằm trong data path của K3s trong thiết kế này.

## 8. Smoke workload và demo workload

### Smoke workload

- Namespace `lab-demo`.
- Một Nginx replica.
- Nội dung cố định `K3s homelab smoke test` từ ConfigMap.
- Được copy vào K3s static manifests directory.
- Được `site.yml` và validation kiểm tra qua HTTP/HTTPS.

Mục tiêu của nó là chứng minh chuỗi HAProxy -> ServiceLB -> Traefik -> Service -> Pod hoạt động mà không phụ thuộc demo app.

### Demo workload

- Namespace `public-app`.
- Hai Nginx replicas từ public image `nginx:alpine`.
- Readiness probe và nodeSelector tang4.
- Bốn manifest tách riêng để quan sát resource boundaries.
- Chỉ deploy bằng `playbooks/app.yml`.

Demo dùng cho việc học lifecycle ứng dụng. Nó không phải cluster addon và không được dùng làm điều kiện dựng cluster.

## 9. HAProxy lifecycle và idempotency

HAProxy chạy container với host networking. Role thực hiện theo chuỗi:

```text
kiểm tra image
  -> pull nếu thiếu
  -> đọc config hiện hữu
  -> quyết định có giữ/render ingress hay không
  -> render template kèm haproxy -c validation
  -> notify handler nếu config đổi
  -> recreate container khi cần
  -> chờ listeners
```

Handler chỉ chạy khi template thay đổi hoặc container thiếu/dừng. Vì vậy một playbook run không có drift không restart HAProxy. Trong phase API đầu tiên, config ingress hiện hữu được giữ lại để tránh đóng tạm thời `80/443`.

## 10. Failure boundaries

| Sự cố | Ảnh hưởng dự kiến |
| --- | --- |
| Tang2 mất | Kubernetes API/control plane mất; workload đang chạy có thể tồn tại tạm thời nhưng không được reconcile |
| Tang3 mất | LAN client mất API endpoint và application entry point; monitoring trên tang3 cũng mất |
| Tang4 mất | Traefik, ServiceLB và application workloads mất |
| DHCP đổi IP | Inventory và HAProxy/kubeconfig có thể trỏ sai cho tới khi cập nhật và converge lại |
| Registry/internet lỗi | Pod mới dùng public image có thể `ImagePullBackOff`; workload đã cache image có thể vẫn chạy |
| Disk tang2 phát sinh I/O error | K3s/SQLite có thể thất bại; dừng recovery tự động và thay disk |

Đây là các single points of failure có chủ đích trong lab. Không nên mô tả kiến trúc này là production-ready hoặc HA.
