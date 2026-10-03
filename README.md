# K3s On-Prem Homelab

An Ansible-automated K3s homelab running a single control plane and dedicated
worker behind an external HAProxy entry point.

HAProxy provides stable API and ingress endpoints. It does not make the
Kubernetes control plane highly available.

## Overview

This repository manages a small, disposable Kubernetes environment on three
existing Ubuntu hosts. It deliberately favors a clear traffic path and low
operational complexity over production high availability.

Ansible runs locally on `server-tang3`. Thin local roles describe this
repository's responsibilities while the pinned upstream
`k3s.orchestration` collection performs K3s server and agent installation.
The repository does not copy or fork upstream installation logic.

## Architecture

```text
                           LAN Clients / Admin
                                   |
                        HTTP / HTTPS / K8s API
                                   |
                                   v
                    +-----------------------------+
                    | server-tang3                |
                    | 192.168.30.45               |
                    |                             |
                    | Ansible Controller          |
                    | HAProxy                     |
                    | Prometheus / Grafana        |
                    +-------------+---------------+
                                  |
                  +---------------+---------------+
                  |                               |
             TCP 6443                        TCP 80/443
                  |                               |
                  v                               v
       +----------------------+       +----------------------+
       | server-tang2         |       | server-tang4         |
       | 192.168.30.44        |       | 192.168.30.35        |
       |                      |       |                      |
       | K3s Server           |       | K3s Agent            |
       | Control Plane        |       | Worker Node          |
       | SQLite               |       | Traefik              |
       | Existing Docker Apps |       | ServiceLB            |
       |                      |       | Application Pods     |
       +----------------------+       +----------------------+
                  \                         /
                   \_____ Flannel VXLAN ___/
```

The control plane is isolated from normal application scheduling with a taint.
Application manifests use `server-tang4`, leaving `server-tang2` focused on the
Kubernetes API and system services.

## Traffic Flow

### Kubernetes API

```text
kubectl
   |
   v
HAProxy 192.168.30.45:6443
   |
   v
K3s API server 192.168.30.44:6443
```

The kubeconfig points to the HAProxy address. This gives administrators one
stable API endpoint even though the lab still has only one control-plane node.

### Application traffic

```text
Client
   |
   v
HAProxy 192.168.30.45:80/:443
   |
   v
ServiceLB on 192.168.30.35
   |
   v
Traefik
   |
   v
Ingress -> Service -> Pod
```

HAProxy does not need one backend per application. It forwards ports `80` and
`443` to the Traefik entry point on the worker. Traefik then selects an Ingress
rule by hostname. For example, all of these names use the same HAProxy backend:

```text
app1.apps.k3s.home.arpa
app2.apps.k3s.home.arpa
grafana.apps.k3s.home.arpa
```

Adding an application normally requires Kubernetes resources, not an HAProxy
configuration change.

## Node Roles

| Inventory group | Host | LAN address | Responsibility |
| --- | --- | --- | --- |
| `control_plane` | `server-tang2` | `192.168.30.44` | K3s server, API, SQLite and system services |
| `load_balancers` | `server-tang3` | `192.168.30.45` | Ansible controller and HAProxy entry point |
| `workers` | `server-tang4` | `192.168.30.35` | K3s agent, Traefik, ServiceLB and applications |

The `server` and `agent` inventory groups are compatibility aliases required by
the upstream K3s collection. Repository code consistently uses
`control_plane`, `workers`, and `load_balancers`.

## Technology Stack

| Component | Version or mode |
| --- | --- |
| K3s | `v1.36.4+k3s1` |
| k3s-ansible | `1.2.2` |
| ansible-core | `2.21.4` |
| HAProxy | `haproxy:3.2.24-alpine3.24` |
| Container runtime | K3s-managed containerd |
| CNI | Flannel VXLAN |
| Datastore | SQLite on the single control plane |
| Ingress | Bundled Traefik |
| Service exposure | K3s ServiceLB restricted to the worker |

## Repository Structure

```text
.
|-- ansible.cfg
|-- collections/requirements.yml
|-- inventories/production/
|   |-- hosts.yml
|   |-- group_vars/
|   `-- host_vars/
|-- playbooks/
|   |-- site.yml
|   |-- preflight.yml
|   |-- cluster.yml
|   |-- addons.yml
|   |-- smoke-test.yml
|   |-- public-app.yml
|   `-- validate.yml
|-- roles/
|   |-- preflight/
|   |-- k3s_control_plane/
|   |-- k3s_worker/
|   |-- haproxy/
|   |-- k3s_addons/
|   `-- validation/
|-- manifests/
|   |-- traefik/
|   |-- smoke-test/
|   `-- public-app.yaml
`-- docs/
    |-- architecture.md
    |-- deployment.md
    `-- troubleshooting.md
```

Role responsibilities are intentionally narrow:

- `preflight`: validates discovered LAN identity, protects existing monitoring,
  disables swap on K3s hosts, and loads only the required kernel modules.
- `k3s_control_plane`: calls the upstream server role, fetches kubeconfig, and
  points the client configuration to HAProxy.
- `k3s_worker`: calls the upstream agent role and verifies readiness and
  ServiceLB labels.
- `haproxy`: validates configuration before a handler recreates the container.
- `k3s_addons`: manages Traefik and ServiceLB placement only.
- `validation`: verifies cluster health, placement, protected containers, and
  complete API/HTTP/HTTPS traffic paths.

## Prerequisites

