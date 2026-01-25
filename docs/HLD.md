
# High-Level Design (HLD) — DevOps Final Project

Last updated: 2026-01-25 22:45:56

## Goal

Deliver a **real GitOps CD loop** for a production-like app (**Strapi + PostgreSQL**) on a Kubernetes cluster (k3s),
with CI building immutable images, Git as the deployment source of truth, automated rollout via Argo CD,
and basic observability via Prometheus + Grafana.

## What is deployed

- **Application:** Strapi CMS
- **Database:** PostgreSQL (StatefulSet)
- **Kubernetes:** k3s (3 nodes) + Traefik (cluster ingress capability, but access is via external NPM reverse proxy)
- **GitOps:** Argo CD (apps-of-apps style)
- **Registry:** GitHub Container Registry (GHCR)
- **Observability:** kube-prometheus-stack + Grafana dashboard loaded via ConfigMap sidecar
- **Storage:** NFS dynamic provisioning (nfs-subdir-external-provisioner) for PVCs

## High-level flow (CI → GitOps → CD)

1. Developer pushes to `main` (changes in `strapi/**`).
2. GitHub Actions builds Strapi image and pushes to GHCR tagged with **commit SHA** (immutable).
3. CI opens a PR that bumps Helm values: `gitops/apps/strapi/chart/values.yaml` → `image.tag: "<SHA>"`.
4. Merge PR → Argo CD detects Git change and auto-syncs → Kubernetes rollout happens automatically.
5. Grafana dashboard shows Kubernetes-level health for Strapi pods (CPU/memory/restarts/replicas).

## Architecture diagram (Mermaid “Visio-like” code)

```mermaid
flowchart LR
  dev[Developer
Git push] --> gh[GitHub repo
cyrixrr/devops-final]
  gh --> gha[GitHub Actions
strapi-ci.yml]
  gha --> ghcr[GHCR
strapi:<commit-sha>]
  gha --> pr[PR: bump image.tag
(values.yaml)]
  pr -->|merge| gitops[GitOps repo state
(main)]
  gitops --> argo[Argo CD
https://192.168.0.101:30080]
  argo --> k3s[k3s cluster]
  k3s --> strapi[Strapi Deployment
ns: strapi]
  k3s --> pg[PostgreSQL StatefulSet
ns: strapi]
  nfs[NFS provisioner
ns: nfs-provisioner] -->|PVC| strapi
  nfs -->|PVC| pg
  k3s --> prom[Prometheus
(kube-prometheus-stack)]
  prom --> graf[Grafana
Dashboards via ConfigMap sidecar]
  user[User Browser] --> npm[Nginx Proxy Manager + Pi-hole DNS]
  npm --> strapi
  user --> graf
  user --> argo
```

## Topics covered (exam checklist)

- SDLC (design → build → deploy → operate)
- Source control + branching (main, bot PRs for GitOps bump)
- CI (build + security scan + image publish)
- CD (Argo auto-sync from Git)
- GitOps (Git is source-of-truth, immutable tags)
- Docker (container build/push)
- Kubernetes (Deployments, StatefulSets, Services, PVCs)
- Configuration & secrets (K8s Secrets for DB + Strapi secrets)
- Observability (Prometheus + Grafana + dashboard provisioning)
- Security (Trivy image scan, no hard-coded secrets in repo)
- IaC-ish (cluster provisioning automated via Ansible playbooks in `/platform/ansible`)

## Environments / access

- Strapi admin: `http://strapi.apps.home.arpa/admin`
- Argo CD UI: `https://192.168.0.101:30080` (NodePort)
- Grafana: NodePort `30030` (exposed internally)

> Note: Ingress is not required because DNS is handled by Pi-hole and traffic is forwarded by Nginx Proxy Manager.

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
