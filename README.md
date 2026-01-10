# DevOps Final Project (Mini Lab)

This repository is a mini DevOps platform + delivery pipeline built as a home lab.

Goal: fully automated software delivery starting from Git, using CI, CD (GitOps), Kubernetes, Vault, observability, and security gates, with infrastructure/bootstrap automated via Ansible.

## What you will see in the demo (end-to-end flow)

1. Developer pushes code to GitHub (feature branch)
2. Open Pull Request (PR)
3. PR CI pipeline runs quality + security gates:
   - Secret scan (gitleaks)
   - Lint + unit tests
   - SAST (semgrep)
   - Vulnerability scanning (trivy)
4. Merge to `main` triggers release pipeline:
   - Build Docker image (immutable artifact)
   - Scan image
   - Push image to **GitHub Container Registry (GHCR)**
   - Update GitOps overlay with new image tag and commit to Git
5. Argo CD detects GitOps change and deploys to Kubernetes (k3s)
6. Application pulls runtime secrets from Vault (no secrets stored in Git)
7. Database credentials are issued dynamically by Vault (TTL-based)
8. Prometheus/Grafana show metrics to prove health and traffic

## Architecture (High-Level)

### Runtime platform
- **k3s Kubernetes cluster (3 nodes)**
- **Argo CD** for GitOps deployments
- **HashiCorp Vault** for secrets (KV + dynamic DB creds)
- **Prometheus + Grafana** for observability
- **GitHub Container Registry (GHCR)** for images
- **Postgres** for DB + migrations

### CI/CD
- **GitHub Actions** (GitHub-hosted runners)
- Pipelines live in `.github/workflows/`

### Lab topology (hardware mapping)

#### Control workstation
- **Main PC (Control workstation)**: i9-14900K / 48GB RAM / 2TB NVMe  
  Used only for control: VS Code, kubectl, helm, ansible, browser dashboards (no workloads)

#### Kubernetes nodes (reserved IPs)
- **dell-optiplex-1** (Dell OptiPlex 3070) — **192.168.0.100**  
  Role: **k3s server (control-plane)**

- **hp-prodesk-1** (HP ProDesk 600 G4) — **192.168.0.101**  
  Role: **k3s agent node**  
  Storage: i5-8500 / 16GB RAM / 2×512GB NVMe (Ubuntu installed on one 512GB; second kept free for future `/data`)

- **hp-prodesk-2** (HP ProDesk 600 G4) — **192.168.0.102**  
  Role: **k3s agent node**  
  Storage: i5-8500 / 16GB RAM / 2×512GB NVMe (Ubuntu installed on one 512GB; second kept free for future `/data`)

#### Shared storage
- **NAS (Zyxel 326)**: NFS storage for Kubernetes PVs (Vault/Postgres/monitoring)

## Value Stream Map (VSM)

Value delivered: from code change to safely running in Kubernetes with evidence (metrics).

Stages:
1) Code + PR → reviewable change  
2) CI Security & Quality → green gates  
3) Build & Package → immutable container image  
4) Publish → image pushed to **GHCR**  
5) GitOps Update → desired state stored in Git  
6) Deploy → Argo CD syncs to k3s  
7) Verify + Observe → smoke test + Grafana evidence  

Target lead times (demo):
- PR checks: < 2 minutes
- Main pipeline to deployed: < 3 minutes

## Repository layout

- `app/` — application source code + Dockerfile + tests
- `db/` — database migrations
- `gitops/` — Kubernetes manifests (GitOps source of truth)
- `platform/ansible/` — Ansible inventory/playbooks/roles for lab bootstrap
- `.github/workflows/` — GitHub Actions pipelines (PR + main)
- `docs/` — detailed documentation (HLD/LLD/VSM/runbook/demo/deep dive)

## Demo script (12–15 minutes)

Tabs to open:
- GitHub PR
- GitHub Actions run
- Argo CD app page
- App `/version`
- Grafana dashboard

Order:
1) High-level design (1–2 min)
2) Repo layout (1 min)
3) PR: show CI gates (3–4 min)
4) Merge: build/push + GitOps update (3–4 min)
5) ArgoCD sync + rollout (1–2 min)
6) Validate `/version` (1 min)
7) Deep dive: Vault (k8s auth + dynamic DB creds TTL) (2–3 min)
8) Future improvements (30–60 sec)

## Deep dive (planned): Vault
- Kubernetes auth method (ServiceAccount → Vault role)
- Vault policies (least privilege)
- Secret injection into pods
- Dynamic DB credentials with TTL (database secrets engine)
- Audit logs (optional)

## Docs

More detailed docs live in `docs/`:
- `docs/HLD.md`
- `docs/LLD.md`
- `docs/VSM.md`
- `docs/RUNBOOK.md`
- `docs/DEMO_SCRIPT.md`
- `docs/DEEP_DIVE_VAULT.md`
