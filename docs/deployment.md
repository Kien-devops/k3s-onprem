# Triển khai K3s Homelab

Tài liệu này mô tả quy trình triển khai từ `server-tang3`. Các bước được sắp theo dependency thực tế: xác minh host trước, tạo API endpoint trước khi cài/join K3s, cấu hình ingress sau khi worker Ready, rồi mới kiểm tra workload.

## 1. Phạm vi của deployment

`playbooks/site.yml` sẽ:

- kiểm tra LAN identity, tài nguyên tối thiểu và protected containers;
- cấu hình HAProxy endpoint;
- cài hoặc reconcile K3s server trên tang2;
- cài hoặc reconcile K3s agent trên tang4;
- cấu hình placement của Traefik/ServiceLB;
- deploy smoke workload;
- kiểm tra API, HTTP/HTTPS và protected workloads.

Playbook sẽ không:

- reset cluster hoặc xóa SQLite;
- sửa APT/dpkg trên tang2;
- chạy `fsck` hoặc filesystem recovery;
- xóa/reconfigure Docker applications trên tang2;
- xóa/reconfigure Prometheus/Grafana trên tang3;
- deploy demo application trong `manifests/app/`.

Demo app có lifecycle độc lập qua `playbooks/app.yml`.

## 2. Source of truth trước khi chạy

| Host | LAN IP trong inventory | SSH/connection | Vai trò |
| --- | --- | --- | --- |
| `server-tang2` | `192.168.30.44` | user `node` | K3s control plane |
| `server-tang3` | `192.168.30.45` | local, user `monitor` | Ansible controller và HAProxy |
| `server-tang4` | `192.168.30.35` | user `node` | K3s worker |

Các IP này là DHCP leases. Trước mỗi lần dựng lại sau khi router hoặc máy reboot, xác minh chúng vẫn đúng. Không thay bằng IP đoán.

K3s data path dùng LAN interface được khai báo trong `host_vars`:

```text
server-tang2 -> wlx347de4475af5
server-tang3 -> wlp2s0
server-tang4 -> wlx58044f3fedb4
```

## 3. Chuẩn bị Ansible controller

Chạy trên `server-tang3` bằng user `monitor`:

```bash
cd /home/monitor/k3s-onprem

python3 -m venv /home/monitor/.venvs/k3s-ansible
source /home/monitor/.venvs/k3s-ansible/bin/activate

python -m pip install -r requirements.txt
ansible-galaxy collection install -r collections/requirements.yml
```

Repo pin các dependency chính:

- `ansible-core==2.21.4`;
- `k3s.orchestration` lấy từ `k3s-io/k3s-ansible` tag/version `1.2.2`;
- các collections phụ trợ cũng được pin trong `collections/requirements.yml`.

SSH identity mặc định là:

```text
/home/monitor/.ssh/k3s_ansible_ed25519
```

Key phải truy cập tang2 và tang4 không cần nhập password. Các remote users cần non-interactive sudo cho managed tasks. Không commit private key, kubeconfig, token hoặc credential vào Git.

## 4. Kiểm tra inventory và kết nối

### 4.1. Xem inventory graph

```bash
ansible-inventory --graph
ansible-inventory --host server-tang2
ansible-inventory --host server-tang3
ansible-inventory --host server-tang4
```

Graph cần có các nhóm `control_plane`, `workers`, `load_balancers`, cùng compatibility groups `server`, `agent`, `k3s_cluster`.

### 4.2. Kiểm tra SSH/Ansible transport

```bash
ansible all -m ping
```

Kết quả mong đợi: cả ba host trả `pong`. Với tang3, `ansible_connection: local` khiến task chạy ngay trên controller; tang2/tang4 đi qua SSH.

Nếu host fail, dừng trước khi deploy và kiểm tra current DHCP IP, SSH user, key, known_hosts và LAN route.

### 4.3. Chạy preflight độc lập

```bash
ansible-playbook playbooks/preflight.yml
```

Preflight xác minh:

- inventory IP tồn tại trên đúng host;
- LAN interface tồn tại;
- node K3s có tối thiểu `1024 MiB` RAM;
- monitoring containers trên tang3 vẫn chạy;
- swap và kernel modules phù hợp mức tối thiểu cho K3s.

