# DevOps Final Project — Home Lab GitOps Platform (Strapi + Postgres)

This repository demonstrates a complete **CI + CD GitOps** delivery loop in a **home lab** environment.

**What’s running in the lab right now:**
- **k3s Kubernetes (3 nodes)**
- **Argo CD** (GitOps deployments)
- **Strapi** (application) + **PostgreSQL** (database)
- **Prometheus + Grafana** (observability)
- **NFS dynamic provisioning** (PVCs via external provisioner)
- **Pi-hole DNS + Nginx Proxy Manager (NPM)** on a local **Proxmox** server for friendly hostnames + reverse proxy
- **Traefik** is used **only** as the k3s ServiceLB/LoadBalancer (no Ingress usage in this project)

---

## The “money shot” demo (real GitOps CD)

**Goal:** *No manual `kubectl apply` / no “deploy from CI”*.  
The **source of truth is Git**.

### End-to-end flow
1. **Change Strapi code** (in `strapi/`) and push to `main`
2. **GitHub Actions CI**:
   - builds Strapi
   - builds & pushes a container image to **GHCR**
   - uses **immutable image tag = commit SHA**
   - opens a **PR** that updates GitOps desired state in:
     - `gitops/apps/strapi/chart/values.yaml` → `image.tag: "<sha>"`
3. **Merge the PR**
4. **Argo CD auto-sync** detects Git change and rolls out Strapi in k3s
5. Verify rollout:
   - `kubectl -n strapi get pods`
   - `kubectl -n strapi describe pod <pod>` shows new image tag
6. Observability proof:
   - Grafana dashboard (CPU/Mem/Restarts/Replicas/Ready Pods)

---

## Access model (Pi-hole + NPM)

I don’t use Kubernetes Ingress for this project. Instead:

- **Pi-hole** provides internal DNS records (`*.apps.home.arpa`)
- **Nginx Proxy Manager (NPM)** runs on a local **Proxmox** server and reverse-proxies to k3s NodePorts
- Example hostnames:
  - `http://strapi.apps.home.arpa/admin`  → Strapi NodePort service
  - `https://argocd.apps.home.arpa` (optional) → Argo CD NodePort service

**Traefik** is the default k3s ServiceLB and exposes cluster services via:
- `192.168.0.100`, `192.168.0.101`, `192.168.0.102`

Traefik is not used for HTTP ingress routing here (NPM handles that externally).

---

## Architecture (high-level)

### Runtime components
- **k3s cluster** (1 server + 2 agents)
- **Argo CD**:
  - root app + app-of-apps structure under `gitops/argocd/`
- **Strapi app** deployed via **Helm chart in Git**
  - chart & values live under `gitops/apps/strapi/chart/`
- **PostgreSQL** deployed via Helm values under `gitops/helm/postgresql/`
- **Observability**
  - kube-prometheus-stack values under `gitops/helm/kube-prometheus-stack/`
  - Grafana dashboard for Strapi is delivered as a **ConfigMap** with label `grafana_dashboard=1`

### Storage
- Dynamic PV provisioning using `nfs-subdir-external-provisioner`
- Strapi PVCs:
  - `strapi-uploads`
- PostgreSQL PVC:
  - `data-strapi-postgresql-0`

---

## Lab topology

### Kubernetes nodes (reserved IPs)
- **dell-optiplex-1** — `192.168.0.100` — **k3s server (control-plane)**
- **hp-prodesk-1** — `192.168.0.101` — **k3s agent**
- **hp-prodesk-2** — `192.168.0.102` — **k3s agent**

### Control workstation
Used only for administration (VS Code, kubectl, helm, ansible, browser dashboards).

### DNS / Reverse proxy
- **Pi-hole** (DNS)
- **Nginx Proxy Manager (NPM)** on **Proxmox** (reverse proxy to NodePorts)

---

## Repository layout (current)

- `strapi/` — Strapi application source + Dockerfile
- `.github/workflows/` — CI pipelines (build/push + PR bump GitOps)
- `gitops/` — GitOps source of truth
  - `gitops/argocd/` — ArgoCD applications (root + apps)
  - `gitops/apps/strapi/chart/` — Strapi Helm chart + `values.yaml` (image.tag is updated via PR)
  - `gitops/helm/` — Helm values for platform components (postgresql, kube-prometheus-stack)
  - `gitops/monitoring/` — Kustomize app for Grafana dashboards (ConfigMaps)
- `platform/ansible/` — Ansible inventory + playbooks for provisioning the 3 nodes
- `platform/helm/` — values used to install ArgoCD (bootstrap)
- `docs/` — exam documentation:
  - `docs/HLD.md`
  - `docs/LLD.md`
  - `docs/VSM.md`
  - `docs/RUNBOOK.md`
  - `docs/DEMO_SCRIPT.md`

---

## Demo checklist (12–15 minutes)

Open these tabs:
- GitHub repo (PR list)
- GitHub Actions run
- Argo CD UI (applications page)
- Strapi admin URL
- Grafana dashboard

Suggested order:
1. **HLD**: components + why GitOps (1–2 min)
2. **LLD**: repo layout + where desired state lives (1–2 min)
3. **CI**: show action builds/pushes image + opens PR bump (2–3 min)
4. **Money shot**:
   - open PR “bump Strapi image tag to `<sha>`”
   - confirm it changes only `gitops/apps/strapi/chart/values.yaml`
   - merge PR
5. **ArgoCD** auto-sync + rollout (1–2 min)
6. **kubectl proof**: pod image tag is new SHA (1 min)
7. **Grafana proof**: Strapi dashboard (1–2 min)
8. **Future improvements** (30–60 sec)

---

## Notes / Future improvements (easy to discuss in Q&A)

- Add **Strapi app-level metrics** (`/metrics`) + ServiceMonitor (real RPS/latency/error rate)
- Add **SAST/secret scanning** gates for the Strapi repo path
- Add **image signing** (cosign) + policy enforcement
- Add **Vault** for dynamic secrets (optional deep dive topic)

---

## Quick commands

### Show apps (ArgoCD)
```bash
kubectl -n argocd get applications
