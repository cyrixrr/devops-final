
---

## `docs/VSM.md`
```markdown
# Value Stream Map (VSM)

Last updated: 2025-12-25

## Value: From code change to running safely in Kubernetes with evidence

### Current working state (today)
GitOps CD path is already working:
- Git → Argo CD → k3s deploy → app reachable

### Target end state (final exam)
GitHub PR → CI gates → build image → scan → push → GitOps commit → Argo CD deploy → Vault secrets → metrics.

---

## Stages

1) Code + PR
- Output: reviewable change
- Gate: PR rules (review + checks)

2) CI Security & Quality (PR pipeline) *(planned next)*
- gitleaks (secrets)
- lint + unit tests
- semgrep (SAST)
- Output: green checks

3) Build & Package (main pipeline) *(planned next)*
- Build Docker image (immutable artifact)
- Tag with git SHA / build number
- Output: image

4) Publish *(planned next)*
- Push image to registry
- Output: published artifact

5) GitOps Update (CD trigger) *(planned next)*
- Pipeline updates GitOps overlay with new image tag and commits
- Output: Git becomes the source of truth for desired state

6) Deploy (Argo CD) *(DONE today)*
- Argo syncs desired state to k3s
- Output: rollout completed

7) Verify + Observe *(partially done; metrics planned)*
- Smoke test: app reachable
- Later: Grafana dashboards and alerts
- Output: evidence of success

---

## Demo lead time targets (final)
- PR checks: < 2 minutes
- Main pipeline → deployed: < 3 minutes

## What is already proven in the lab (today)
- Argo CD pulls from the repo and applies Kustomize overlay
- GitOps deployment works end-to-end: Git → Argo CD → k3s → running Pod/Service
- Service exposure can be controlled as code via overlay patches (NodePort)
