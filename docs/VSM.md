# Value Stream Map (VSM)

## Value: From code change to running safely in Kubernetes

Stages:
1) Code + PR
   - Output: reviewable change
   - Gate: PR rules (review, checks)

2) CI Security & Quality (PR pipeline)
   - gitleaks (hardcoded secrets)
   - lint + unit tests
   - semgrep (SAST)
   - trivy (vulnerability scan)
   - Output: green checks

3) Build & Package (main pipeline)
   - Build Docker image (immutable artifact)
   - Tag with git sha / build number
   - Output: image

4) Publish
   - Push image to local registry
   - Output: published artifact

5) GitOps Update (CD trigger)
   - Pipeline updates GitOps overlay with new image tag and commits
   - Output: Git becomes the source of truth for desired state

6) Deploy (Argo CD)
   - Argo syncs desired state to k3s
   - Output: rollout completed

7) Verify + Observe
   - smoke test (/health)
   - check Grafana metrics
   - Output: evidence of success

## Demo lead time targets
- PR checks: < 2 minutes
- Main pipeline → deployed: < 3 minutes
