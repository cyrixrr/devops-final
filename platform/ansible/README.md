
# Platform Provisioning (Ansible) — 3-node k3s lab

This folder documents how the **3 Kubernetes nodes** (k3s) are provisioned using Ansible.
Even if your environment is already built, this is valuable for the exam because it proves
**configuration management** and “as code” infrastructure practices.

## Structure (matches your repo)

- Inventory: `platform/ansible/inventory/hosts.ini`
- Playbooks:
  - `platform/ansible/playbooks/01-baseline.yml` — base OS setup (users, packages, sysctl, time, etc.)
  - `platform/ansible/playbooks/02-nfs.yml` — NFS client/server pieces used for dynamic PV provisioning (if applicable)
  - `platform/ansible/playbooks/03-k3s.yml` — installs/configures k3s (server + agents)
- Kubeconfig output (optional path): `platform/ansible/playbooks/kubeconfig`

## Prerequisites

On your control machine (laptop/workstation):
- Ansible installed
- SSH access to the three nodes (passwordless recommended)

Example:
```bash
sudo apt-get update
sudo apt-get install -y ansible
```

## Inventory example

Open: `platform/ansible/inventory/hosts.ini` and ensure it defines:
- 1 server (control-plane)
- 2 agents (workers)

Typical grouping:
- `[k3s_server]`
- `[k3s_agents]`
- `[k3s_cluster:children]`

## Run order (idempotent)

From repo root:

```bash
cd platform/ansible

# 1) Baseline OS
ansible-playbook -i inventory/hosts.ini playbooks/01-baseline.yml

# 2) NFS prerequisites (if your playbook configures it)
ansible-playbook -i inventory/hosts.ini playbooks/02-nfs.yml

# 3) Install / configure k3s cluster
ansible-playbook -i inventory/hosts.ini playbooks/03-k3s.yml
```

## Getting kubeconfig

Your playbooks may already copy kubeconfig into `playbooks/kubeconfig`.
If so, export it:

```bash
export KUBECONFIG=$PWD/playbooks/kubeconfig
kubectl get nodes -o wide
```

## Exam talking points

- “I can reprovision the cluster from scratch with Ansible playbooks.”
- “This reduces drift, improves repeatability, and supports immutable infrastructure practices.”
- “GitOps then manages the apps on top of the cluster.”

## Access notes

## Notes about access in this lab

Provisioning brings up the k3s nodes, but human-friendly access is provided by:
- **Pi-hole** for DNS (`*.apps.home.arpa`)
- **Nginx Proxy Manager** for reverse-proxy to NodePorts

Traefik remains the k3s **Service LoadBalancer** (IPs: `192.168.0.100/101/102`).
