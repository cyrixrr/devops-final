
# Platform Help (Quick Commands)

This folder is a **cheat sheet** for the live demo and day-2 operations.

## Kubernetes quick checks

```bash
kubectl get nodes -o wide
kubectl get ns
kubectl -n argocd get applications
kubectl -n strapi get pods,svc,pvc
kubectl -n monitoring get pods | head
```

## GitOps: force refresh (Argo)

```bash
kubectl -n argocd annotate application monitoring-extras argocd.argoproj.io/refresh=hard --overwrite
kubectl -n argocd annotate application strapi argocd.argoproj.io/refresh=hard --overwrite
kubectl -n argocd get applications
```

## Strapi rollout checks

```bash
kubectl -n strapi rollout status deploy/strapi --timeout=180s
kubectl -n strapi get pod -l app=strapi -o jsonpath='{.items[0].spec.containers[0].image}{"\n"}'
kubectl -n strapi logs deploy/strapi --tail=120
```

## Monitoring dashboard checks

```bash
kubectl -n monitoring get cm strapi-grafana-dashboard -o name
kubectl -n monitoring logs statefulset/monitoring-kps-grafana -c grafana-sc-dashboard --tail=200
```

## Trigger CI (safe)

```bash
echo "# $(date -Iseconds)" >> strapi/CI_TRIGGER.md
git add strapi/CI_TRIGGER.md
git commit -m "chore(strapi): trigger CI"
git push origin main
```

## Rollback (GitOps)

```bash
git log --oneline -- gitops/apps/strapi/chart/values.yaml | head -n 10
git revert <bump_commit_sha>
git push origin main
```

## Access and Networking (Pi-hole + NPM)

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