Thông báo DHCP là warning có chủ đích, không block homelab khi IP hiện tại vẫn đúng.

## 5. Kiểm tra cú pháp trước khi thay đổi

```bash
ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/app.yml --syntax-check
```

Syntax check không chứng minh remote host khỏe, nhưng bắt lỗi YAML/playbook trước khi hội tụ.

## 6. Dựng hoặc reconcile toàn bộ nền tảng

```bash
ansible-playbook playbooks/site.yml
```

### Phase 1 - Preflight

Role `preflight` chạy trên tất cả hosts. Nó fail sớm nếu inventory trỏ sai IP/interface, thiếu RAM hoặc monitoring hiện hữu không còn chạy.

### Phase 2 - HAProxy API

Role `haproxy` chạy trên tang3 với `haproxy_enable_ingress: false`. Nó tạo endpoint `192.168.30.45:6443` trước khi K3s server và agent phụ thuộc endpoint đó.

Nếu config cũ đã có frontend application, `haproxy_preserve_existing_ingress: true` giữ `80/443` trong lần reconcile để tránh gián đoạn app đang chạy.

### Phase 3 - K3s control plane

Role `k3s_control_plane` kiểm tra version/config/service trước. Chỉ khi có drift, role mới gọi `k3s.orchestration.k3s_server`.

Server config đặt:

- node name và node IP theo inventory;
- Flannel trên LAN interface;
- Pod CIDR `10.42.0.0/16`;
- Service CIDR `10.43.0.0/16`;
- TLS SAN `192.168.30.45`;
- secrets encryption;
- control-plane taint.

Sau khi server chạy, role fetch kubeconfig về `/home/monitor/.kube/config` và đổi API endpoint sang HAProxy.

### Phase 4 - K3s worker

Role `k3s_worker` gọi upstream agent role khi cần. Tang4 join qua:

```text
https://192.168.30.45:6443
```

Role đợi node `Ready`, rồi xác minh hai ServiceLB labels.

### Phase 5 - Cluster addons

Role `k3s_addons` copy `manifests/traefik/helmchartconfig.yml` vào K3s static manifests directory, đợi Traefik rollout và xác minh Traefik/ServiceLB chỉ chạy trên tang4.

Phase này cấu hình cluster platform, không deploy user application.

### Phase 6 - HAProxy ingress

Role `haproxy` chạy lại với `haproxy_enable_ingress: true` để đảm bảo:

```text
192.168.30.45:80  -> 192.168.30.35:80
192.168.30.45:443 -> 192.168.30.35:443
```

Template được validate bằng `haproxy -c` trước khi handler được phép recreate container.

### Phase 7 - Smoke test

`playbooks/smoke-test.yml` copy manifest vào:

```text
/var/lib/rancher/k3s/server/manifests/lab-smoke-test.yaml
```

K3s static manifest controller apply resource, rồi Ansible chờ Deployment rollout và gửi request qua HAProxy với hostname `demo.apps.k3s.home.arpa`.

### Phase 8 - Validation

`playbooks/validate.yml` được import cuối site run để kiểm tra node, Pod placement, API response, HTTP/HTTPS response và protected containers.

## 7. Xác minh độc lập sau deployment

Chạy validation lần nữa nếu cần tách riêng kết quả khỏi deployment:

```bash
ansible-playbook playbooks/validate.yml
```

Kiểm tra Kubernetes trực tiếp:

```bash
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
kubectl --kubeconfig /home/monitor/.kube/config get pods -A -o wide
kubectl --kubeconfig /home/monitor/.kube/config -n kube-system get svc traefik -o wide
```

Kết quả quan trọng:

- `server-tang2` và `server-tang4` đều `Ready`;
- Traefik và `svclb-traefik-*` nằm trên tang4;
- smoke Pod nằm trên tang4;
- kubeconfig dùng server `https://192.168.30.45:6443`.

Test data path ngoài cluster:

```bash
curl -k https://192.168.30.45:6443/ping
curl -H 'Host: demo.apps.k3s.home.arpa' http://192.168.30.45/
curl -k -H 'Host: demo.apps.k3s.home.arpa' https://192.168.30.45/
```

Kết quả mong đợi lần lượt là `pong` và nội dung `K3s homelab smoke test`.

## 8. Triển khai demo application

