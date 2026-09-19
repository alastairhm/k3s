# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Kubernetes manifests for a single-node k3s homelab cluster (node `optiplex3050`), managed
alongside a Portainer agent running in-cluster. See README.md for cluster details, layout, and
usage.

## Commands

There is no build/lint/test tooling — this repo is plain YAML. The relevant commands are:

- Apply an app: `kubectl apply -f manifests/applications/namespace.yaml -f manifests/applications/<app>/`
- Validate manifests without applying: `kubectl apply --dry-run=client -f manifests/applications/<app>/`
- Check status: `kubectl -n applications get pods,svc,ingress`
- Remove an app: `kubectl delete -f manifests/applications/<app>/`

`kubectl` requires `KUBECONFIG=$HOME/.kube/config` (already exported in `~/.bashrc` on this host)
because k3s's default kubectl otherwise tries to read the root-only `/etc/rancher/k3s/k3s.yaml`.

## Structure

- `manifests/applications/` — one directory per experiment/app, each with its own
  Deployment/Service/(optional) Ingress. Namespace `applications` is shared across them.
- New experiments should follow the same pattern: a new subdirectory under
  `manifests/applications/` rather than a flat pile of YAML files.
