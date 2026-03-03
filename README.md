# [5EAM-Agency] 

> Infrastructure and application deployments managed via [FluxCD](https://fluxcd.io/) using GitOps principles.

---

## Overview

This repo follows the [FluxCD monorepo pattern](https://fluxcd.io/flux/guides/repository-structure/) for managing Kubernetes clusters, shared infrastructure, and application deployments across multiple environments.

| Item | Detail |
|------|--------|
| **Flux Version** | `v2.x.x` |
| **Kubernetes** | `v1.xx` |
| **Environments** | `production` · `staging` · `dev` |
| **Container Registry** | `ghcr.io/[your-org]` |

---

## Repository Structure

```
fleet-infra/                          # Root of this repository
│
├── clusters/                         # Cluster-level Flux entrypoints
│   ├── production/
│   │   ├── flux-system/              # Flux bootstrap components (auto-generated)
│   │   │   ├── gotk-components.yaml
│   │   │   ├── gotk-sync.yaml
│   │   │   └── kustomization.yaml
│   │   ├── infrastructure.yaml       # Kustomization → ../infrastructure/production
│   │   └── apps.yaml                 # Kustomization → ../apps/production
│   ├── staging/
│   │   ├── flux-system/
│   │   ├── infrastructure.yaml
│   │   └── apps.yaml
│   └── dev/
│       ├── flux-system/
│       ├── infrastructure.yaml
│       └── apps.yaml
│
├── infrastructure/                   # Shared cluster infrastructure
│   ├── base/                         # Environment-agnostic base configs
│   │   ├── cert-manager/
│   │   │   ├── namespace.yaml
│   │   │   ├── helmrelease.yaml
│   │   │   └── kustomization.yaml
│   │   ├── ingress-nginx/
│   │   │   ├── namespace.yaml
│   │   │   ├── helmrelease.yaml
│   │   │   └── kustomization.yaml
│   │   ├── monitoring/
│   │   │   ├── namespace.yaml
│   │   │   ├── helmrelease.yaml
│   │   │   └── kustomization.yaml
│   │   └── sources/                  # HelmRepositories & GitRepositories
│   │       ├── bitnami.yaml
│   │       ├── prometheus-community.yaml
│   │       └── kustomization.yaml
│   ├── staging/
│   │   └── kustomization.yaml        # Patches on top of base
│   └── production/
│       └── kustomization.yaml        # Patches on top of base (HA replicas, etc.)
│
├── apps/                             # Application deployments
│   ├── base/                         # Base app manifests (DRY)
│   │   ├── [app-name]/
│   │   │   ├── namespace.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── ingress.yaml
│   │   │   ├── helmrelease.yaml      # (if using Helm)
│   │   │   └── kustomization.yaml
│   │   └── [app-name-2]/
│   │       └── ...
│   ├── staging/
│   │   ├── kustomization.yaml        # References base, applies staging patches
│   │   └── [app-name]-patch.yaml     # e.g. replica count, resource limits
│   └── production/
│       ├── kustomization.yaml
│       └── [app-name]-patch.yaml
│
├── projects/                         # Optional: Multi-team project isolation
│   ├── team-a/
│   │   ├── rbac.yaml                 # Team-scoped RBAC
│   │   ├── namespace.yaml
│   │   └── kustomization.yaml
│   └── team-b/
│       └── ...
│
└── docs/                             # Documentation
    ├── runbooks/
    ├── architecture.md
    └── onboarding.md
```

---

## Reconciliation Order

Flux applies resources in dependency order. The chain is:

```
clusters/<env>/flux-system       (Flux controllers — bootstrapped)
        ↓
clusters/<env>/infrastructure    (Namespaces, Helm repos, controllers)
        ↓
clusters/<env>/apps              (Application workloads)
```

Dependency is enforced via `dependsOn` in each cluster Kustomization:

```yaml
# clusters/production/apps.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  dependsOn:
    - name: infrastructure        # ← waits for infra to be Ready
  interval: 10m
  path: ./apps/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

---

## Getting Started

### Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| `flux` CLI | `>= 2.x` | `brew install fluxcd/tap/flux` |
| `kubectl` | `>= 1.28` | [docs](https://kubernetes.io/docs/tasks/tools/) |
| `kubeconfig` | configured | — |

### Bootstrap a New Cluster

```bash
# 1. Export your GitHub token
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-github-username>

# 2. Verify prerequisites
flux check --pre

# 3. Bootstrap Flux onto the cluster
flux bootstrap github \
  --owner=${GITHUB_USER} \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/production \
  --personal
```

> Flux will commit `clusters/production/flux-system/` to the repo automatically.

### Add a New Application

```bash
# 1. Create base manifests
mkdir -p apps/base/my-app
# Add deployment.yaml, service.yaml, kustomization.yaml...

# 2. Create environment overlays
mkdir -p apps/staging apps/production
# Edit kustomization.yaml in each to include my-app

# 3. Commit and push — Flux handles the rest
git add . && git commit -m "feat: add my-app" && git push
```

---

## Environments

| Environment | Cluster | Branch/Path | Auto-sync |
|------------|---------|-------------|-----------|
| `dev` | `dev-cluster` | `./clusters/dev` | Yes |
| `staging` | `staging-cluster` | `./clusters/staging` | Yes |
| `production` | `prod-cluster` | `./clusters/production` | Gated (PRs only) |

---

## Secrets Management

Secrets are **never** stored in plaintext in this repository. Choose one:

- **[SOPS + Age](https://fluxcd.io/flux/guides/mozilla-sops/)** — Encrypt secrets in-repo
- **[Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)** — Encrypt with cluster key
- **[External Secrets Operator](https://external-secrets.io/)** — Pull from Vault / AWS SSM / etc.

```bash
# Example: encrypt a secret with SOPS + Age
sops --encrypt --age $(cat age.pub) secret.yaml > secret.enc.yaml
```

---

## Image Automation

Flux can automatically update image tags when new images are pushed:

```yaml
# Example: ImageUpdateAutomation watches a registry and opens PRs
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: [app-name]
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    commit:
      author:
        name: fluxbot
        email: flux@[your-org].io
      messageTemplate: "chore: update [app-name] image to {{range .Updated.Images}}{{.}}{{end}}"
    push:
      branch: main
  update:
    path: ./apps
    strategy: Setters
```

---

## Common Operations

```bash
# Watch reconciliation status
flux get kustomizations --watch

# Force immediate reconciliation
flux reconcile kustomization apps --with-source

# Suspend an app (stop syncing)
flux suspend kustomization [app-name]

# Resume
flux resume kustomization [app-name]

# View Flux events
kubectl get events -n flux-system --sort-by='.lastTimestamp'

# Check HelmRelease status
flux get helmreleases --all-namespaces
```

---

## Refs

- [FluxCD Documentation](https://fluxcd.io/flux/)
- [Flux Repository Structure Guide](https://fluxcd.io/flux/guides/repository-structure/)
- [Flux Upgrade Guide](https://fluxcd.io/flux/installation/upgrade/)
- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/)

---

## Contributing

1. Branch from `main`
2. Make changes in the appropriate `base/` or environment overlay
3. Open a PR — Flux will preview changes via [flux-local](https://github.com/allenporter/flux-local) (if configured)
4. Merge → Flux reconciles automatically

---

## Core Crew

| Name | Role | Contact |
|------|------|---------|
| [Name] | Platform Lead | @handle |
| [Name] | DevOps Engineer | @handle |

---

*Managed with ❤️ by [YOUR-ORG] Platform Team using [FluxCD](https://fluxcd.io/)*