- All hosts must be reachable over the `192.168.30.0/24` LAN.
- Run Ansible from `server-tang3` as `monitor`.
- `/home/monitor/.ssh/k3s_ansible_ed25519` must reach tang2 and tang4 without a
  password.
- The remote users need non-interactive sudo for the managed tasks.
- Docker must already be running on tang3 for the HAProxy container.
- Python 3, a virtual environment, and internet access to the configured public
  registries are required.
- LAN ports `6443`, `80`, and `443` on tang3 must not be owned by another
  process.

No healthy APT state is required on tang2. The wrapper deliberately calls the
upstream K3s roles without the collection's distro-package prerequisite role.

## Deployment

On `server-tang3`:

```bash
cd /home/monitor/k3s-onprem
source /home/monitor/.venvs/k3s-ansible/bin/activate
python -m pip install -r requirements.txt
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook playbooks/site.yml --syntax-check
ansible-inventory --graph
ansible-playbook playbooks/site.yml
```

`playbooks/site.yml` is the primary entry point and exposes the lifecycle in
execution order:

```text
Preflight
   -> HAProxy API endpoint
   -> K3s control plane
   -> K3s worker
   -> Cluster addons
   -> HAProxy application ingress
   -> Smoke test
   -> Validation
```

The playbook converges the current lab. It does not reset K3s, recreate SQLite,
repair APT, run filesystem recovery, or remove pre-existing containers.

See [docs/deployment.md](docs/deployment.md) for the detailed workflow.

## Validation

Run the complete validation independently:

```bash
ansible-playbook playbooks/validate.yml
```

Useful direct checks:

```bash
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
kubectl --kubeconfig /home/monitor/.kube/config get pods -A -o wide
curl -H 'Host: demo.apps.k3s.home.arpa' http://192.168.30.45/
curl -k -H 'Host: demo.apps.k3s.home.arpa' https://192.168.30.45/
```

Validation fails if either node is not Ready, Traefik or ServiceLB is misplaced,
HAProxy listeners are unavailable, the smoke request fails, or protected Docker
and monitoring containers are missing.

## Deploying Applications

A new public image follows this path:

```text
Public Registry
      -> containerd pulls image
      -> Deployment / Pod
      -> ClusterIP Service
      -> Ingress
      -> Traefik
      -> ServiceLB
      -> HAProxy
      -> Client
```

The repository includes `manifests/public-app.yaml`, which contains the three
resources an application normally needs:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: public-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeSelector:
        kubernetes.io/hostname: server-tang4
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: public-app
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  namespace: public-app
spec:
  ingressClassName: traefik
  rules:
    - host: nginx.apps.k3s.home.arpa
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
```

Deploy and test the maintained example:

```bash
ansible-playbook playbooks/public-app.yml
kubectl --kubeconfig /home/monitor/.kube/config get pods -n public-app -o wide
curl -H 'Host: nginx.apps.k3s.home.arpa' http://192.168.30.45/
```

`nginx:alpine` is intentionally simple for this lab. Production workloads
should pin an immutable digest or controlled version tag.

## Networking

- LAN: `192.168.30.0/24`
- Pod CIDR: `10.42.0.0/16`
- Service CIDR: `10.43.0.0/16`
- API endpoint: `192.168.30.45:6443`
- Application endpoints: `192.168.30.45:80` and `:443`
- Traefik ServiceLB address: `192.168.30.35`

K3s node and Flannel traffic use LAN addresses. Tailscale is only an
administrative access path and is not part of Kubernetes networking.

For local name resolution, point application hostnames to `192.168.30.45` in
LAN DNS or a client hosts file:

```text
192.168.30.45 nginx.apps.k3s.home.arpa
```

A hosts entry only performs name resolution; the client still needs a network
route to `192.168.30.0/24`.

## Operations

Common operator commands call Ansible directly so the execution behavior stays
visible:

```bash
ansible-playbook playbooks/site.yml --syntax-check
ansible-inventory --graph
ansible-playbook playbooks/site.yml
ansible-playbook playbooks/validate.yml
ansible-playbook playbooks/smoke-test.yml
ansible-playbook playbooks/public-app.yml
```

The admin kubeconfig is stored at `/home/monitor/.kube/config` on tang3 and
uses `192.168.30.45:6443`. Treat it as a cluster administrator credential.

This repository manages K3s, HAProxy, Traefik placement, and lab manifests. It
does not manage or remove the existing Docker applications on tang2 or the
Prometheus/Grafana stack on tang3; preflight and validation only protect them.

## Known Limitations

1. The SSD in `server-tang2` has known physical read and ext4 errors.
2. APT/dpkg on tang2 is unhealthy.
3. Both conditions are accepted because this is a disposable homelab.
4. Important or irreplaceable data must not depend on tang2.
5. All three LAN addresses are DHCP leases.
6. If a lease changes, inventory and generated HAProxy configuration may need
   updating.
7. There is only one Kubernetes control-plane node.
8. HAProxy is also a single endpoint.
9. This architecture is not intended to provide production high availability.

## Troubleshooting

Start with the smallest failing layer:

```bash
ansible all -m ping
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
docker logs --tail 100 k3s-haproxy
curl -H 'Host: demo.apps.k3s.home.arpa' http://192.168.30.45/
```

Do not use this repository to repair tang2 APT/dpkg or its filesystem. If K3s
fails with real I/O errors, stop and replace the failing disk rather than risk
the existing Docker workloads.

See [docs/troubleshooting.md](docs/troubleshooting.md) for component-specific
checks and [docs/architecture.md](docs/architecture.md) for ownership and
network decisions.
