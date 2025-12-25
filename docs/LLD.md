# Low-Level Design (LLD)

Last updated: 2025-12-25

## 1) Nodes and roles

| Hostname           | Model               | Role                      | IP            |
|-------------------|---------------------|---------------------------|---------------|
| dell-optiplex-1    | Dell OptiPlex 3070   | k3s server (control-plane)| 192.168.0.100 |
| hp-prodesk-1       | HP ProDesk 600 G4    | k3s agent                  | 192.168.0.101 |
| hp-prodesk-2       | HP ProDesk 600 G4    | k3s agent                  | 192.168.0.102 |

## 2) Bootstrap automation (Ansible)

Inventory:
- `platform/ansible/inventory/hosts.ini`

Baseline playbook:
- `platform/ansible/playbooks/01-baseline.yml`
- Purpose:
  - update apt cache / install baseline packages
  - enable chrony
  - disable swap (k8s requirement)
  - load kernel modules
  - sysctl networking settings for Kubernetes
- **This does NOT install k3s**. It prepares nodes to be Kubernetes-ready.

k3s install playbook:
- `platform/ansible/playbooks/03-k3s.yml`
- Notes:
  - installs pinned version `v1.30.6+k3s1`
  - server uses fixed node IP + TLS SAN: `192.168.0.100`
  - reads join token from server and installs agents
  - fetches kubeconfig from server to the control workstation

Kubeconfig output:
- `~/git/devops-final/kubeconfig/k3s.yaml`
- Must point to: `https://192.168.0.100:6443`

## 3) GitOps (Argo CD + Kustomize)

Argo CD installed in:
- namespace `argocd`

Argo CD installation method:
- Helm chart `argo/argo-cd`

Argo CD access:
- Service type set to `NodePort` via Helm values:
  - `platform/helm/argocd-values.yaml`
- Validate ports:
  - `kubectl -n argocd get svc argocd-server -o wide`

GitOps repository:
- this repo itself: `https://github.com/cyrixrr/devops-final.git`

Argo CD Application manifest:
- `gitops/apps/myapp/argocd-app.yaml`
- Sync policy:
  - automated sync + prune + self-heal
  - CreateNamespace option

Kustomize structure:
- Base: `gitops/apps/myapp/base`
- Overlay: `gitops/apps/myapp/overlays/dev`
- Overlay applies:
  - namespace: `myapp`
  - namePrefix: `dev-`
  - (and Service type NodePort in rendered output)

## 4) Application

App source:
- `app/main.py`

App container build:
- `app/Dockerfile`

Endpoints:
- `/health` → simple health check
- `/version` → prints `APP_VERSION` and `GIT_SHA`

Kubernetes service mapping:
- service port 80 → container port 8000

## 5) CI/CD pipelines (GitHub Actions)

Main pipeline:
- `.github/workflows/main.yml`

What it does on push to `main`:
1. run unit tests (pytest)
2. build image from `app/`
3. push to GHCR:
   - `ghcr.io/cyrixrr/myapp:<commit_sha>`
   - `ghcr.io/cyrixrr/myapp:latest`
4. update GitOps deployment image tag to `<commit_sha>`
5. set env vars `APP_VERSION` and `GIT_SHA` to `<commit_sha>`
6. commit and push the GitOps change

Important requirement:
- GHCR package must be **Public** (or use imagePullSecret).
