# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Flux GitOps repository for a k3s homelab cluster. Pushing to `main` on GitHub causes Flux to reconcile the cluster state within ~1 minute (poll interval). There is no build or test step — changes are applied by committing valid YAML.

## Cluster access

`kubectl` and `flux` run directly on this machine (no SSH needed). The cluster's Traefik ingress LoadBalancer IP is `10.0.100.50`. MetalLB hands out IPs from the `10.0.100.50–10.0.100.60` pool.

Useful commands:
```bash
kubectl get all -A                        # cluster-wide overview
flux get all -A                           # Flux reconciliation status
flux reconcile kustomization flux-system  # force re-sync from Git
flux logs --follow                        # watch Flux events
kubectl describe helmrelease <name> -n <ns>  # diagnose a failed HelmRelease
```

## Repo layout & Flux wiring

```
clusters/k3s-playground/
  kustomization.yaml       # root Kustomization — lists every top-level resource
  flux-system/             # Flux controllers + GitRepository + root Kustomization (DO NOT EDIT)
  metallb/                 # MetalLB HelmRelease (Flux Kustomization with healthCheck)
  metallb-config/          # IPAddressPool + L2Advertisement (depends on metallb)
  headlamp/                # Headlamp HelmRelease + Ingress
  adguard/                 # AdGuard Home Deployment, Services, Ingress (plain manifests, no Helm)
```

Flux reconciles `clusters/k3s-playground` as the root path. Adding a new app means:
1. Create a directory under `clusters/k3s-playground/<app>/`
2. Add its manifest(s) to `clusters/k3s-playground/kustomization.yaml` **or** create a Flux `Kustomization` CRD (like metallb does) if you need `dependsOn`, health checks, or a separate sync interval.

## Ingress pattern

All HTTP(S) ingress goes through Traefik (installed by k3s). Traefik's `websecure` entrypoint has `--entryPoints.websecure.http.tls=true` but no default certificate — it serves `CN=TRAEFIK DEFAULT CERT` (self-signed) unless a TLS secret is referenced in the Ingress. The standard annotations for dual HTTP+HTTPS are:

```yaml
annotations:
  traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
  traefik.ingress.kubernetes.io/router.tls: "true"
```

## DNS

`.lan` domains are resolved by AdGuard Home running in the cluster (LoadBalancer IP `10.0.100.53`). Adding a new `*.lan` hostname requires a DNS rewrite entry in AdGuard pointing to the Traefik LoadBalancer IP (`10.0.100.50`), in addition to the Ingress rule.

## Renovate

Renovate watches `clusters/**/*.yaml` for Helm chart versions and container image tags. `automerge: false` for all Kubernetes/Helm updates — PRs are created but must be merged manually.
