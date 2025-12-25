
---

## `docs/DEEP_DIVE_VAULT.md`
```markdown
# Deep Dive: Vault

Last updated: 2025-12-25

Status: planned (not implemented yet).  
Reason: current focus was building the k3s + GitOps platform slice first.

## Why Vault in this project
Goal: remove secrets from:
- Git
- CI logs
- Kubernetes manifests

Vault provides:
- central secrets management
- audit capabilities
- dynamic credentials (best demo value)

## Planned Vault features for the demo
1) Vault deployed in k3s (Helm)
2) Vault unsealed (manual for demo or auto-unseal optional)
3) Kubernetes auth enabled
4) Policies:
   - least privilege (app can read only its secrets)
5) Secret injection:
   - env vars or agent injector
6) Dynamic DB credentials:
   - Postgres secrets engine
   - TTL-based users (proves “security automation”)

## Deep dive demo outline (2–3 min)
- Show k8s ServiceAccount used by the app
- Show Vault role bound to SA + namespace
- Show policy allowing only `kv/data/myapp/*`
- Show app getting secret without storing it in Git
- Show dynamic DB credentials generation + TTL expiry

## Planned files (to be added later)
- `platform/helm/vault-values.yaml`
- `gitops/apps/vault/...` or Argo CD Application for Vault
- updates to `gitops/apps/myapp/...` to consume Vault secrets
