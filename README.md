# ScoutFlow GitOps Repository

GitOps deployment configurations for the ScoutFlow NBA player tracking application using ArgoCD.

## 📁 Repository Structure

```
scoutflow-gitops/
├── argocd/
│   └── apps/
│       ├── scoutflow-staging.yaml      # Staging environment (auto-sync)
│       └── scoutflow-production.yaml   # Production environment (manual sync)
└── environments/
    ├── staging/
    │   └── values.yaml                 # Staging config (latest tags, 1 replica)
    └── production/
        └── values.yaml                 # Production config (v1.0.0, 2 replicas)
```

## 🏗️ Architecture

### Three-Repository Model

| Repository | Purpose |
|------------|---------|
| [scoutflow-app](https://github.com/omerbh7/scoutflow-app) | Application code + Helm charts |
| [scoutflow-infra](https://github.com/omerbh7/scoutflow-infra) | Terraform infrastructure (EKS, VPC, ArgoCD) |
| **scoutflow-gitops** (this repo) | Deployment configurations |

### Deployment Flow

```
1. Code pushed to scoutflow-app
         ↓
2. CI builds and pushes Docker image to ECR
         ↓
3. ArgoCD detects change (watches this repo)
         ↓
4. ArgoCD syncs cluster to match Git state
         ↓
5. Application deployed! 🎉
```

## 🚀 Quick Start

### Prerequisites

- EKS cluster running (from `scoutflow-infra`)
- ArgoCD installed on cluster
- `kubectl` configured for cluster access
- ECR images available

### Deploy Staging Environment

```bash
# Apply ArgoCD Application
kubectl apply -f argocd/apps/scoutflow-staging.yaml

# Watch sync progress
kubectl get applications -n argocd

# Check pods
kubectl get pods -n staging
```

### Deploy Production Environment

```bash
# Apply ArgoCD Application
kubectl apply -f argocd/apps/scoutflow-production.yaml

# Sync is MANUAL - go to ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open: https://localhost:8080
# Login and click SYNC on scoutflow-production
```

## 🌍 Environments

| Environment | Namespace | Sync Policy | Image Tags | Replicas |
|-------------|-----------|-------------|------------|----------|
| Staging | `staging` | Automated | `latest` | 1 |
| Production | `production` | Manual | `v1.0.0` | 2 |

## 📚 Documentation

- [Deployment Guide](docs/deployment-guide.md) - Detailed setup instructions
- [Promotion Workflow](docs/promotion-workflow.md) - How to promote to production

## 🔐 ECR Authentication

ECR credentials expire every 12 hours. Create secret:

```bash
# Get ECR password
ECR_PASSWORD=$(aws ecr get-login-password --region us-east-1)

# Create secret in staging namespace
kubectl create secret generic ecr-credentials \
  --from-literal=password="$ECR_PASSWORD" \
  -n staging

# Create secret in production namespace
kubectl create secret generic ecr-credentials \
  --from-literal=password="$ECR_PASSWORD" \
  -n production
```

## 🛠️ Common Operations

### Update Staging (Automatic)

Staging auto-deploys when new `latest` images are pushed:

```bash
# In scoutflow-app repository
git push origin main

# CI builds and pushes latest tag
# ArgoCD auto-syncs staging (no action needed)
```

### Promote to Production (Manual)

```bash
# 1. Create release tag in scoutflow-app
cd ../scoutflow-app
git tag v1.1.0
git push origin v1.1.0

# 2. Update production values
cd ../scoutflow-gitops
vim environments/production/values.yaml
# Change imageTag: v1.0.0 → v1.1.0

# 3. Commit and push
git add environments/production/values.yaml
git commit -m "Release v1.1.0 to production"
git push origin main

# 4. Sync in ArgoCD UI (manual approval)
```

## 🔍 Troubleshooting

### Application Not Syncing

```bash
# Check application status
kubectl describe application scoutflow-staging -n argocd

# Check ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

### Image Pull Errors

```bash
# Verify ECR secret exists
kubectl get secret ecr-credentials -n staging

# Recreate if expired (12h TTL)
# See "ECR Authentication" section above
```

## 📊 Monitoring

### ArgoCD UI

```bash
# Port forward to ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Open: https://localhost:8080
```

## 🏷️ Tags

- `latest` - Auto-built from `main` branch (staging)
- `v1.x.x` - Release tags (production)

---

**Maintained by**: DevOps Team  
**Project**: ScoutFlow NBA Player Tracking