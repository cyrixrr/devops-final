## Current status (as of 2025-12-25)

### Lab nodes and roles (reserved IPs)
- **dell-optiplex-1** (Dell OptiPlex 3070) — **192.168.0.100** — planned **k3s server**
- **hp-prodesk-1** (HP ProDesk 600 G4) — **192.168.0.101** — planned **k3s agent**
- **hp-prodesk-2** (HP ProDesk 600 G4) — **192.168.0.102** — planned **k3s agent**
- **NAS (Zyxel 326)** — NFS storage (IP: TBD)

### Automation applied
- **Ansible baseline playbook applied on all nodes**: `platform/ansible/playbooks/01-baseline.yml`

What the baseline does:
- installs common packages (curl/jq/git/nfs-common/chrony/etc.)
- disables swap (required by Kubernetes)
- loads kernel modules (`overlay`, `br_netfilter`)
- applies sysctl networking settings for Kubernetes

Important:
- **This baseline does NOT install k3s.**
- It only prepares the nodes to be “Kubernetes-ready”. k3s installation is the next step.
