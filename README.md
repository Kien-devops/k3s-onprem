# K3s on-prem homelab

Disposable single-control-plane K3s lab deployed with Ansible.

```text
LAN clients / admin
        |
        v
server-tang3 (192.168.30.45)
HAProxy :6443, :80, :443
       / \
      v   v
server-tang2                 server-tang4
192.168.30.44               192.168.30.35
K3s server / SQLite         K3s agent / workloads / ingress
```

This architecture provides stable API and ingress endpoints but does not
provide high availability.

## Current host inventory

| Host | LAN address | Role |
| --- | --- | --- |
| `server-tang2` | `192.168.30.44/24` | K3s server/control plane, SQLite |
| `server-tang3` | `192.168.30.45/24` | Ansible controller and HAProxy |
| `server-tang4` | `192.168.30.35/24` | K3s agent, Traefik, ServiceLB and application workloads |

## Important lab constraints

- The three LAN addresses are DHCP leases. If they change, update the inventory
  and HAProxy configuration before rerunning Ansible.
- `server-tang2` has known SSD read errors and ext4 metadata errors. The lab is
  intentionally allowed to continue while K3s remains operational.
- APT/dpkg on `server-tang2` is unhealthy because an existing `udev` file cannot
  be read from disk. The cluster play deliberately bypasses the collection's
  distro-package prerequisite role and does not attempt further APT repair.
- Existing Docker workloads on tang2 and the Grafana/Prometheus/power monitoring
  stack on tang3 are outside the cluster and must not be removed.
- Tailscale is used only for administration. K3s node and Flannel traffic use
  the `192.168.30.0/24` LAN.

## Pinned versions

- K3s: `v1.36.4+k3s1`
- k3s-ansible collection: `1.2.2`
- ansible-core: `2.21.4`
- HAProxy container: `haproxy:3.2.24-alpine3.24`

## Repository layout

```text
ansible.cfg
collections/requirements.yml
inventories/production/hosts.yml
manifests/
  smoke-test/nginx.yml
  traefik/helmchartconfig.yml
playbooks/
  preflight.yml
  baseline.yml
  haproxy-api.yml
  cluster.yml
  addons.yml
  haproxy-ingress.yml
  smoke-test.yml
  validate.yml
  site.yml
roles/haproxy/
requirements.txt
```

The HAProxy container starts its master as root only so it can bind host ports
`80` and `443`; the HAProxy worker then drops to the image's `haproxy` user and
group. No host network sysctl is changed.

## Run from server-tang3

```bash
cd /home/monitor/k3s-onprem
source /home/monitor/.venvs/k3s-ansible/bin/activate
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook playbooks/site.yml
```

Validation:

```bash
kubectl --kubeconfig /home/monitor/.kube/config get nodes -o wide
curl -H 'Host: demo.apps.k3s.home.arpa' http://192.168.30.45/
curl -k -H 'Host: demo.apps.k3s.home.arpa' https://192.168.30.45/
```

Local DNS is optional for the lab. To use the hostname directly, map
`demo.apps.k3s.home.arpa` to `192.168.30.45` in the client DNS or hosts file.
