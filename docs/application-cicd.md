# Triển khai Application lên K3s bằng CI/CD

Tài liệu này mô tả cách một repository ứng dụng độc lập build container image và tự động deploy lên cụm K3s homelab hiện tại.

Kiến trúc được thiết kế theo nguyên tắc:

- Repository `k3s-onprem` quản lý cluster, HAProxy, Traefik và ServiceLB.
- Repository ứng dụng quản lý source code, Dockerfile và Kubernetes workload của chính nó.
- CI chạy trên GitHub-hosted runner.
- CD chạy trên self-hosted runner trong mạng LAN.
- Application pipeline không được chạy `playbooks/site.yml` hoặc thay đổi HAProxy.

## 1. Vì sao cần self-hosted runner?

Kubernetes API của homelab chỉ được expose trong LAN:

```text
https://192.168.30.45:6443
```

GitHub-hosted runner trên Internet không có route trực tiếp tới địa chỉ này. Vì vậy nên chia pipeline thành hai phần:

```text
Developer push code
        |
        v
GitHub-hosted runner
        |
        | test -> build -> scan
        | push immutable image :<git-sha>
        v
GitHub Container Registry
        |
        v
Self-hosted runner trên server-tang3
        |
        | kubectl apply
        v
HAProxy 192.168.30.45:6443
        |
        v
K3s API server-tang2
        |
        v
Deployment -> Pods trên server-tang4
        |
        v
Service -> Ingress -> Traefik
        |
        v
http://app1.onprem.site
```

Tang3 chỉ thực hiện phần deploy nhẹ bằng `kubectl`. Docker build chạy trên GitHub-hosted runner để tránh cạnh tranh CPU, RAM và disk với HAProxy, Prometheus và Grafana.

> Chỉ nên cho private repository hoặc repository được kiểm soát sử dụng self-hosted runner. Workflow không tin cậy có thể đọc credential và chiếm quyền máy runner.

## 2. Phân chia trách nhiệm giữa hai repository

### Repository hạ tầng `k3s-onprem`

Quản lý:

- K3s server và worker;
- HAProxy;
- Traefik và ServiceLB;
- cluster-level configuration;
- namespace bootstrap và RBAC cho application deployer.

### Repository ứng dụng `my-app`

Quản lý:

- source code và test;
- Dockerfile;
- container image;
- Deployment;
- ClusterIP Service;
- Ingress;
- CI/CD workflow;
- rollout validation của ứng dụng.

App repository không được quản lý K3s installation, HAProxy container hoặc workload của ứng dụng khác.

## 3. Cấu trúc repository ứng dụng

```text
my-app/
|-- src/
|-- tests/
|-- Dockerfile
|-- k8s/
|   |-- deployment.yml
|   |-- service.yml
|   `-- ingress.yml
`-- .github/
    `-- workflows/
        `-- ci-cd.yml
```

Namespace và RBAC không nằm trong pipeline ứng dụng. Platform admin bootstrap chúng một lần trước khi bật CD.

## 4. Kubernetes manifests của ứng dụng

Ví dụ dưới đây giả định:

- namespace: `my-app`;
- Deployment: `my-app`;
- container name: `app`;
- container port: `8080`;
- health endpoint: `/health`;
- hostname: `app1.onprem.site`.

### `k8s/deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 2

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1

  selector:
    matchLabels:
      app: my-app

  template:
    metadata:
      labels:
        app: my-app
    spec:
      nodeSelector:
        kubernetes.io/hostname: server-tang4

      containers:
        - name: app
          # Pipeline thay image placeholder trước khi apply.
          image: example.invalid/my-app:placeholder

          ports:
            - name: http
              containerPort: 8080

          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5

          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 15
            periodSeconds: 10
```

`readinessProbe` quyết định khi nào Pod được Service nhận traffic. `livenessProbe` phát hiện container bị treo và cho phép kubelet restart container.

Nếu ứng dụng không chạy port `8080` hoặc không có `/health`, phải sửa manifest cho đúng với application thực tế.

### `k8s/service.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: my-app
spec:
  type: ClusterIP

  selector:
    app: my-app

  ports:
    - name: http
      port: 80
      targetPort: http
