# High-Level Design (HLD)

## Goal
Build an automated software delivery process (CI + CD) in a home lab:

**GitHub → CI gates → build Docker image → push to GHCR → GitOps commit → Argo CD sync → k3s deploy → verify (/version) → (later) observability + Vault**

## Architecture Overview

### Code and delivery flow
1. Developer pushes code to GitHub (feature branch / main)
2. CI runs (tests + security gates) on GitHub Actions
3. Build an immutable Docker image and push to GHCR:
   - `ghcr.io/cyrixrr/myapp:<commit_sha>`
   - `ghcr.io/cyrixrr/myapp:latest`
4. Pipeline updates GitOps manifests (as code) to use `<commit_sha>`
5. Argo CD detects Git change and reconciles desired state into Kubernetes (k3s)
6. App is reachable via NodePort; `/version` shows the deployed commit SHA

### Runtime platform
- **k3s (3 nodes)** for Kubernetes
- **Argo CD** for GitOps (continuous delivery)
- **Kustomize** for Kubernetes manifest composition (base + overlays)
- **GHCR** (GitHub Container Registry) for container images
- (Planned) **Vault** for secrets management and deep dive
- (Planned) **Prometheus/Grafana** for observability evidence

## Lab topology (hardware mapping)
- **Control workstation (main PC)**: VS Code, kubectl, helm, ansible, browser
- **dell-optiplex-1** (192.168.0.100): k3s server (control-plane)
- **hp-prodesk-1** (192.168.0.101): k3s agent
- **hp-prodesk-2** (192.168.0.102): k3s agent
- **NAS (Zyxel 326)**: planned NFS storage (optional later)

## “As code” principle
- Kubernetes manifests: `gitops/`
- Bootstrap/cluster install: `platform/ansible/`
- CI pipelines: `.github/workflows/`
- Documentation: `docs/`

## Deep dive vertical (chosen)
**GitOps** (Argo CD + Kustomize) + automated GitOps updates from CI.

(Next deep dive planned: Vault)
