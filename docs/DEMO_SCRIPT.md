# Demo Script (Draft)

## Tabs open (prepare in advance)
- GitHub repo (PR view)
- GitHub Actions (latest runs)
- Argo CD UI (Application view): `https://192.168.0.100:32422`
- App `/version` endpoint (browser tab or curl) *(after app is deployed)*
- Grafana dashboard (or Prometheus targets page) *(after monitoring is deployed)*
- (Optional) Vault UI/CLI tab for deep dive *(after Vault is deployed)*

## Preset environment rule
- k3s cluster is already running
- Argo CD installed via Helm and reachable via NodePort
- Argo CD already connected to GitOps repo *(once we add the Application)*
- Monitoring already running (Prometheus/Grafana) *(later milestone)*
- Vault already unsealed and configured *(later milestone)*
- Pipeline caches warmed (avoid long waits) *(later milestone once CI is ready)*

---

## Order (12–15 minutes)

### 1) High-level solution design (1–2 min)
- One sentence flow: **GitHub → CI gates → build image → scan → push → GitOps commit → Argo CD deploy → observe**
- Mention secrets: **Vault (no secrets in Git)** *(deep dive vertical)*

### 2) Low-level design + lab topology (1–2 min)
- Show node roles + IPs:
  - `dell-optiplex-1` (k3s server) `192.168.0.100`
  - `hp-prodesk-1` (agent) `192.168.0.101`
  - `hp-prodesk-2` (agent) `192.168.0.102`
- NAS provides NFS for persistent storage *(planned)*

### 3) “As code” platform bootstrap (Ansible + Helm) (1–2 min)
- Show Ansible repo paths:
  - `platform/ansible/inventory/hosts.ini`
  - `platform/ansible/playbooks/01-baseline.yml`
  - `platform/ansible/playbooks/03-k3s.yml`
- Say explicitly:
  - **Baseline doesn’t install k3s — it prepares nodes (swap off, sysctl/modules, packages).**
- Show Helm values for platform components:
  - `platform/helm/argocd-values.yaml` (Argo CD NodePort “as code”)

Quick proof platform is ready:
```bash
kubectl --kubeconfig ~/git/devops-final/kubeconfig/k3s.yaml get nodes -o wide
kubectl -n argocd get pods
kubectl -n argocd get svc argocd-server -o wide
