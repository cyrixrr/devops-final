
# Deep Dive — GitOps CD loop (Strapi)

Last updated: 2026-01-25 22:45:56

## The “rule”

**Git is the single source of truth for what runs in the cluster.**
The registry (GHCR) is just an artifact store.

That means:
- Kubernetes **does not** track `:latest`
- Kubernetes runs **exactly** the tag written in Git (`values.yaml`)
- A deployment happens only when Git changes (PR merged)

## The loop in practice

### 1) CI produces an immutable artifact
- Every build is tagged with the **full commit SHA**
- Example: `ghcr.io/cyrixrr/strapi:00ef021b0d13673172a44a5ca7a4a53332d3e322`
- Immutable tags make rollbacks and audits trivial

### 2) CI proposes the deployment change as a PR
CI creates a PR that changes only:

- `gitops/apps/strapi/chart/values.yaml`
  - `image.tag: "<SHA>"`

This is the “money shot” because you can show:
- What changed
- Why it changed
- The exact version that will run

### 3) Merge PR → Argo CD sync → rollout
Argo watches the repo branch `main`.
After merge:
- Argo auto-syncs the Strapi app
- Kubernetes does a Deployment rollout
- New pod runs the new image tag

## How to demo (fast + reliable)

### A) Show the PR
In GitHub:
- Open the latest PR from `github-actions[bot]`
- Confirm it changes only `values.yaml`

### B) Merge it
- Merge PR
- Immediately go to Argo CD UI and show the app syncing

### C) Prove rollout in the cluster
```bash
kubectl -n strapi rollout status deploy/strapi --timeout=180s
kubectl -n strapi get pod -l app=strapi -o jsonpath='{.items[0].spec.containers[0].image}{"\n"}'
```

### D) Prove it from the UI
- `http://strapi.apps.home.arpa/admin` loads
- Grafana “Strapi - Kubernetes Overview” dashboard shows the pod health

## Why this is “real GitOps”

- No manual `kubectl apply` in the deployment path
- No “latest tag drift” (non-repeatable deployments)
- Every rollout is a Git commit (audit trail)
- Rollback is a Git revert

## Failure modes & what you say in the exam

- “If image pull fails” → check GHCR perms + tag in `values.yaml`
- “If DB auth fails” → secret key mismatch; verify `strapi-postgres-credentials`
- “If Argo shows OutOfSync” → refresh hard; check manifests render
- “If dashboard missing” → ConfigMap label `grafana_dashboard: "1"` and namespace `monitoring`

## Rollback (the clean way)

Rollback = set `image.tag` back to a previous SHA (via PR or revert).

```bash
# Find previous deploy commits
git log --oneline -- gitops/apps/strapi/chart/values.yaml | head

# Revert the bump commit (creates new commit)
git revert <commit_sha>
git push
```

## Demo Access

### Demo URLs used during the GitOps deep dive

- Argo CD: `https://argocd.apps.home.arpa`
- Strapi Admin: `http://strapi.apps.home.arpa/admin`
- Grafana: `http://grafana.apps.home.arpa`

These are served via **Pi-hole DNS + NPM**, while Argo CD deploys workloads to k3s.

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
