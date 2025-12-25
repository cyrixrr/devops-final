# Value Stream Map (VSM)

## Value: From code change to running safely in Kubernetes (with evidence)

### Stage 1) Code + PR
- Output: reviewable change
- Gate: PR rules (review, checks)

### Stage 2) CI Quality & Security (PR pipeline – planned/optional)
- gitleaks (hardcoded secrets)
- lint + unit tests
- semgrep (SAST)
- trivy (vulnerability scan)
- Output: green checks

### Stage 3) Build & Package (main pipeline)
- Build Docker image (immutable artifact)
- Tag with commit SHA
- Output: image

### Stage 4) Publish
- Push image to GHCR:
  - `ghcr.io/cyrixrr/myapp:<commit_sha>`
  - `ghcr.io/cyrixrr/myapp:latest`
- Output: published artifact

### Stage 5) GitOps Update (CD trigger)
- Pipeline updates GitOps manifests to the same commit SHA:
  - `image: ghcr.io/cyrixrr/myapp:<commit_sha>`
  - env: `APP_VERSION=<commit_sha>`, `GIT_SHA=<commit_sha>`
- Pipeline commits this to Git:
  - commit message like `gitops: deploy <sha>`
- Output: Git becomes the source of truth for desired state

### Stage 6) Deploy (Argo CD)
- Argo CD detects Git change and syncs to k3s
- Output: rollout completed, app Healthy

### Stage 7) Verify + Observe
- Verify externally:
  - `/health`
  - `/version` returns the running commit SHA (proof)
- (Planned) Prometheus/Grafana dashboards for metrics evidence

## Demo lead time targets
- CI job: a couple of minutes (depends on runner)
- GitOps sync + rollout: ~1 minute
