# High-Level Design (HLD)

## Goal
Build an automated software delivery process:
GitHub → CI → container image → registry → GitOps → Kubernetes → Vault secrets → observability.

## Architecture Overview
Developer workflow:
- Work on a feature branch
- Open Pull Request (PR)
- CI checks run
- Merge to main triggers build + release
- GitOps update triggers deploy

Runtime platform:
- k3s Kubernetes cluster (3 nodes)
- Argo CD for GitOps deployments
- Vault for secrets (including dynamic DB creds)
- Prometheus + Grafana for observability
- Local container registry for images

## Logical Flow (end-to-end)
1. Developer pushes code to GitHub
2. GitHub Actions (self-hosted runner) runs PR checks:
   - gitleaks → lint/tests → semgrep → trivy
3. Merge to main runs release pipeline:
   - build image → scan → push to registry
   - update GitOps overlay (new image tag) and commit
4. Argo CD detects GitOps change and syncs to k3s
5. App starts in k3s:
   - reads secrets from Vault (no secrets in Git)
   - uses Vault dynamic DB credentials
6. Observability:
   - Prometheus scrapes metrics
   - Grafana dashboards show health + request metrics

## Lab Topology (physical)
- Main PC: control workstation (VS Code, kubectl, helm, ansible)
- Small PC #1: k3s server + GitHub runner + registry
- Small PC #2: k3s agent
- Small PC #3: k3s agent
- NAS: NFS storage for registry + Kubernetes PVs (Vault/Postgres/etc.)
