# K3s On-Prem Homelab bằng Ansible

Repository này tự động hóa một cụm K3s nhỏ trên ba máy Ubuntu trong cùng mạng LAN. Mục tiêu là học cách các lớp Ansible, K3s, HAProxy, ServiceLB, Traefik và Kubernetes workload nối với nhau trong một hệ thống thật, nhưng vẫn giữ kiến trúc đủ đơn giản cho homelab.

Đây là lab dùng một control plane, không phải kiến trúc production và không cung cấp High Availability (HA - tính sẵn sàng cao). HAProxy tạo endpoint ổn định cho Kubernetes API và application traffic; HAProxy không làm cho control plane trở thành HA.

## Tổng quan kiến trúc

```text
                         Máy quản trị / LAN Client
                                   |
                    +--------------+--------------+
                    |                             |
             Kubernetes API                 HTTP / HTTPS
                 TCP 6443                    TCP 80 / 443
                    |                             |
                    v                             v
          +------------------------------------------------+
          | server-tang3 - 192.168.30.45                   |
          |                                                |
          | Ansible Controller                             |
          | HAProxy container                              |
          | Prometheus / Grafana hiện hữu                  |
          +----------------------+-------------------------+
                                 |
                    +------------+------------+
                    |                         |
             nhánh API :6443          nhánh app :80/:443
                    |                         |
                    v                         v
     +-----------------------------+   +-----------------------------+
     | server-tang2                |   | server-tang4                |
     | 192.168.30.44               |   | 192.168.30.35               |
     |                             |   |                             |
     | K3s server / control plane  |   | K3s agent / worker          |
     | Kubernetes API             |   | ServiceLB                   |
     | SQLite datastore           |   | Traefik Ingress Controller  |
     | Docker workloads hiện hữu  |   | Application Pods            |
     +-----------------------------+   +-----------------------------+
                    \                         /
                     \____ Flannel VXLAN ____/
```

Sau HAProxy, traffic tách thành hai nhánh có mục đích khác nhau:

- Nhánh Kubernetes API: client dùng `192.168.30.45:6443`; HAProxy chuyển TCP tới K3s API trên `server-tang2:6443`.
- Nhánh application: client dùng `192.168.30.45:80` hoặc `:443`; HAProxy chuyển TCP tới `server-tang4`, sau đó ServiceLB và Traefik chọn ứng dụng theo Ingress hostname.

Các thuật ngữ cốt lõi trong repo:

- Control plane (mặt điều khiển): quản lý trạng thái cluster và cung cấp Kubernetes API.
- Worker node (nút thực thi): nơi scheduler đặt application Pods để chạy workload.
- Endpoint (điểm truy cập): địa chỉ IP/port ổn định mà client kết nối.
- Load balancer (bộ cân bằng/chuyển tiếp traffic): nhận kết nối ở một địa chỉ và đưa tới backend.
- Ingress (quy tắc định tuyến traffic vào cluster): ánh xạ hostname/path tới Kubernetes Service.
- Manifest (tệp khai báo trạng thái mong muốn): YAML mô tả Kubernetes resource cần tồn tại.
- Idempotency (tính chạy lặp an toàn): chạy lại automation khi không có drift sẽ không tạo thay đổi ngoài dự kiến.

| Nhóm inventory | Máy | LAN IP | Trách nhiệm |
| --- | --- | --- | --- |
| `control_plane` | `server-tang2` | `192.168.30.44` | K3s server, API, SQLite và system services |
| `load_balancers` | `server-tang3` | `192.168.30.45` | Ansible controller, HAProxy và monitoring hiện hữu |
| `workers` | `server-tang4` | `192.168.30.35` | K3s agent, ServiceLB, Traefik và application workloads |

Control plane có taint `node-role.kubernetes.io/control-plane=true:NoSchedule`, vì vậy workload thông thường không được schedule lên tang2. Demo application còn dùng `nodeSelector` để chỉ định rõ tang4.

## Công nghệ và thông số hiện tại