```

Không cần `NodePort`. Traffic bên ngoài đi vào Traefik trước, sau đó Traefik truy cập ClusterIP Service qua mạng Kubernetes.

### `k8s/ingress.yml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: my-app
spec:
  ingressClassName: traefik

  rules:
    - host: app1.onprem.site
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  name: http
```

HAProxy không cần thay đổi khi thêm application này. Data path vẫn là:

```text
Client
  -> 192.168.30.45:80/443
  -> HAProxy server-tang3
  -> 192.168.30.35:80/443
  -> ServiceLB
  -> Traefik
  -> Ingress app1.onprem.site
  -> ClusterIP Service my-app
  -> my-app Pods
```

## 5. Bootstrap namespace và RBAC

Platform admin thực hiện bước này một lần bằng admin kubeconfig. Không chạy bootstrap ở mỗi application deployment.

Tạo file `my-app-deployer-rbac.yml` trong repository hạ tầng hoặc một vị trí quản trị phù hợp:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: my-app
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deployer
  namespace: my-app
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

  - apiGroups: [""]
    resources: ["pods", "pods/log", "events"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer
  namespace: my-app
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: my-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ci-deployer
```

Apply bằng admin kubeconfig:

```bash
kubectl --kubeconfig /home/monitor/.kube/config \
  apply -f my-app-deployer-rbac.yml
```

Sau đó tạo kubeconfig riêng cho ServiceAccount và lưu trên tang3:

```text
/home/monitor/.kube/my-app-deployer.config
```

File này phải:

- trỏ tới `https://192.168.30.45:6443`;
- chỉ chứa credential của `ci-deployer`;
- có permission `0600`;
- không được commit vào Git;
- không dùng chung với admin kubeconfig.

Role trên không cho phép pipeline xóa resource. Nếu muốn pipeline hỗ trợ `kubectl delete`, phải bổ sung verb `delete` có chủ đích.

## 6. Cài self-hosted runner trên tang3

Trong repository GitHub của ứng dụng:

```text
Settings
  -> Actions
  -> Runners
  -> New self-hosted runner
```

Chọn Linux và architecture phù hợp, sau đó chạy các lệnh đăng ký do GitHub sinh ra trên tang3. Registration token có thời hạn nên không lưu nó vào repository.

Đặt label riêng cho runner:

```text
k3s-deploy
```

Runner cần:

- outbound HTTPS port `443` tới GitHub;
- `kubectl`;
- quyền đọc `/home/monitor/.kube/my-app-deployer.config`;
- route tới `192.168.30.45:6443`;
- không cần sudo để deploy application.

Không nên cho workflow từ public fork hoặc pull request không tin cậy chạy trên runner này.

## 7. GitHub Actions workflow

Tạo `.github/workflows/ci-cd.yml`:

```yaml
name: CI/CD to K3s Homelab

on:
  push:
    branches:
      - main

permissions:
  contents: read
  packages: write

concurrency:
  group: deploy-my-app-homelab
  cancel-in-progress: false

jobs:
  build:
    name: Test, build and push
    runs-on: ubuntu-latest

    outputs:
      image: ${{ steps.image.outputs.value }}

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      # Thay bằng lệnh test thực tế của ứng dụng.
      - name: Run tests
        run: |
          echo "Run application tests here"

      - name: Generate immutable image name
        id: image
        shell: bash
        run: |
          IMAGE="ghcr.io/${GITHUB_REPOSITORY,,}:${GITHUB_SHA}"
          echo "value=${IMAGE}" >> "${GITHUB_OUTPUT}"

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.image.outputs.value }}

  deploy:
    name: Deploy to K3s
    needs: build

    runs-on:
      - self-hosted
      - linux
      - x64
      - k3s-deploy

    environment: homelab

    permissions:
      contents: read

    env:
      KUBECONFIG: /home/monitor/.kube/my-app-deployer.config
      IMAGE: ${{ needs.build.outputs.image }}

    steps:
      - name: Checkout deployment manifests
        uses: actions/checkout@v4

      - name: Render and apply deployment image
        shell: bash
        run: |
          set -euo pipefail
          kubectl set image \
            -f k8s/deployment.yml \
            app="${IMAGE}" \
            --local \
            -o yaml |
          kubectl apply -f -

      - name: Apply service and ingress
        run: |
          kubectl apply -f k8s/service.yml
          kubectl apply -f k8s/ingress.yml

      - name: Wait for rollout
        run: |
          kubectl -n my-app rollout status \
            deployment/my-app \
            --timeout=180s

      - name: Verify pods run on tang4
        shell: bash
        run: |
          set -euo pipefail
          test "$(kubectl -n my-app get pods \
            -l app=my-app \
            -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' \
            | sort -u)" = "server-tang4"

      - name: Smoke test through HAProxy
        run: |
          curl --fail --show-error --silent \
            -H 'Host: app1.onprem.site' \
            http://192.168.30.45/health
```

