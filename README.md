# ScoutFlow GitOps (Operations)

**Purpose**: Kubernetes manifests and GitHub Actions for ArgoCD deployments.

## Environments
- dev/staging/prod folders with Helm values or Kustomize

## CI/CD Flow
- Feature branches → CI tests + ECR push
- main merge → update dev manifests
- Tags → update prod manifests

See docs/gitops-flow.md for details.