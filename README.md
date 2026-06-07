# boukingolts-cluster

GitOps cluster configuration for [boukingolts.art](https://boukingolts.art) — a live, multi-tenant art gallery platform. This repository is the single source of truth for everything running in the production Kubernetes cluster. ArgoCD watches this repo and continuously reconciles cluster state to match.

---

## Live Environment

| URL | Description |
|-----|-------------|
| [boukingolts.art](https://boukingolts.art) | Combined gallery view |
| [elena.boukingolts.art](https://elena.boukingolts.art) | Elena's gallery |
| [alexey.boukingolts.art](https://alexey.boukingolts.art) | Alexey's gallery |
| [api.boukingolts.art](https://api.boukingolts.art) | Backend API |
| [grafana.boukingolts.art](https://grafana.boukingolts.art) | Monitoring dashboard |
| [argocd.boukingolts.art](https://argocd.boukingolts.art) | ArgoCD GitOps UI |
| [staging.boukingolts.art](https://staging.boukingolts.art) | Staging frontend |
| [staging.api.boukingolts.art](https://staging.api.boukingolts.art) | Staging API |

---

## Infrastructure

**Cluster:** k3s on Oracle Cloud Infrastructure (ARM64/Ampere A1, il-jerusalem-1)  
**Ingress:** Traefik with automatic Let's Encrypt TLS  
**GitOps:** ArgoCD App of Apps — all workloads managed declaratively from this repository  
**Secrets:** Sealed Secrets (namespace-scoped encryption; controller key backed up externally)

---

## Architecture

```
GitHub (boukingolts-backend)
    │
    ├── push to dev ──► GitHub Actions CI
    │                       │  audit → build (linux/amd64 + arm64) → trivy scan
    │                       │  push to GHCR (ghcr.io/yevgenio/gallery-backend:<sha>)
    │                       └──► update staging values.yaml ──► ArgoCD ──► gallery-backend-staging
    │
    └── merge to main ──► GitHub Actions CI (promote only)
                              │  read verified tag from staging values.yaml
                              └──► write to prod values.yaml ──► ArgoCD ──► gallery-backend (prod)
```

**Build-once-promote:** the image built and verified on `dev` is the exact binary promoted to production. The `main` branch never triggers a rebuild — it only updates the image tag, ensuring what ran in staging is what runs in production.

---

## Deployed Applications

| App | Namespace | Description |
|-----|-----------|-------------|
| `gallery-backend` | `gallery-backend` | Node.js/Express API, 2 replicas, port 5000 |
| `gallery-backend-staging` | `gallery-backend-staging` | Staging API (isolated namespace + dedicated database) |
| `sealed-secrets` | `sealed-secrets` | In-cluster secrets decryption controller |
| `kube-prometheus-stack` | `monitoring` | Prometheus (45-day retention) + Grafana |
| `monitoring-extras` | `monitoring` | ServiceMonitor, custom Grafana dashboard |
| `argocd` | `argocd` | ArgoCD self-management configuration |

---

## Repository Structure

```
boukingolts-cluster/
├── apps/                               # ArgoCD Application definitions (App of Apps)
│   ├── root.yaml                       # Root application — watches the apps/ folder
│   ├── gallery-backend.yaml            # Production backend
│   ├── gallery-backend-staging.yaml    # Staging backend
│   ├── monitoring.yaml                 # kube-prometheus-stack
│   ├── monitoring-extras.yaml          # ServiceMonitor + Grafana dashboard
│   ├── sealed-secrets.yaml             # Sealed Secrets controller
│   └── argocd.yaml                     # ArgoCD self-management
│
└── manifests/                          # Helm charts per application
    ├── gallery-backend/                # Deployment, Service, IngressRoute, SealedSecret
    ├── gallery-backend-staging/        # Same structure — separate namespace and secrets
    ├── monitoring/                     # Grafana dashboard JSON, ServiceMonitor, IngressRoute
    └── argocd/                         # IngressRoute + params ConfigMap
```

Changes to the cluster are made by committing to this repository — not by running `kubectl` commands directly. ArgoCD detects changes and reconciles within seconds via webhook.

---

## CI/CD Flow

**Staging deploy** (on push to `dev` in `boukingolts-backend`):
1. GitHub Actions builds a multi-arch image (amd64 + arm64), runs npm audit and Trivy CVE scan
2. Image pushed to GHCR tagged with the commit SHA
3. `manifests/gallery-backend-staging/values.yaml` updated with the new tag
4. ArgoCD deploys to the `gallery-backend-staging` namespace automatically

**Production promote** (on merge to `main` in `boukingolts-backend`):
1. CI reads the verified image tag from `gallery-backend-staging/values.yaml`
2. Writes it to `gallery-backend/values.yaml` — no Docker build, no new image
3. ArgoCD deploys the same image to production

**Rollback:** revert the `values.yaml` commit; ArgoCD reconciles automatically within seconds.

---

## Observability

Prometheus scrapes the backend API's `/metrics` endpoint every 30 seconds via a dedicated ServiceMonitor. Custom application metrics:

- `http_requests_total` — labeled by method, route, status, and artist
- `http_request_duration_seconds` — p95 latency histogram
- `gallery_product_views_total` — per-artwork view counters, labeled by artist
- `gallery_event_views_total` — per-event view counters, labeled by artist

The Grafana "Gallery Overview" dashboard surfaces request rate, p95 latency, error rate, top artwork/event views, and backend memory — with an artist filter variable for per-tenant traffic isolation.

A `metricRelabelings` rule on the ServiceMonitor collapses high-cardinality bot-probe routes into a single series before storage, keeping Prometheus storage predictable at scale.

---

## Secrets Management

All secrets are stored as `SealedSecret` resources — encrypted in Git, decryptable only by the Sealed Secrets controller running in the cluster. Secrets are namespace-scoped: a secret sealed for `gallery-backend` cannot be decrypted in `gallery-backend-staging` and vice versa.

The controller's private key is not stored in this repository. It is backed up to an external secrets store.

---

## Multi-Tenancy

The platform serves multiple independent galleries from a single backend deployment. Tenant isolation is enforced at every layer:

- **Edge:** incoming subdomain mapped to an `x-artist` request header by the Next.js edge proxy
- **Database:** all queries filtered by the `artist` field — no cross-tenant data leakage possible at the query level
- **Admin access:** the archive subdomain is JWT-gated at the edge before requests reach the application
- **Observability:** Grafana artist filter variable isolates per-tenant metrics without separate deployments
