# Deployment

## Controller setup

Run from `server-tang3`:

```bash
cd /home/monitor/k3s-onprem
python3 -m venv /home/monitor/.venvs/k3s-ansible
source /home/monitor/.venvs/k3s-ansible/bin/activate
python -m pip install -r requirements.txt
ansible-galaxy collection install -r collections/requirements.yml
```

The SSH identity configured in inventory must reach tang2 and tang4, and the
remote users need non-interactive sudo. Keep credentials and kubeconfig outside
Git.

## Inspect before convergence

```bash
ansible-inventory --graph
ansible-playbook playbooks/site.yml --syntax-check
ansible all -m ping
```

The inventory graph contains repository-facing groups plus `server` and
`agent` aliases required by the upstream K3s roles.

## Converge the lab

```bash
ansible-playbook playbooks/site.yml
```

The lifecycle is ordered so the API endpoint exists before K3s configuration,
the worker joins before Traefik placement is asserted, and validation runs only
after ingress and the smoke application are available.

On an already-running lab, HAProxy retains existing application listeners
during the early API phase. This prevents an idempotent run from temporarily
removing ports 80 and 443.

## Validate independently

```bash
ansible-playbook playbooks/validate.yml
```

The validation role checks both Kubernetes state and external traffic. It also
asserts that the pre-existing Docker and monitoring containers are still
running.

## Deploy the public image example

```bash
ansible-playbook playbooks/public-app.yml
kubectl --kubeconfig /home/monitor/.kube/config get pods -n public-app -o wide
curl -H 'Host: nginx.apps.k3s.home.arpa' http://192.168.30.45/
```

The example remains separate from cluster addons because it represents a user
workload rather than cluster infrastructure.

## DHCP address changes

If a lease changes, update the corresponding `ansible_host` in
`inventories/production/hosts.yml`. Host-specific interface names remain under
`host_vars`. HAProxy backend addresses and the API endpoint are derived from
inventory, so a convergence run regenerates them.