| Thành phần | Phiên bản / chế độ |
| --- | --- |
| K3s | `v1.36.4+k3s1` |
| Upstream k3s-ansible | `k3s.orchestration` phiên bản `1.2.2` |
| ansible-core | `2.21.4` |
| HAProxy | `haproxy:3.2.24-alpine3.24` chạy bằng Docker |
| Container runtime của K3s | containerd do K3s quản lý |
| Container Network Interface (CNI) | Flannel VXLAN |
| Datastore | SQLite trên một control plane |
| Ingress Controller | Traefik được bundle cùng K3s |
| Service exposure | K3s ServiceLB, giới hạn ở tang4 |
| Pod CIDR | `10.42.0.0/16` |
| Service CIDR | `10.43.0.0/16` |

## Cấu trúc repository

```text
.
|-- .gitignore
|-- ansible.cfg
|-- requirements.txt
|-- dien.md
|-- collections/
|   `-- requirements.yml
|-- inventories/
|   `-- production/
|       |-- hosts.yml
|       |-- group_vars/
|       |   |-- all.yml
|       |   |-- control_plane.yml
|       |   |-- workers.yml
|       |   `-- load_balancers.yml
|       `-- host_vars/
|           |-- server-tang2.yml
|           |-- server-tang3.yml
|           `-- server-tang4.yml
|-- playbooks/
|   |-- site.yml
|   |-- preflight.yml
|   |-- cluster.yml
|   |-- addons.yml
|   |-- smoke-test.yml
|   |-- validate.yml
|   `-- app.yml
|-- roles/
|   |-- preflight/
|   |-- k3s_control_plane/
|   |-- k3s_worker/
|   |-- haproxy/
|   |-- k3s_addons/
|   `-- validation/
|-- manifests/
|   |-- traefik/
|   |   `-- helmchartconfig.yml
|   |-- smoke-test/
|   |   `-- nginx.yml
|   `-- app/
|       |-- namespace.yml
|       |-- deployment.yml
|       |-- service.yml
|       `-- ingress.yml
`-- docs/
    |-- application-cicd.md
    |-- app1-microservices.md
    |-- architecture.md
    |-- deployment.md
    `-- troubleshooting.md
