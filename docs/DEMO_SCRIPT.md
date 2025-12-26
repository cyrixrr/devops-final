
---

## `docs/DEMO_SCRIPT.md`
```md
# Demo Script (Draft)

## Tabs open (prepare in advance)
- GitHub repo (Commits / PR view)
- GitHub Actions (latest runs)
- Argo CD UI (Application view)
- Terminal: kubectl checks
- App `/version` endpoint (browser tab or curl)

## Preset environment rule (avoid waiting)
- k3s cluster already running
- Argo CD already installed and reachable
- myapp already registered in Argo CD (Application exists)
- One successful GH Actions run already exists (so you can show logs quickly)

---

## Order (12–15 minutes)

### 1) High-level solution design (1–2 min)
Say the flow:
**GitHub → CI → build image → push GHCR → GitOps commit → Argo CD sync → k3s deploy → /version proof**
Mention: Git is the source of truth for desired state (GitOps).

### 2) Low-level design + lab topology (1–2 min)
Show node roles + IPs:
- dell-optiplex-1 (server) 192.168.0.100
- hp-prodesk-1 (agent) 192.168.0.101
- hp-prodesk-2 (agent) 192.168.0.102

### 3) “As code” bootstrap (Ansible) (1 min)
Show:
- `platform/ansible/playbooks/01-baseline.yml` (k8s readiness)
- `platform/ansible/playbooks/03-k3s.yml` (k3s install + kubeconfig fetch)

### 4) GitOps structure (Kustomize + Argo) (1–2 min)
Show Kustomize:
- base + overlay:
  - `gitops/apps/myapp/base`
  - `gitops/apps/myapp/overlays/dev`
Render quickly:
```bash
kubectl kustomize gitops/apps/myapp/overlays/dev | sed -n '1,80p'

- PR gates demo trigger: Fri 26 Dec 2025 02:54:35 AM EET
