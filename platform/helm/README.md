
# Platform Helm Notes

This repo mixes:
- Helm charts/values for install-time components (monitoring, postgres, etc.)
- Argo CD applications that apply those charts declaratively (GitOps)

## What you have in this repo

- `platform/helm/argocd-values.yaml`
  - values used to install/upgrade Argo CD (if you bootstrap Argo via Helm)
- `gitops/helm/kube-prometheus-stack/values.yaml`
  - values for kube-prometheus-stack (Prometheus + Grafana)
- `gitops/helm/postgresql/strapi-values.yaml`
  - values for Postgres used by Strapi
- `gitops/apps/strapi/chart/`
  - Helm chart that deploys Strapi + its K8s objects

## Typical bootstrap (if you ever need to rebuild)

1) Install Argo CD (one-time bootstrap)
```bash
kubectl create ns argocd --dry-run=client -o yaml | kubectl apply -f -

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  -n argocd \
  -f platform/helm/argocd-values.yaml
```

2) Apply root app (apps-of-apps)
```bash
kubectl -n argocd apply -f gitops/argocd/root-app.yaml
```

From here on: **Argo manages everything** from Git.

## Exam talking points

- “Bootstrap is Helm once, then everything else is GitOps (Argo).”
- “Strapi deploy is controlled by Git change (values.yaml image.tag).”