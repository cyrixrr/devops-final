
---

## `docs/DEMO_SCRIPT.md`
```markdown
# Demo Script (Draft)

## Tabs open (prepare in advance)
- GitHub repo (PR view)
- GitHub Actions (latest runs)
- Argo CD UI: `https://192.168.0.100:30443`
- App URL (NodePort): `http://192.168.0.100:32094`
- Grafana dashboard (later milestone)
- (Optional) Vault UI/CLI tab for deep dive (later milestone)

## Preset environment rule
- k3s cluster is already running
- Argo CD installed via Helm and reachable via NodePort
- GitOps app exists and is synced (no waiting)
- Monitoring/Vault/CI will be added next (when ready)

---

## Order (12–15 minutes)

### 1) High-level solution design (1–2 min)
- One sentence flow: **GitHub → CI gates → build image → scan → push → GitOps commit → Argo CD deploy → observe**
- Mention secrets: **Vault (no secrets in Git)** (planned deep dive)

### 2) Low-level design + lab topology (1–2 min)
- Show node roles + IPs:
  - `dell-optiplex-1` (k3s server) `192.168.0.100`
  - `hp-prodesk-1` (agent) `192.168.0.101`
  - `hp-prodesk-2` (agent) `192.168.0.102`

### 3) “As code” bootstrap (Ansible + Helm + GitOps) (2 min)
- Ansible:
  - `platform/ansible/playbooks/01-baseline.yml` (prep only)
  - `platform/ansible/playbooks/03-k3s.yml` (install cluster)
- Helm:
  - `platform/helm/argocd-values.yaml` (NodePorts pinned)
- GitOps:
  - `gitops/apps/myapp/...` (Kustomize + Argo App)

Quick proof:
```bash
kubectl --kubeconfig ~/git/devops-final/kubeconfig/k3s.yaml get nodes -o wide
kubectl -n argocd get applications
kubectl -n myapp get pods -o wide