Chỉ chạy bước này sau khi `site.yml` và validation thành công:

```bash
ansible-playbook playbooks/app.yml
```

Playbook thực hiện:

1. tạo `/opt/k3s-lab/manifests/app` trên tang2;
2. copy bốn file từ `manifests/app/`;
3. apply theo thứ tự Namespace -> Deployment -> Service -> Ingress;
4. đợi cả hai Nginx replicas Ready;
5. xác minh cả hai Pod chạy trên `server-tang4`;
6. gửi request từ tang3 qua HAProxy và Traefik.

Kiểm tra resource:

```bash
kubectl --kubeconfig /home/monitor/.kube/config get namespace public-app
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get deployment,pods -o wide
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get service
kubectl --kubeconfig /home/monitor/.kube/config -n public-app get ingress
```

Test qua IP HAProxy, không cần DNS:

```bash
curl -H 'Host: nginx.onprem.site' http://192.168.30.45/
```

Response cần chứa:

```html
<h1>Welcome to nginx!</h1>
```

Deploy demo app không thay đổi HAProxy. HAProxy vẫn chỉ chuyển `80/443` tới tang4; Traefik đọc Ingress mới và route hostname tới Service.

## 9. Mở demo bằng browser Windows

Mở Notepad bằng quyền Administrator, sửa file:

```text
C:\Windows\System32\drivers\etc\hosts
```

Thêm dòng:

```text
192.168.30.45 nginx.onprem.site
```

Xác minh trong PowerShell:

```powershell
findstr /C:"nginx.onprem.site" "$env:SystemRoot\System32\drivers\etc\hosts"
ping -n 1 nginx.onprem.site
curl.exe -v http://nginx.onprem.site/
```

Sau đó mở:

```text
http://nginx.onprem.site
```

Nếu hostname resolve đúng nhưng kết nối timeout, vấn đề là route/firewall tới LAN, không phải hosts file.

## 10. Kiểm tra idempotency

Chạy lại cùng playbook:

```bash
ansible-playbook playbooks/site.yml
ansible-playbook playbooks/app.yml
```

Trong trạng thái không drift, kết quả mong đợi là `changed=0`. Đặc biệt HAProxy container không được restart nếu config và container state đã đúng.

Một task báo `changed` không tự động có nghĩa là lỗi idempotency; cần xem task nào thay đổi và desired state có thực sự drift hay không. Repo đã đặt `changed_when: false` cho các lệnh read-only và chỉ đánh dấu `kubectl apply` thay đổi khi output chứa `created` hoặc `configured`.

## 11. Chạy từng phần

```bash
# Chỉ baseline
ansible-playbook playbooks/preflight.yml

# Chỉ K3s server và worker
ansible-playbook playbooks/cluster.yml

# Chỉ Traefik/ServiceLB configuration
ansible-playbook playbooks/addons.yml

# Chỉ smoke workload
ansible-playbook playbooks/smoke-test.yml

# Chỉ validation
ansible-playbook playbooks/validate.yml

# Chỉ demo app
ansible-playbook playbooks/app.yml
```

Các playbook rời giả định dependency trước đó đã tồn tại. Ví dụ `addons.yml` không tự dựng control plane; `app.yml` không tự cài Traefik hoặc HAProxy.

## 12. Khi DHCP IP thay đổi

1. Xác định LAN IP thật trên từng máy.
2. Cập nhật `ansible_host` tương ứng trong `inventories/production/hosts.yml`.
3. Không thay `host_vars` interface nếu tên interface không đổi.
4. Kiểm tra inventory và connectivity.
5. Chạy lại `site.yml` để regenerate HAProxy config và kubeconfig.
6. Cập nhật LAN DNS/Windows hosts nếu IP tang3 đổi.

```bash
ansible-inventory --graph
ansible all -m ping
ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/site.yml
```

## 13. Điều kiện phải dừng

Dừng deployment thay vì cố repair khi:

- mất SSH hoặc LAN connectivity;
- K3s không thể install/start;
- filesystem I/O errors trực tiếp làm K3s thất bại;
- thay đổi tiếp theo có nguy cơ phá Docker workloads hiện hữu.

Không chạy `fsck` trên mounted filesystem, không reset cluster và không purge package chỉ để làm sạch một homelab đang hoạt động.
