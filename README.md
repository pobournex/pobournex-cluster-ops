# pobournex-cluster-ops

GitOps repository for my homelab Kubernetes cluster. [Argo CD](https://argo-cd.readthedocs.io/) watches this repo and automatically syncs each application's manifests to the cluster.

## Cluster

- 3-node K3s cluster (1 control plane, 2 workers) on Proxmox VE
- MetalLB for LoadBalancer IPs and ingress-nginx for HTTP routing (installed separately from this repo)
- Argo CD for continuous delivery, with auto-sync enabled

## Applications

| App | Path | Namespace |
|-----|------|-----------|
| Miniflux (RSS reader, with PostgreSQL) | `apps/miniflux` | `miniflux` |
| Podinfo (demo app) | `apps/podinfo` | `podinfo` |
| Stirling-PDF | `apps/stirling-pdf` | `stirling-pdf` |
| Uptime Kuma | `apps/uptime-kuma` | `uptime-kuma` |

## How it works

1. Each application has its own folder under `apps/` containing its Kubernetes manifests.
2. An Argo CD Application points at that folder on the `main` branch and deploys it to its own namespace.
3. Changes are made by committing to Git. Argo CD detects the new commit and syncs it to the cluster.

## Bootstrap (rebuild from scratch)

Rough order to recreate the cluster from an empty set of nodes:

1. **K3s:** install a server on the control-plane node and join the two workers with the node token. If you use MetalLB, disable K3s's bundled service load balancer (`--disable servicelb`), and `--disable traefik` if you use ingress-nginx instead.
2. **MetalLB:** install it and create an `IPAddressPool` and `L2Advertisement` for a free range on your LAN.
3. **ingress-nginx:** install it with its official manifests or Helm chart.
4. **Argo CD:**
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
5. **Secrets:** create the Kubernetes Secrets the manifests expect (next section). These are never committed.
6. **Applications:** create one Argo CD Application per folder under `apps/`. Example for Miniflux:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: miniflux
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/pobournex/pobournex-cluster-ops.git
       path: apps/miniflux
     destination:
       server: https://kubernetes.default.svc
       namespace: miniflux
     syncPolicy:
       automated: {}
       syncOptions:
         - CreateNamespace=true
   ```
   This mirrors the live Applications: auto-sync on, tracking the repo's default branch (`targetRevision` unset), and no `prune` or `selfHeal`, so resources deleted from Git are not removed from the cluster automatically. Repeat for `podinfo`, `stirling-pdf`, and `uptime-kuma`, changing `name`, `path`, and `namespace`. `CreateNamespace=true` matters for `podinfo`, whose manifest does not define its own Namespace.

Argo CD then syncs everything from this repo.

## Secrets

No secrets are stored in this repository. Credentials live in Kubernetes Secrets created directly in the cluster and are referenced from the manifests with `secretKeyRef`.

Miniflux expects a Secret named `miniflux-secrets` in the `miniflux` namespace with two keys:

| Key | Used for |
|-----|----------|
| `db-password` | PostgreSQL password, and Miniflux's database connection |
| `admin-password` | Miniflux admin account password |

Create it before the first sync:

```bash
kubectl create namespace miniflux
kubectl -n miniflux create secret generic miniflux-secrets \
  --from-literal=db-password='<choose-a-password>' \
  --from-literal=admin-password='<choose-a-password>'
```

If the Secret is missing, the Miniflux pods stay in `CreateContainerConfigError` until it exists.

## Storage

The Miniflux PostgreSQL database uses a PersistentVolumeClaim (`local-path`), so data survives pod restarts.

## Stack

Kubernetes (K3s) · Argo CD · MetalLB · ingress-nginx · Proxmox VE · YAML manifests

## History

The cluster and these Argo CD applications have been running since February 2026. This repository was republished in October 2026 with a fresh Git history, after credentials that were in early commits were rotated and moved into Kubernetes Secrets.

## Screenshots

Argo CD showing all four applications Synced and Healthy:

![Argo CD applications](docs/argocd-apps.png)

The detailed views below use Miniflux as the example; the other apps follow the same pattern.

Resource tree for the Miniflux app, including the PostgreSQL PersistentVolumeClaim:

![Miniflux resource tree in Argo CD](docs/miniflux-resource-tree.png)

The 3-node K3s cluster and the running Miniflux pods:

![Cluster nodes and Miniflux pods](docs/cluster-nodes-pods.png)
