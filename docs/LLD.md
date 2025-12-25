# Low-Level Design (LLD)

Last updated: 2025-12-25

## 1) Hosts, roles, IPs
| Hostname        | Model              | Role                       | IP            |
|----------------|--------------------|----------------------------|---------------|
| dell-optiplex-1 | Dell OptiPlex 3070 | k3s server (control-plane) | 192.168.0.100 |
| hp-prodesk-1    | HP ProDesk 600 G4   | k3s agent                  | 192.168.0.101 |
| hp-prodesk-2    | HP ProDesk 600 G4   | k3s agent                  | 192.168.0.102 |

Router DHCP reservations keep IPs stable.

## 2) Automation applied (what is already done)

- **Ansible baseline playbook applied on all nodes**: `platform/ansible/playbooks/01-baseline.yml`  
  Purpose: prepare hosts for Kubernetes (packages, time sync, swap off, kernel modules, sysctl).

- **k3s installed via Ansible**: `platform/ansible/playbooks/03-k3s.yml`  
  - Version pinned: `v1.30.6+k3s1`
  - Server: `dell-optiplex-1` (192.168.0.100)
  - Agents: `hp-prodesk-1` (192.168.0.101), `hp-prodesk-2` (192.168.0.102)

- **kubeconfig fetched to control workstation**  
  Path: `~/git/devops-final/kubeconfig/k3s.yaml`  
  (Fetched from `/etc/rancher/k3s/k3s.yaml` and rewritten to point at `192.168.0.100`)

- **Argo CD installed via Helm**  
  Chart: `argo/argo-cd`  
  Values file: `platform/helm/argocd-values.yaml`  
  Access: NodePort (HTTPS preferred)

- **First GitOps application deployed**: `myapp`  
  - Argo Application manifest: `gitops/apps/myapp/argocd-app.yaml`
  - Kustomize structure: `gitops/apps/myapp/base` + `gitops/apps/myapp/overlays/dev`
  - Service exposure: NodePort managed via overlay patch

## 3) k3s details
- Server API: `https://192.168.0.100:6443`
- Kubeconfig location (control workstation): `~/git/devops-final/kubeconfig/k3s.yaml`

Verify:
```bash
kubectl --kubeconfig ~/git/devops-final/kubeconfig/k3s.yaml get nodes -o wide
kubectl --kubeconfig ~/git/devops-final/kubeconfig/k3s.yaml get pods -A