```

### Bốn khái niệm cần phân biệt

```text
Inventory = WHO       Máy nào được quản lý, thuộc nhóm nào, dùng biến nào
Role      = HOW       Cách một trách nhiệm được thực hiện và hội tụ trạng thái
Playbook  = WHAT/WHEN Role hoặc task nào chạy, trên nhóm nào, theo thứ tự nào
Manifest  = WHAT RUNS Tài nguyên nào thực sự chạy bên trong Kubernetes
```

Ví dụ: inventory xác định tang4 thuộc `workers`; role `k3s_worker` mô tả cách cài và join agent; `site.yml` quyết định role đó chạy sau control plane; manifest `deployment.yml` yêu cầu Kubernetes chạy hai Nginx Pod trên tang4.

## Inventory: xác định WHO

`inventories/production/hosts.yml` là bản đồ máy và nhóm:

- `control_plane`: chứa `server-tang2`.
- `workers`: chứa `server-tang4`.
- `load_balancers`: chứa `server-tang3`.
- `server` và `agent`: compatibility aliases (nhóm tương thích) mà collection upstream `k3s.orchestration` yêu cầu.
- `k3s_cluster`: nhóm gộp control plane và worker để chạy các kiểm tra chung.

Các LAN IP được ghi rõ trong inventory vì hiện tại DHCP cấp địa chỉ. Khi lease thay đổi, phải cập nhật inventory rồi chạy lại Ansible để kubeconfig và HAProxy nhận endpoint mới.

Biến được chia theo phạm vi:

- `group_vars/all.yml`: giá trị dùng chung như K3s version, Pod/Service CIDR, kubeconfig và hostname test.
- `group_vars/control_plane.yml`: nội dung `server_config_yaml`, gồm node IP, Flannel interface, TLS SAN và control-plane taint.
- `group_vars/workers.yml`: nội dung `agent_config_yaml` và hai label để ServiceLB chỉ chạy trên pool `ingress`.
- `group_vars/load_balancers.yml`: Docker connection cục bộ, HAProxy image và backend IP lấy động từ inventory.
- `host_vars/server-tang*.yml`: SSH user và tên LAN interface riêng của từng máy.

Ví dụ, `api_endpoint` không hard-code lại IP tang3 ở nhiều chỗ mà được suy ra từ host đầu tiên trong nhóm `load_balancers`. Tương tự, backend API và ingress của HAProxy được suy ra từ nhóm `control_plane` và `workers`.

## Roles: mô tả HOW

### `preflight`

Role này fail-closed trước khi thay đổi cụm:

1. Xác minh `ansible_host` thật sự tồn tại trên host.
2. Xác minh đúng LAN interface trong `host_vars`.
3. Kiểm tra node K3s có ít nhất `1024 MiB` RAM.
4. Cảnh báo dependency vào DHCP lease.
5. Trên tang3, xác minh `power-grafana` và `power-prometheus` vẫn chạy.
6. Trên node K3s, tắt active swap, comment swap entry trong `/etc/fstab` và load `overlay`, `br_netfilter`.

Role không sửa APT/dpkg và không thực hiện filesystem recovery.

### `k3s_control_plane`

Đây là wrapper role (role bao ngoài) quanh `k3s.orchestration.k3s_server`:

1. Kiểm tra binary, version, config và trạng thái `k3s.service`.
2. Chỉ gọi upstream role khi có drift hoặc K3s chưa chạy.
3. Đọc node token với `no_log`, cung cấp token cho bước join worker.
4. Fetch kubeconfig gốc từ tang2 về tang3.
5. Thay endpoint `127.0.0.1:6443` bằng `192.168.30.45:6443`.
6. Chờ API đi qua HAProxy.

Wrapper giữ logic đặc thù của homelab trong repo, còn logic cài K3s phức tạp được giao cho collection upstream đã pin version. Repo không fork hoặc copy code cài đặt upstream, đồng thời không gọi role cài distro prerequisites vì tang2 có trạng thái APT không ổn định nhưng K3s vẫn có thể hoạt động.

### `k3s_worker`

Role này kiểm tra binary, version, config, systemd unit, endpoint join và trạng thái `k3s-agent.service`. Khi cần hội tụ, nó gọi `k3s.orchestration.k3s_agent`. Sau đó role:

- xác minh service `k3s-agent` active;
- đợi node tang4 ở trạng thái `Ready`;
- xác minh hai ServiceLB labels `enablelb=true` và `lbpool=ingress`.

Worker join API qua `https://192.168.30.45:6443`; HAProxy chuyển kết nối tới tang2.

### `haproxy`

Role được tách theo chuẩn Ansible:

- `defaults/main.yml`: giá trị mặc định về container, config path, ports và cách giữ ingress listener hiện hữu.
- `tasks/main.yml`: tạo thư mục, kiểm tra/pull image khi thiếu, render template, kiểm tra container và chờ listener.
- `templates/haproxy.cfg.j2`: định nghĩa frontend/backend cho `6443`, `80`, `443` và stats cục bộ.
- `handlers/main.yml`: chỉ xóa/tạo lại container khi template thay đổi hoặc container không chạy.

Trước khi ghi config, module `template` dùng `haproxy -c` trong image đã pin để validate. Tính idempotency (chạy lặp không tạo thay đổi khi trạng thái đã đúng) đến từ việc chỉ pull image khi thiếu, chỉ notify handler khi config đổi, và không recreate container ở lần chạy không có drift.

Trong phase đầu của `site.yml`, role chỉ cần API listener. Nếu ingress listener đã tồn tại, `haproxy_preserve_existing_ingress` giữ `80/443` để một lần chạy idempotent không gây gián đoạn application traffic.

### `k3s_addons`

Role này chỉ quản lý cluster-level configuration:

- copy `HelmChartConfig` cho Traefik;
- gắn Traefik Service với ServiceLB pool `ingress`;
- đợi Traefik rollout;
- xác minh Traefik và ServiceLB Pod chỉ chạy trên tang4.

