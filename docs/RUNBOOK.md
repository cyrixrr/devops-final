# Runbook

Last updated: 2025-12-25

## 1) Lab inventory (hostnames, roles, reserved IPs)

Final reserved IP mapping (router DHCP reservations):

| Hostname           | Model               | Role (today) | IP            |
|-------------------|---------------------|--------------|---------------|
| dell-optiplex-1    | Dell OptiPlex 3070   | k3s server   | 192.168.0.100 |
| hp-prodesk-1       | HP ProDesk 600 G4    | k3s agent    | 192.168.0.101 |
| hp-prodesk-2       | HP ProDesk 600 G4    | k3s agent    | 192.168.0.102 |
| NAS                | Zyxel 326            | NFS storage  | (fill in)     |

Notes:
- Each node has Ubuntu Server installed on **one** NVMe.
- Second NVMe on HP nodes is unused for now (optional later: `/data`).

## 2) SSH access (control workstation → nodes)

From main PC:
```bash
ssh stefanl@192.168.0.100 "hostname"
ssh stefanl@192.168.0.101 "hostname"
ssh stefanl@192.168.0.102 "hostname"

### Baseline vs k3s
`01-baseline.yml` prepares the nodes for Kubernetes:
- packages, time sync, swap off, kernel modules, sysctl
**It doesn’t install k3s yet — it just makes the machines “Kubernetes-ready”.**

