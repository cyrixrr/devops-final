# Demo Script (Draft)

## Tabs open (prepare in advance)
- GitHub repo (PR view)
- GitHub Actions (latest runs)
- Argo CD (Application view)
- App `/version` endpoint (browser tab or curl)
- Grafana dashboard (or Prometheus targets page)
- (Optional) Vault UI/CLI tab for deep dive

## Preset environment rule
- k3s cluster is already running
- Argo CD already connected to GitOps repo
- Vault already unsealed and configured
- Monitoring already running (Prometheus/Grafana)
- Pipeline caches warmed (avoid long waits)

---

## Order (12–15 minutes)

### 1) High-level solution design (1–2 min)
- One sentence flow: **GitHub → CI gates → build image → scan → push → GitOps commit → Argo CD deploy → observe**
- Mention secrets: **Vault (no secrets in Git)**

### 2) Low-level design + lab topology (1–2 min)
- Show node roles + IPs:
  - `dell-optiplex-1` (k3s server) `192.168.0.100`
  - `hp-prodesk-1` (agent) `192.168.0.101`
  - `hp-prodesk-2` (agent) `192.168.0.102`
- NAS provides NFS for persistent storage (if used)

### 3) “As code” platform bootstrap (Ansible) (1 min)
- Show repo paths:
  - `platform/ansible/inventory/hosts.ini`
  - `platform/ansible/playbooks/01-baseline.yml`
- Say explicitly:
  - **Baseline doesn’t install k3s — it prepares nodes (swap off, sysctl/modules, packages).**

### 4) Repo layout (30–60 sec)
- `app/`, `db/`, `gitops/`, `.github/workflows/`, `docs/`

### 5) PR pipeline (CI gates) (3–4 min)
- Open an example PR (already created)
- Show CI checks:
  - gitleaks (secrets)
  - lint + tests
  - SAST (semgrep)
  - image/vuln scan (trivy) *(if part of PR pipeline)*

### 6) Merge to main → release pipeline (3–4 min)
- Show the completed run (preset) or trigger a merge if fast:
  - build Docker image (immutable)
  - scan image
  - push to registry
  - update GitOps overlay (commit/tag)

### 7) GitOps deploy (Argo CD) (1–2 min)
- Argo CD detects change, syncs
- Show rollout status / healthy state

### 8) Validate app + observability (1–2 min)
- Hit `/version` (shows new tag/commit)
- Show Grafana (request rate / error rate / latency) or Prometheus target health

### 9) Deep dive (Vault) (2–3 min)
- k8s auth method → service account → Vault role/policy
- show secret injection or dynamic DB credentials (TTL)

### 10) Future improvements (30–60 sec)
- Progressive delivery (Argo Rollouts)
- Policy as code (OPA/Gatekeeper or Kyverno)
- Chaos test (Litmus or simple pod kill) + alerting
- SBOM + provenance (cosign/slsa)