Role không triển khai demo application. Addon là thành phần nền tảng của cluster; Nginx là user workload và có lifecycle riêng.

### `validation`

Role chia kiểm tra theo host:

- Trên tang2: hai node `Ready`, Traefik/ServiceLB/smoke Pod ở đúng tang4, Docker workloads `code-frontend-1` và `code-backend-1` còn chạy.
- Trên tang3: listeners `6443/80/443`, API trả `pong`, smoke app chạy qua HTTP và HTTPS, các container `k3s-haproxy`, `power-grafana`, `power-prometheus` còn chạy.

## Playbooks: xác định WHAT và WHEN

Role là đơn vị tái sử dụng mô tả cách đạt một trạng thái. Playbook chọn host, gọi role/task và sắp thứ tự thực thi.

| Playbook | Mục đích |
| --- | --- |
| `playbooks/site.yml` | Entry point đầy đủ: dựng/reconcile nền tảng, smoke test và validation |
| `playbooks/preflight.yml` | Chỉ chạy baseline và guard rails |
| `playbooks/cluster.yml` | Chỉ reconcile K3s control plane và worker |
| `playbooks/addons.yml` | Chỉ reconcile Traefik/ServiceLB placement |
| `playbooks/smoke-test.yml` | Triển khai smoke workload và test ingress HTTP |
| `playbooks/validate.yml` | Chạy toàn bộ kiểm tra sau triển khai |
| `playbooks/app.yml` | Triển khai và kiểm tra demo Nginx riêng biệt |

Luồng của `site.yml`:

```text
Preflight
   -> HAProxy API :6443
   -> K3s control plane
   -> K3s worker
   -> Traefik / ServiceLB addons
   -> HAProxy ingress :80/:443
   -> Smoke workload
   -> Validation
```

`site.yml` không deploy demo application trong `manifests/app/`. Muốn chạy demo Nginx, phải gọi độc lập `playbooks/app.yml`. Smoke workload vẫn nằm trong `site.yml` vì nó là phép thử kỹ thuật cho đường ingress của cluster.

## Manifests: mô tả WHAT RUNS trong Kubernetes

### Cluster configuration và smoke workload

- `manifests/traefik/helmchartconfig.yml`: thêm label `lbpool=ingress` vào Traefik LoadBalancer Service để ServiceLB chọn tang4.
- `manifests/smoke-test/nginx.yml`: một workload nhỏ trong namespace `lab-demo`, trả chuỗi `K3s homelab smoke test`. K3s static manifest controller tự apply file khi playbook copy nó vào `/var/lib/rancher/k3s/server/manifests/`.

### Demo application trong `manifests/app/`

Các resource được tách file và apply theo dependency order:

1. `namespace.yml`: tạo namespace `public-app`, là ranh giới logic cho demo.
2. `deployment.yml`: tạo Deployment `nginx`, hai replicas, pull `nginx:alpine`, có readiness probe và `nodeSelector` buộc cả hai Pod chạy trên `server-tang4`.
3. `service.yml`: tạo Service loại `ClusterIP`, chọn Pod có label `app=nginx` và cung cấp cổng ổn định bên trong cluster.
4. `ingress.yml`: khai báo hostname `nginx.onprem.site`, path `/`, `ingressClassName: traefik`, rồi trỏ tới Service `nginx`.

Service không cần `NodePort`: HAProxy không đi thẳng tới Service của app. HAProxy đi tới cổng `80/443` mà ServiceLB mở cho Traefik trên tang4; Traefik sau đó truy cập ClusterIP Service qua mạng Kubernetes.

## Traefik, ServiceLB và HAProxy phối hợp thế nào

- HAProxy là external TCP entry point. Nó chỉ biết tang2 cho API và tang4 cho application ports.
- ServiceLB là load balancer implementation tích hợp của K3s. Với Traefik Service loại `LoadBalancer`, ServiceLB chiếm cổng `80/443` trên tang4.
- Traefik là Ingress Controller: nó watch các Kubernetes Ingress, đọc hostname/path và tạo routing động tới đúng Service.
- Service cung cấp virtual IP và cân bằng request tới các Pod matching selector.

