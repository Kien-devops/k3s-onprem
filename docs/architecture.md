# Architecture

## Design intent

This is a disposable learning environment with one K3s control plane and one
worker. The architecture keeps the public-facing entry point separate from
Kubernetes and makes the traffic path visible without introducing production
HA components.

HAProxy on `server-tang3` has two responsibilities:

1. Forward `:6443` to the K3s API on `server-tang2`.
2. Forward `:80/:443` to the Traefik/ServiceLB entry point on `server-tang4`.

HAProxy does not inspect Kubernetes objects. Traefik watches Ingress resources
and maps application hostnames to Services and Pods. Consequently, adding an
Ingress does not require a new HAProxy backend.

## Component ownership

| Component | Managed by this repository | Notes |
| --- | --- | --- |
| K3s server and agent | Yes | Installation delegated to `k3s.orchestration` |
| HAProxy container/config | Yes | Configuration is validated before recreation |
| Traefik configuration | Yes | Bundled K3s chart, constrained to tang4 |
| ServiceLB placement | Yes | Node labels and service pool select tang4 |
| Demo application | Yes | `manifests/app/`, separate from cluster addons |
| Validation workload | Yes | `manifests/smoke-test/`, verifies ingress independently |
| Docker apps on tang2 | No | Validated but never removed or reconfigured |
| Prometheus/Grafana on tang3 | No | Validated but never removed or reconfigured |
| LAN DHCP and Tailscale | No | Existing external networking |

## Cluster join

The local `k3s_worker` role calls the upstream agent role. The upstream role
obtains the K3s node token from the first member of the compatibility `server`
group and joins the agent through `https://192.168.30.45:6443`. HAProxy then
forwards that connection to tang2.

## Scheduling and ingress placement

The control-plane node is tainted with
`node-role.kubernetes.io/control-plane=true:NoSchedule`. Tang4 has the labels:

```text
svccontroller.k3s.cattle.io/enablelb=true
svccontroller.k3s.cattle.io/lbpool=ingress
```

The Traefik LoadBalancer Service selects the `ingress` pool. This places its
ServiceLB Pod on tang4, where ports 80 and 443 are exposed. HAProxy always sends
application traffic to this node.

## Demo application flow

```text
Public Registry
      -> K3s/containerd pulls nginx:alpine
      -> Deployment
      -> Pods on server-tang4
      <- ClusterIP Service
      <- Ingress
      <- Traefik
      <- ServiceLB
      <- HAProxy on server-tang3
      <- Client
```

The demo is split across `manifests/app/namespace.yml`, `deployment.yml`,
`service.yml`, and `ingress.yml`. It is deployed by `playbooks/app.yml` and is
not part of `k3s_addons`.

No per-application HAProxy backend is required. HAProxy only forwards
`:80/:443` to tang4; Traefik performs hostname routing from Kubernetes Ingress
resources.

## Failure boundaries

HAProxy gives clients stable addresses, not redundancy. Failure of tang2 loses
the API/control plane; existing Pods may continue temporarily but cannot be
reconciled. Failure of tang3 loses external API and application entry points.
Failure of tang4 loses application workloads and ingress.

These boundaries are intentional for the lab and are not solved by this
repository.