### Pipeline thực hiện những gì?

```text
Source artifact
  -> source code từ Git commit

Build artifact
  -> immutable container image
  -> ghcr.io/<owner>/<repo>:<git-sha>

Deployment artifact
  -> Deployment + Service + Ingress manifests
  -> Deployment được render với image SHA cụ thể

Validation
  -> rollout status
  -> Pod placement
  -> HTTP smoke test qua HAProxy
```

Không dùng duy nhất tag `latest`. Git commit SHA giúp truy vết chính xác version đang chạy và rollback về image cũ.

## 8. Public và private container image

### Public image

Phù hợp nhất để bắt đầu trong homelab:

- tang4 pull trực tiếp từ GHCR;
- không cần `imagePullSecret`;
- không lưu registry credential trong cluster.

### Private image

Nếu GHCR package để private:

1. tạo registry credential chỉ có quyền đọc package;
2. lưu credential thành Kubernetes Secret trong namespace `my-app`;
3. thêm `imagePullSecrets` vào Pod spec;
4. không dùng workflow `GITHUB_TOKEN` làm credential lâu dài trong cluster.

## 9. DNS hoặc Windows hosts

Nếu chưa có DNS nội bộ, thêm trên máy Windows:

```text
192.168.30.45 app1.onprem.site
```

Kiểm tra:

```powershell
ping -n 1 app1.onprem.site
curl.exe http://app1.onprem.site/
```

Hosts entry chỉ giải quyết hostname. Client vẫn cần route tới LAN `192.168.30.0/24`.

## 10. Rolling deployment và rollback

Deployment sử dụng RollingUpdate:

```text
Pod version cũ đang phục vụ
        |
        v
Tạo Pod version mới
        |
        v
Readiness probe thành công
        |
        v
Service đưa traffic vào Pod mới
        |
        v
Pod cũ được terminate
```

Nếu rollout thất bại:

```bash
kubectl -n my-app rollout status deployment/my-app
kubectl -n my-app describe deployment my-app
kubectl -n my-app get pods -o wide
kubectl -n my-app get events --sort-by=.lastTimestamp
```

Rollback nhanh về ReplicaSet trước:

```bash
kubectl -n my-app rollout undo deployment/my-app
kubectl -n my-app rollout status deployment/my-app
```

Cách repeatable hơn là chạy lại pipeline với image SHA trước đó:

```text
ghcr.io/<owner>/<repo>:<previous-git-sha>
```

## 11. Security boundaries

- Chỉ deploy từ protected branch `main`.
- Nên dùng GitHub Environment `homelab` và required reviewer trước deploy.
- Không chạy untrusted pull request trên self-hosted runner.
- Runner chỉ giữ kubeconfig có quyền trong namespace của ứng dụng.
- Không dùng admin kubeconfig cho CI/CD.
- Không lưu token, kubeconfig hoặc registry password trong Git.
- Image phải được gắn immutable tag hoặc digest.
- Build job không chạy trên tang3.
- Pipeline app không được thay HAProxy hoặc cluster-level components.

## 12. Khi nào nên dùng GitOps?

Phương án trong tài liệu là push-based deployment:

```text
GitHub Actions -> kubectl -> cluster
```

Nó phù hợp với homelab hiện tại vì dễ hiểu và ít thành phần.

Khi có nhiều app hoặc nhiều environment, có thể chuyển sang GitOps:

```text
CI build image
  -> cập nhật deployment repository
  -> Argo CD phát hiện desired state mới
  -> Argo CD pull và reconcile cluster
```

GitOps giúp GitHub Actions không cần kubeconfig, nhưng bổ sung Argo CD, GitOps repository, reconciliation policy và secret management. Với một worker và một vài application, chưa cần thêm độ phức tạp này.

## 13. Tài liệu tham khảo chính thức

- [GitHub: Adding self-hosted runners](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/add-runners)
- [GitHub: Secure use reference](https://docs.github.com/en/actions/reference/security/secure-use)
- [GitHub: Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Kubernetes: kubectl rollout](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)