Vì vậy HAProxy không biết Nginx Pod nằm ở đâu, không biết namespace nào và không cần một backend mới cho mỗi app. Mười hostname khác nhau vẫn có thể dùng cùng backend tang4; Traefik phân loại request dựa trên HTTP `Host` hoặc TLS routing tương ứng.

## Luồng truy cập application

Lệnh test không cần DNS:

```bash
curl -H 'Host: nginx.onprem.site' http://192.168.30.45/
```

Request đi qua 11 bước:

1. `curl` kết nối tới LAN IP `192.168.30.45`, port `80`.
2. Header `Host: nginx.onprem.site` được gửi trong HTTP request.
3. HAProxy frontend `application_http` nhận TCP connection trên tang3.
4. HAProxy backend chuyển connection tới `192.168.30.35:80`.
5. ServiceLB trên tang4 nhận traffic ở host port `80`.
6. ServiceLB đưa traffic vào Traefik LoadBalancer Service.
7. Traefik Ingress Controller nhận request.
8. Traefik so khớp hostname và path với Ingress `public-app/nginx`.
9. Ingress route trỏ tới ClusterIP Service `public-app/nginx` port `80`.
10. Service selector `app=nginx` chọn một trong hai Pod Ready.
11. Nginx container trả HTML về client theo đường kết nối ngược lại.

```text
Client / curl
  |  Host: nginx.onprem.site
  v
192.168.30.45:80 (HAProxy tang3)
  |
  v
192.168.30.35:80 (ServiceLB tang4)
  |
  v
Traefik
  |  match Ingress host/path
  v
ClusterIP Service public-app/nginx
  |  selector app=nginx
  +-------------------+
  |                   |
  v                   v
nginx Pod 1        nginx Pod 2
```

Muốn dùng browser Windows, mở PowerShell hoặc Notepad bằng quyền Administrator và thêm vào `C:\Windows\System32\drivers\etc\hosts`:

```text
192.168.30.45 nginx.onprem.site
```

Sau đó mở `http://nginx.onprem.site`. Hosts entry chỉ giải quyết name resolution; máy Windows vẫn phải có route tới LAN `192.168.30.0/24`.

## Luồng Kubernetes API

```text
kubectl trên tang3
  |  kubeconfig server: https://192.168.30.45:6443
  v
HAProxy tang3 :6443
  |
  v
K3s API server tang2 :6443
  |
  v
SQLite datastore trên tang2
```

Endpoint ổn định không đồng nghĩa với HA. Nếu tang2 hỏng, HAProxy không có control plane thứ hai để chuyển sang. Nếu tang3 hỏng, endpoint bên ngoài mất dù K3s server trên tang2 có thể vẫn chạy.

## Phân lớp tài nguyên

| Lớp | Vị trí | Ý nghĩa |
| --- | --- | --- |
| Infrastructure automation | `inventories/`, `roles/`, `playbooks/` | Tự động hóa host, K3s, HAProxy và kiểm tra |
| Cluster configuration | `manifests/traefik/` | Cấu hình cách ingress platform được expose |
| Smoke workload | `manifests/smoke-test/` | Kiểm tra đường đi hệ thống sau khi dựng cluster |
| Demo workload | `manifests/app/` | Ứng dụng học tập có lifecycle riêng |

Smoke test trả một chuỗi được kiểm soát và được `site.yml` dùng làm health proof. Demo app dùng public image, hai replicas và các manifest tách riêng để học resource model. Demo app hỏng không đồng nghĩa cluster ingress hỏng; smoke test giúp tách hai loại lỗi này.

## Ứng dụng microservices `app1` đang chạy trên cluster

