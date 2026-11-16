# k3s

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Kubernetes manifests for a single-node k3s homelab cluster, managed alongside Portainer
(the Portainer agent runs in-cluster; the Portainer server itself runs outside this repo's scope).

## Cluster

- Node: `optiplex3050` (single control-plane node), internal IP `192.168.3.43`, DuckDNS hostname
  `dell3050.duckdns.org`
- Ingress: Traefik (k3s default), exposed via the built-in ServiceLB at `192.168.3.43:80`/`:443`.
  Apps share the one DuckDNS hostname via path-based routing (see below), since DuckDNS's free
  tier gives one hostname rather than wildcard subdomains.
- Namespaces of interest:
  - `applications` — where experiment workloads in this repo live
  - `portainer` — Portainer agent

## Layout

```
manifests/
  applications/
    namespace.yaml       # the `applications` namespace
    whoami/               # example test app (traefik/whoami)
      deployment.yaml
      service.yaml        # NodePort 30080, for quick testing with no DNS setup
      ingress.yaml         # dell3050.duckdns.org/whoami
      middleware.yaml      # strips the /whoami prefix before forwarding
    homer/                 # homelab dashboard (github.com/bastienwirtz/homer)
      configmap.yaml       # config.yml — service links shown on the dashboard
      deployment.yaml
      service.yaml        # NodePort 30081
      ingress.yaml         # dell3050.duckdns.org/homer
      middleware.yaml      # strips the /homer prefix before forwarding
    telemetryiq/           # GT7 telemetry recorder (github.com/a1exus/TelemetryIQ)
      deployment.yaml      # hostPort 33740/udp for GT7 telemetry; locally built image
      configmap-patches.yaml  # patched compare.html and server.py, mounted over the image's copies
      pvc.yaml             # SQLite database (local-path)
      service.yaml         # NodePort 30082, LAN only (no auth, so no Ingress)
```

Each subdirectory under `manifests/applications/` is a self-contained app: Deployment + Service
(+ optional Ingress). Add new experiments the same way — one directory per app.

## Usage

Apply everything for an app:

```
kubectl apply -f manifests/applications/namespace.yaml -f manifests/applications/whoami/
```

Check status:

```
kubectl -n applications get pods,svc,ingress
```

Reach the whoami test app:

- NodePort (no setup required): `curl http://192.168.3.43:30080/`
- Ingress (via DuckDNS): `curl http://dell3050.duckdns.org/whoami/`

Reach the homer dashboard:

- NodePort (no setup required): open `http://192.168.3.43:30081/`
- Ingress (via DuckDNS): open `http://dell3050.duckdns.org/homer/`
- Edit `manifests/applications/homer/configmap.yaml` to add/change dashboard links, then
  `kubectl apply -f manifests/applications/homer/configmap.yaml && kubectl -n applications rollout restart deployment/homer`

New apps added under `manifests/applications/` follow the same path-based pattern: give the
Ingress `host: dell3050.duckdns.org` with a unique `path: /<app>`, and add a matching
`Middleware` with `stripPrefix.prefixes: ["/<app>"]` referenced via the
`traefik.ingress.kubernetes.io/router.middlewares: applications-<app>-stripprefix@kubernetescrd`
annotation (see whoami/ or homer/ for a working example).

## TelemetryIQ (GT7 telemetry)

There is no published image, so build it from upstream and import it into k3s's containerd
(the import needs sudo). The tag is the upstream commit and must match `deployment.yaml`:

```
git clone https://github.com/a1exus/TelemetryIQ.git && cd TelemetryIQ && git checkout 89ae54b
docker build -t telemetryiq:89ae54b .
docker save telemetryiq:89ae54b | sudo k3s ctr images import -
```

Set `PS_IP` in `deployment.yaml` to the PlayStation's IP (ideally a DHCP reservation). Auto-discovery
uses a LAN broadcast that can't leave the pod network. In GT7, turn on UDP telemetry under
Settings → Options. Then:

```
kubectl apply -f manifests/applications/namespace.yaml -f manifests/applications/telemetryiq/
kubectl -n applications logs -f deploy/telemetryiq    # look for [gt-telem] event: lines
```

`configmap-patches.yaml` holds patched copies of two upstream files (at 89ae54b), mounted over
the image's copies: `compare.html` (a ReferenceError left `/compare` empty) and `server.py` (one
stalled WebSocket client, such as a sleeping phone, froze the live HUD for everyone). If you
rebuild from a newer upstream commit, check whether these are fixed there and drop the mounts if so.

Live HUD: `http://192.168.3.43:30082/`. Lap comparison: `http://192.168.3.43:30082/compare`.
Laps are recorded to `/data/telemetry.db` on the PVC. Keep the replica count at 1 because
of the hostPort and the single SQLite file.

Remove an app:

```
kubectl delete -f manifests/applications/whoami/
```

## Using these manifests with Portainer

Portainer can manage the same resources two ways:

1. **Kubernetes view**: Portainer's Kubernetes UI can browse/edit the resources applied via
   `kubectl` above directly (they show up under the `applications` namespace).
2. **Application deployment**: In Portainer, use "Create from manifest" and paste the contents
   of a directory's YAML files to deploy without a local `kubectl` — useful for quick UI-driven
   experiments that you then export back into this repo to keep as source of truth.

## kubeconfig note

k3s's `kubectl` defaults to reading `/etc/rancher/k3s/k3s.yaml` (root-only) instead of the
standard `~/.kube/config`, unless `KUBECONFIG` is set. This host has
`export KUBECONFIG="$HOME/.kube/config"` in `~/.bashrc`, with a copy of the cluster's kubeconfig
at that path owned by the local user.
