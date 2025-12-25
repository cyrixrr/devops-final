# High-Level Design (HLD)

Last updated: 2025-12-25

## Goal
Build an automated software delivery process in a home lab:
**GitHub → CI gates → build image → scan → push → GitOps commit → Argo CD deploy → observe**
with **Vault** for secrets (deep dive vertical).

## Architecture Overview
### Control workstation (Main PC)
Used only as a control unit:
- VS Code + Git
- Ansible
- kubectl/helm
- Browser for Argo CD UI

### Lab topology (compute)
- `dell-optiplex-1` (192.168.0.100): **k3s server** (control-plane)
- `hp-prodesk-1` (192.168.0.101): **k3s agent**
- `hp-prodesk-2` (192.168.0.102): **k3s agent**
- NAS Zyxel 326: planned NFS storage for PVs (optional later)

### Platform components (current + planned)
**Current (already working):**
- k3s Kubernetes cluster (installed via Ansible)
- Argo CD (installed via Helm; exposed via NodePort)
- GitOps application example (`myapp`) deployed from repo via Argo CD

**Planned next:**
- CI pipelines (GitHub Actions): gitleaks, lint/tests, semgrep, trivy, build/push image
- Registry (local or GHCR)
- Observability (Prometheus + Grafana)
- Vault (Kubernetes auth + dynamic DB creds) + deep dive
- Postgres + migrations
- Optional NFS PVs from NAS

## Current implemented slice (already proven)
✅ GitOps CD path:
- GitHub repo contains Kustomize manifests under `gitops/`
- Argo CD syncs from `main` and deploys to k3s automatically
- Sample app (`myapp`) is deployed and reachable via NodePort

✅ “As code” platform layer:
- k3s installed via Ansible playbook
- Argo CD installed via Helm values file
- app exposure controlled via GitOps overlay patch (NodePort)

## Demo entry points (today)
- Argo CD UI (HTTPS): `https://192.168.0.100:30443`
- Sample app (`myapp`): `http://192.168.0.100:32094`

## Topics covered already (from course list)
- Source control
- Infrastructure as code (Ansible, Helm values, GitOps manifests)
- Kubernetes
- Continuous Delivery (GitOps)
- Docker (used for runtime later; app image build planned)
- Security + secrets management (planned via Vault)
- Observability (planned)
