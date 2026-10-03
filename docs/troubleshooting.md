# Troubleshooting

## Ansible cannot reach a host

```bash
ansible all -m ping
ansible-inventory --graph
```

Confirm the current DHCP address, SSH key, user, and LAN interface. Tailscale
may be used to administer a host, but Kubernetes inventory must continue to use
LAN addresses.

## kubectl cannot reach the API

Check each hop in order:

```bash
ss -lnt | grep 6443                     # tang3
docker logs --tail 100 k3s-haproxy      # tang3
curl -k https://192.168.30.45:6443/ping # expected: pong
systemctl status k3s                    # tang2
```

The kubeconfig server must be `https://192.168.30.45:6443`.

## An application hostname does not respond

```bash
kubectl get ingress -A
kubectl get svc -A
kubectl get pods -A -o wide
curl -v -H 'Host: nginx.apps.k3s.home.arpa' http://192.168.30.45/
```

Then verify the layers in order: HAProxy listener, ServiceLB Pod, Traefik Pod,
Ingress hostname, Service selector, and application Pod readiness.

A hosts-file entry does not create a network route. If name resolution returns
`192.168.30.45` but TCP fails, the client must join the LAN or obtain a valid
route to `192.168.30.0/24`.

## Traefik or ServiceLB is on the wrong node

```bash
kubectl -n kube-system get pods -o wide
kubectl get node server-tang4 --show-labels
kubectl -n kube-system get helmchartconfig traefik -o yaml
```

Tang4 must have both `svccontroller.k3s.cattle.io` labels and the Traefik
Service must use the `ingress` pool.

## Public image pull fails

```bash
kubectl -n public-app describe pod
kubectl -n public-app get events --sort-by=.lastTimestamp
```

Check DNS, internet access, the registry tag, and anonymous registry rate
limits from tang4.

## Tang2 APT or filesystem errors

APT/dpkg is already known to be unhealthy because of physical disk/read errors.
Do not run automated APT repair, `fsck` on the mounted filesystem, or storage
recovery through this repository. If K3s itself fails with I/O errors, stop the
deployment and replace the disk while protecting the existing Docker workloads.
