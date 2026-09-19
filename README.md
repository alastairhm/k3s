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
