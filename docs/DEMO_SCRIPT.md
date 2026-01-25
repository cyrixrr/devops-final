
# 12–15 min Demo Script (copy/paste friendly)

Last updated: 2026-01-25 22:45:56

Goal: demonstrate end-to-end DevOps pipeline with **real GitOps CD** for Strapi + Postgres on k3s.

## 0) Pre-demo checklist (30s)
```bash
kubectl -n argocd get applications
kubectl -n strapi get pods,svc,pvc
kubectl -n monitoring get pods | head
```
Open tabs in browser:
- Strapi: `http://strapi.apps.home.arpa/admin`
- ArgoCD: `https://192.168.0.101:30080`
- Grafana: NodePort `30030` (or your NPM URL if you proxy it)

---

## 1) High-level design (1 min)
Say:
- “GitHub Actions builds immutable image (commit SHA) → opens PR bumping GitOps values → merge triggers Argo auto-sync → rollout.”
- “Observability is via Prometheus/Grafana; dashboard is provisioned from Git via Argo.”

Show diagram (in `docs/HLD.md`).

---

## 2) Low-level design (2 min)
```bash
kubectl -n strapi get deploy,svc,pvc
kubectl -n strapi get pod -l app=strapi -o jsonpath='{.items[0].spec.containers[0].image}{"\n"}'
kubectl -n strapi get secret strapi-postgres-credentials -o name
kubectl -n argocd get applications
```
Explain quickly:
- Strapi runs as Deployment, Postgres as StatefulSet
- Uploads PVC mounted at `/opt/app/public/uploads`
- DB creds and Strapi secrets are K8s secrets

---

## 3) The deep dive (GitOps “money shot”) (5–6 min)

### A) Trigger CI (already safe)
Edit a harmless file:
```bash
echo "# $(date -Iseconds)" >> strapi/CI_TRIGGER.md
git add strapi/CI_TRIGGER.md
git commit -m "chore(strapi): trigger CI"
git push origin main
```

### B) Show GitHub Actions run
In GitHub → Actions:
- Show it builds/pushes `ghcr.io/.../strapi:<SHA>`
- Show Trivy scan output

### C) Show PR created by bot
In GitHub → Pull Requests:
- Open newest PR from `github-actions[bot]`
- Confirm only file changed:
  - `gitops/apps/strapi/chart/values.yaml`
  - `image.tag: "<SHA>"`

### D) Merge the PR
Merge PR → say:
- “Now Git is updated, so Argo will deploy it.”

### E) Watch Argo + rollout
```bash
kubectl -n argocd get applications
kubectl -n strapi rollout status deploy/strapi --timeout=180s
kubectl -n strapi get pod -l app=strapi -o jsonpath='{.items[0].spec.containers[0].image}{"\n"}'
```
Explain:
- “The running image tag matches the Git commit SHA.”

---

## 4) Observability proof (2 min)
Grafana:
- Open dashboard: **“Strapi - Kubernetes Overview”**
- Show panels: CPU, memory, restarts, replicas, pods ready
Say:
- “This is Kubernetes-level health, no app code changes.”

(Optional quick load test):
```bash
for i in $(seq 1 30); do curl -s -o /dev/null -w "%{http_code}\n" http://strapi.apps.home.arpa/admin/; done
```

---

## 5) Security & operations (2 min)
Say:
- “Secrets are not in Git; DB password is K8s secret.”
- “Image is scanned in CI (Trivy).”
- “Rollback is Git revert (auditable).”

Show runbook:
- `docs/RUNBOOK.md`

---

## 6) Future improvements (1 min)
- Add app-level metrics (/metrics) + ServiceMonitor + latency/RPS dashboard
- Policy checks: SAST + secret scanning, enforce HIGH/CRITICAL fail on main
- Add staging/prod overlays (Helm values per env) + promotion PRs
- Add automated DB migrations gate (pre-sync hook)

---

## 7) Q&A buffer (1–2 min)
Stay ready with:
```bash
kubectl -n strapi describe pod -l app=strapi | sed -n '1,120p'
kubectl -n argocd get app strapi -o wide
```

## Access Links

### Open the UIs (fast)

- Argo CD: `https://argocd.apps.home.arpa`
- Strapi: `http://strapi.apps.home.arpa/admin`
- Grafana: `http://grafana.apps.home.arpa`

(All are reachable via **Pi-hole DNS + Nginx Proxy Manager**.)

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
