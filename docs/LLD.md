
# Low-Level Design (LLD) — DevOps Final Project

Last updated: 2026-01-25 22:45:56

## Repository layout (what matters for the demo)

- **App source:** `strapi/`
- **CI pipeline:** `.github/workflows/strapi-ci.yml`
- **GitOps apps (Argo):** `gitops/argocd/`
- **Strapi Helm chart:** `gitops/apps/strapi/chart/`
- **PostgreSQL Helm values:** `gitops/helm/postgresql/strapi-values.yaml`
- **Monitoring stack values:** `gitops/helm/kube-prometheus-stack/values.yaml`
- **Monitoring extras (dashboards via kustomize):** `gitops/monitoring/`

## Kubernetes namespaces

- `argocd` — Argo CD components
- `strapi` — Strapi + PostgreSQL (app + DB)
- `monitoring` — Prometheus, Grafana, Alertmanager (kube-prometheus-stack)
- `nfs-provisioner` — NFS external provisioner

## Strapi deployment (runtime design)

**Service**
- Type: NodePort
- Port: 1337 → NodePort `31337`
- External access: through NPM at `http://strapi.apps.home.arpa`

**Persistence**
- PVC: `strapi-uploads` (StorageClass: `nfs-client`)
- Mount: `/opt/app/public/uploads`

**Secrets**
- DB password: secret `strapi-postgres-credentials`, key `password`
- Strapi secrets: secret `strapi-secrets` (keys: `APP_KEYS`, `API_TOKEN_SALT`, `ADMIN_JWT_SECRET`, `JWT_SECRET`)

## PostgreSQL (StatefulSet design)

- Service: `strapi-postgresql` (ClusterIP, port 5432)
- PVC: `data-strapi-postgresql-0` (StorageClass: `nfs-client`)
- User/DB: `strapi`
- Password: from `strapi-postgres-credentials`

## GitOps (Argo CD applications)

Apps live in: `gitops/argocd/apps/*.yaml`

- `platform-apps` (root “apps-of-apps”) → manages:
  - `monitoring-kps` (kube-prometheus-stack)
  - `monitoring-extras` (dashboards/configmaps via kustomize)
  - `strapi` (Strapi Helm chart)
  - `strapi-postgresql` (PostgreSQL Helm release for Strapi)

Sync policy:
- Automated sync enabled
- Self-heal enabled
- Prune enabled

## Monitoring extras (Grafana dashboard)

Grafana dashboard sidecar is configured to watch ConfigMaps with:
- label `grafana_dashboard: "1"`
- in namespace `monitoring`

Dashboard ConfigMap is managed via Argo CD app `monitoring-extras` from path:
- `gitops/monitoring/`
  - `kustomization.yaml`
  - `dashboards/strapi-dashboard-cm.yaml`
  - `placeholders/keep.yaml` (keeps kustomize non-empty)

## CI pipeline (Strapi)

Workflow: `.github/workflows/strapi-ci.yml`

- Trigger: push to `main` (paths `strapi/**` or workflow file)
- Steps:
  1. Checkout code
  2. Install deps + build Strapi admin
  3. Build image + push to GHCR with **immutable tag = commit SHA**
  4. Trivy scan HIGH/CRITICAL (non-blocking for the exam demo)
  5. Create PR updating `gitops/apps/strapi/chart/values.yaml` → `image.tag: "<SHA>"`

Why PR (not direct push):
- Keeps GitOps changes visible/auditable
- Perfect “demo moment”: merge PR → Argo auto-rollout

## Quick verification commands (copy/paste)

```bash
# Strapi is running and uses immutable image tag
kubectl -n strapi get pods -l app=strapi -o wide
kubectl -n strapi get pod -l app=strapi -o jsonpath='{.items[0].spec.containers[0].image}{"\n"}'

# Argo apps health
kubectl -n argocd get applications

# Dashboard ConfigMap exists (Grafana sidecar will pick it up)
kubectl -n monitoring get configmap strapi-grafana-dashboard -o name
```

## Access and Networking

### Local DNS + Reverse Proxy (Pi-hole + Nginx Proxy Manager)

This lab uses **Pi-hole as local DNS** and **Nginx Proxy Manager (NPM)** as the reverse proxy for friendly URLs under `*.apps.home.arpa`.

**Flow (browser → app):**
1. Client resolves `strapi.apps.home.arpa` via **Pi-hole**
2. Pi-hole returns the IP of the **NPM** host
3. NPM routes the request to the correct upstream (NodePort / service)

**NPM Proxy Hosts (current):**
| Hostname | Upstream (destination) | Notes |
|---|---:|---|
| `argocd.apps.home.arpa` | `https://192.168.0.100:30443` | Custom cert (TLS) |
| `grafana.apps.home.arpa` | `http://192.168.0.100:30030` | HTTP only |
| `strapi.apps.home.arpa` | `http://192.168.0.101:31337` | HTTP only (Strapi admin) |
| `pihole.apps.home.arpa` | `http://192.168.0.200:80` | Pi-hole UI |
| `proxmox.apps.home.arpa` | `https://192.168.0.11:8006` | Custom cert (TLS) |
| `plex.apps.home.arpa` | `http://192.168.0.210:32400` | Plex UI |
| `nas223.apps.home.arpa` | `http://192.168.0.98:5000` | Synology UI |

> **Note:** Traefik is used only as the **k3s Service LoadBalancer** implementation and exposes `192.168.0.100`, `192.168.0.101`, `192.168.0.102`. External access for the demo uses **Pi-hole + NPM**, not Ingress.

```mermaid
flowchart LR
  U[User Browser] -->|DNS query| P[Pi-hole DNS]
  P -->|A record: NPM IP| U
  U -->|HTTP(S) request| N[Nginx Proxy Manager]
  N -->|proxy_pass to NodePort| K[k3s nodes: 192.168.0.100/101/102]
  K -->|Service/Pod| S[Strapi / ArgoCD / Grafana]
```
