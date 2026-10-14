# Uptime Kuma on the local k3d cluster

This manifest runs the official Uptime Kuma 2 image as a single StatefulSet. Its
SQLite database and configuration live in a 5 GiB `local-path` PersistentVolumeClaim.
The pod requests 100 millicores and 256 MiB of memory, with limits of 500
millicores and 768 MiB. Kubernetes probes the HTTP server and restarts the pod
when it stops responding.

## Deploy

```sh
kubectl apply -f kubernetes/uptime-kuma.yaml
kubectl -n uptime-kuma rollout status statefulset/uptime-kuma
```

Open <http://uptime.192.168.1.3.nip.io:19080> from a device on the same LAN, then
complete Uptime Kuma's initial setup and create its administrator account. This
uses the existing OpenChoreo data-plane HTTP gateway and does not expose the
container port directly.

The route is plain HTTP because the current LAN gateway has no TLS listener. Use
it only on a trusted network; use a TLS-enabled private ingress or VPN before
accessing it over an untrusted network. The hostname embeds the cluster host's
current LAN address (`192.168.1.3`); update the `HTTPRoute` hostname if that
address changes. A DHCP reservation for the host keeps the address stable.

## Data and restart behavior

Do not delete the `uptime-kuma` PVC when updating or restarting the workload.
The `local-path` volume survives pod restarts and host power cycles as long as
the k3d node container and its data directory are retained. The k3d node and
server load balancer currently use Docker's `unless-stopped` restart policy, so
they start again when the Docker engine starts. Docker Desktop (or the Docker
engine) must itself start after login/boot.

This local volume is not a backup: deleting/recreating the k3d cluster or losing
the host disk can remove the database. Back up the PVC contents before cluster
rebuilds, and keep a copy outside the cluster for disaster recovery.

## Monitors

After creating the administrator account, add the HTTP(S), TCP, DNS, or other
monitors you need in the Kuma dashboard. Use the cluster's service DNS names for
in-cluster checks (for example, `service.namespace.svc.cluster.local`) and the
LAN hostnames for checks that should include the gateway. Configure notification
providers in the dashboard if you want alerts. The app is not preloaded with
monitor targets or notification credentials.
