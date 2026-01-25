
# Runbook — Strapi GitOps Platform

Last updated: 2026-01-25 22:45:56

## Quick links

- Strapi admin: `http://strapi.apps.home.arpa/admin`
- Argo CD: `https://192.168.0.101:30080`
- Grafana dashboard: “Strapi - Kubernetes Overview”

## Day-0: Verify cluster baseline

```bash
kubectl -n argocd get pods
kubectl -n monitoring get pods
kubectl -n nfs-provisioner get pods
```

## Deploy / Update Strapi (normal path)

✅ Normal deploy happens by merging the CI-generated PR that bumps:

`gitops/apps/strapi/chart/values.yaml` → `image.tag: "<SHA>"`

To check status:
```bash
kubectl -n strapi rollout status deploy/strapi --timeout=180s
kubectl -n strapi get pods -l app=strapi -o wide
kubectl -n strapi get pod -l app=strapi -o jsonpath='{.items[0].spec.containers[0].image}{"\n"}'
```

## Rollback (GitOps-safe)

Option 1: **Revert the bump commit** (recommended)
```bash
git log --oneline -- gitops/apps/strapi/chart/values.yaml | head -n 10
git revert <bump_commit_sha>
git push origin main
```

Option 2: **New PR setting tag to previous SHA**
- Edit `values.yaml` tag to known-good SHA
- PR → merge → Argo deploys it

## Common issues & fixes

### 1) Strapi CrashLoop: DB auth failed
Symptoms:
- Logs show: `password authentication failed for user "strapi"`

Checks:
```bash
kubectl -n strapi get secret strapi-postgres-credentials -o yaml
kubectl -n strapi logs deploy/strapi --tail=120
kubectl -n strapi logs statefulset/strapi-postgresql --tail=120
```

Fix:
- Ensure Strapi uses secret key `password`
- Ensure Postgres is initialized with matching password
- If Postgres already has old password in its data dir, changing secret alone won’t change DB users.
  In that case, either update DB user password inside Postgres or recreate the DB volume (only for non-prod/demo).

### 2) Argo app OutOfSync / ComparisonError
```bash
kubectl -n argocd get app monitoring-extras -o yaml | sed -n '1,160p'
kubectl -n argocd annotate app monitoring-extras argocd.argoproj.io/refresh=hard --overwrite
```

### 3) Dashboard not visible in Grafana
Requirements:
- ConfigMap in `monitoring` with label `grafana_dashboard: "1"`

Check:
```bash
kubectl -n monitoring get cm strapi-grafana-dashboard -o yaml | sed -n '1,80p'
kubectl -n monitoring logs statefulset/monitoring-kps-grafana -c grafana-sc-dashboard --tail=200
```

### 4) PVC pending
```bash
kubectl -n strapi get pvc
kubectl -n nfs-provisioner logs deploy/nfs-subdir-nfs-subdir-external-provisioner --tail=200
```

## Operational “health” commands

```bash
kubectl -n strapi get events --sort-by=.metadata.creationTimestamp | tail -n 30
kubectl -n strapi top pods 2>/dev/null || true
kubectl -n monitoring get pods
```

## DNS and Reverse Proxy

### Troubleshooting: DNS / Reverse Proxy (Pi-hole + NPM)

Quick checks:
- DNS resolves:
  - `dig +short strapi.apps.home.arpa`
  - `dig +short argocd.apps.home.arpa`
- NPM can reach upstreams:
  - `curl -I http://192.168.0.101:31337/admin`
  - `curl -kI https://192.168.0.100:30443/`
  - `curl -I http://192.168.0.100:30030/`

If a hostname resolves but UI is down:
- verify the upstream NodePort Service exists (`kubectl -n <ns> get svc`)
- verify the Pod is Ready (`kubectl -n <ns> get pods`)
- check NPM host is Online and points to the correct destination

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