Ngoài smoke workload và demo Nginx do repository này quản lý, cluster đang chạy ứng dụng microservices từ repository độc lập [`Kien-devops/app1`](https://github.com/Kien-devops/app1):

| Thành phần | Trạng thái triển khai |
| --- | --- |
| Namespace | `microservices-demo` |
| Frontend | 2 replicas, NGINX unprivileged |
| Backend | `auth-service`, `user-service`, `product-service` |
| Worker | Toàn bộ application Pod chạy trên `server-tang4` |
| Registry | `ghcr.io/kien-devops/app1/*:<full-git-sha>` |
| Ingress | `app1.onprem.site`, class `traefik` |
| Public URL | `https://app1.onprem.site` qua Cloudflare Tunnel |
| CD runner | `server-tang3-k3s-deploy` trên `server-tang3` |

Luồng traffic:

```text
Internet
  -> Cloudflare Tunnel
  -> HAProxy server-tang3 :80
  -> ServiceLB / Traefik server-tang4
  -> Ingress app1.onprem.site
  -> frontend hoặc API ClusterIP Service
  -> application Pods
```

`k3s-onprem` sở hữu vòng đời platform: node, K3s, HAProxy, Traefik và ServiceLB. Repository `app1` sở hữu source code, image, namespace-scoped workload manifests và CI/CD release. Vì vậy `playbooks/site.yml` không deploy hoặc rollback `app1`.

Xem mô tả workload, routing, security boundary và cách kiểm tra tại [docs/app1-microservices.md](docs/app1-microservices.md). Hướng dẫn tổng quát để tích hợp thêm application nằm tại [docs/application-cicd.md](docs/application-cicd.md).

## Điều kiện trước khi triển khai

- Ba máy truy cập nhau qua LAN `192.168.30.0/24`.
- Chạy Ansible trên `server-tang3` bằng user `monitor`.
- SSH key `/home/monitor/.ssh/k3s_ansible_ed25519` truy cập được tang2 và tang4 không cần password.
- Remote users có non-interactive sudo cho managed tasks.
- Docker đang chạy trên tang3.
- Có Python 3, internet để cài collection/pull image và các port `6443`, `80`, `443` trên tang3 chưa bị service khác chiếm.

## Triển khai theo thứ tự

Chạy trên `server-tang3`:

```bash
cd /home/monitor/k3s-onprem
python3 -m venv /home/monitor/.venvs/k3s-ansible
source /home/monitor/.venvs/k3s-ansible/bin/activate
python -m pip install -r requirements.txt
ansible-galaxy collection install -r collections/requirements.yml

ansible-inventory --graph
ansible all -m ping
ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/site.yml
ansible-playbook playbooks/validate.yml
```

Sau khi nền tảng đã khỏe, triển khai demo app riêng:

```bash
ansible-playbook playbooks/app.yml
kubectl --kubeconfig /home/monitor/.kube/config get pods -n public-app -o wide
kubectl --kubeconfig /home/monitor/.kube/config get service -n public-app
kubectl --kubeconfig /home/monitor/.kube/config get ingress -n public-app
curl -H 'Host: nginx.onprem.site' http://192.168.30.45/
```

Chi tiết từng bước nằm trong [docs/deployment.md](docs/deployment.md).

## Giới hạn và nguyên tắc vận hành

- Tang2 có SSD/ext4 và APT/dpkg không khỏe; repo không tự repair chúng.
- Không chạy `fsck` trên mounted filesystem và không recovery theo cách có nguy cơ phá Docker workloads hiện hữu.
- Nếu K3s thực sự thất bại vì I/O error, dừng và thay disk.
- Ba LAN IP hiện là DHCP lease. Khi IP đổi, cập nhật inventory và chạy lại Ansible.
- Chỉ có một control plane, một HAProxy endpoint và một worker; mỗi máy đều là single point of failure tương ứng.
- SQLite phù hợp lab một control plane, không phải datastore HA.
- `nginx:alpine` phù hợp demo; production nên pin version/digest bất biến.
- Admin kubeconfig `/home/monitor/.kube/config` là credential đặc quyền và không được commit vào Git.

Đọc thêm:

- [Kiến trúc và luồng hệ thống](docs/architecture.md)
- [Quy trình triển khai](docs/deployment.md)
- [Troubleshooting theo từng lớp](docs/troubleshooting.md)
- [Ứng dụng microservices app1 đang chạy](docs/app1-microservices.md)
- [Mẫu tích hợp application CI/CD](docs/application-cicd.md)
