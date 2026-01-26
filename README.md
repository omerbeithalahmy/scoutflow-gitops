# ScoutFlow GitOps Repository

GitOps deployment configurations for the ScoutFlow NBA player tracking application using ArgoCD.

## 📁 Repository Structure

```
scoutflow-gitops/
├── argocd/
│   └── apps/
│       ├── scoutflow-dev.yaml         # Dev environment (auto-sync)
│       ├── scoutflow-stage.yaml       # Stage environment (auto-sync)
│       └── scoutflow-prod.yaml        # Production environment (manual sync)
└── environments/
    ├── dev/
    │   └── values.yaml                # Dev config (latest tags, 1 replica)
    ├── stage/
    │   └── values.yaml                # Stage config (latest tags, 2 replicas)
    └── prod/
        └── values.yaml                # Prod config (v1.0.0, 3 replicas)
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

## 🌍 Environments

| Environment | Namespace | Sync Policy | Image Tags | Replicas | Purpose |
|-------------|-----------|-------------|------------|----------|---------|
| **Dev** | `dev` | Automated | `latest` | 1 | Development & feature testing |
| **Stage** | `stage` | Automated | `latest` | 2 | Pre-production QA & testing |
| **Prod** | `prod` | Manual | `v1.0.0` | 3 | Production workloads |

### Resource Allocation

| Environment | Backend CPU | Backend RAM | Frontend CPU | Frontend RAM |
|-------------|-------------|-------------|--------------|--------------|
| **Dev** | 100m-500m | 128Mi-256Mi | 50m-200m | 64Mi-128Mi |
| **Stage** | 100m-500m | 128Mi-256Mi | 50m-200m | 64Mi-128Mi |
| **Prod** | 100m-500m | 128Mi-256Mi | 50m-200m | 64Mi-128Mi |

## 🚀 Quick Start

### Prerequisites

- EKS cluster running (from `scoutflow-infra`)
- ArgoCD installed on cluster
- `kubectl` configured for cluster access
- ECR images available
- Database credentials from Terraform

### 1. Setup Database Credentials

For each environment, retrieve the database password from Terraform:

```bash
# Dev environment
cd ~/scoutflow-infra/environments/dev
export DEV_DB_PASSWORD=$(terraform output -raw db_password)

# Stage environment
cd ~/scoutflow-infra/environments/stage
export STAGE_DB_PASSWORD=$(terraform output -raw db_password)

# Prod environment
cd ~/scoutflow-infra/environments/prod
export PROD_DB_PASSWORD=$(terraform output -raw db_password)
```

Update the values files:

```bash
cd ~/scoutflow-gitops

# Update dev
sed -i '' "s/POSTGRES_PASSWORD: <REPLACE_WITH_TERRAFORM_OUTPUT>/POSTGRES_PASSWORD: $DEV_DB_PASSWORD/" environments/dev/values.yaml

# Update stage
sed -i '' "s/POSTGRES_PASSWORD: <REPLACE_WITH_TERRAFORM_OUTPUT>/POSTGRES_PASSWORD: $STAGE_DB_PASSWORD/" environments/stage/values.yaml

# Update prod
sed -i '' "s/POSTGRES_PASSWORD: <REPLACE_WITH_TERRAFORM_OUTPUT>/POSTGRES_PASSWORD: $PROD_DB_PASSWORD/" environments/prod/values.yaml

# Commit changes
git add environments/*/values.yaml
git commit -m "Configure database credentials"
git push origin main
```

### 2. Deploy Environments

```bash
# Deploy dev (auto-syncs)
kubectl apply -f argocd/apps/scoutflow-dev.yaml

# Deploy stage (auto-syncs)
kubectl apply -f argocd/apps/scoutflow-stage.yaml

# Deploy prod (requires manual sync in ArgoCD UI)
kubectl apply -f argocd/apps/scoutflow-prod.yaml

# Watch sync progress
kubectl get applications -n argocd

# Check pods in each environment
kubectl get pods -n dev
kubectl get pods -n stage
kubectl get pods -n prod
```

### 3. Sync Production (Manual)

```bash
# Port forward to ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Open: https://localhost:8080
# Login and click SYNC on scoutflow-prod
```

## 🔄 Deployment Workflow

### Development → Stage → Production

```
┌─────────────────────────────────────────────────────────┐
│ 1. Development                                          │
│    • Push to main branch                                │
│    • CI builds 'latest' tag                             │
│    • ArgoCD auto-deploys to dev                         │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 2. Stage Testing                                        │
│    • Same 'latest' images deployed to stage             │
│    • QA team tests in production-like environment       │
│    • ArgoCD auto-syncs stage                            │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 3. Production Release                                   │
│    • Create version tag: git tag v1.1.0                 │
│    • CI builds versioned images                         │
│    • Update prod values.yaml with new version           │
│    • Manual sync in ArgoCD UI                           │
└─────────────────────────────────────────────────────────┘
```

## 🔐 ECR Authentication

ECR credentials expire every 12 hours. Create secret:

```bash
# Get ECR password
ECR_PASSWORD=$(aws ecr get-login-password --region us-east-1)

# Create secret in all namespaces
for ns in dev stage prod; do
  kubectl create secret docker-registry ecr-registry-secret \
    --docker-server=279987127424.dkr.ecr.us-east-1.amazonaws.com \
    --docker-username=AWS \
    --docker-password="$ECR_PASSWORD" \
    -n $ns
done
```

## 🛠️ Common Operations

### Update Dev/Stage (Automatic)

Both dev and stage auto-deploy when new `latest` images are pushed:

```bash
# In scoutflow-app repository
git push origin main

# CI builds and pushes latest tag
# ArgoCD auto-syncs dev and stage (no action needed)
```

### Promote to Production (Manual)

```bash
# 1. Create release tag in scoutflow-app
cd ../scoutflow-app
git tag v1.1.0
git push origin v1.1.0

# 2. Wait for CI to build versioned images

# 3. Update production values
cd ../scoutflow-gitops
vim environments/prod/values.yaml
# Change imageTag: v1.0.0 → v1.1.0

# 4. Commit and push
git add environments/prod/values.yaml
git commit -m "Release v1.1.0 to production"
git push origin main

# 5. Sync in ArgoCD UI (manual approval)
```

## 🔍 Troubleshooting

### Application Not Syncing

```bash
# Check application status
kubectl describe application scoutflow-dev -n argocd
kubectl describe application scoutflow-stage -n argocd
kubectl describe application scoutflow-prod -n argocd

# Check ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

### Image Pull Errors

```bash
# Verify ECR secret exists
kubectl get secret ecr-registry-secret -n dev
kubectl get secret ecr-registry-secret -n stage
kubectl get secret ecr-registry-secret -n prod

# Recreate if expired (12h TTL)
# See "ECR Authentication" section above
```

### Database Connection Issues

```bash
# Check database pod status
kubectl get pods -n dev -l app=database
kubectl get pods -n stage -l app=database
kubectl get pods -n prod -l app=database

# Check database credentials
kubectl get secret -n dev
# Verify POSTGRES_PASSWORD is set correctly in values.yaml
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

### Application Health

```bash
# Check all applications
kubectl get applications -n argocd

# Watch specific environment
kubectl get pods -n dev -w
kubectl get pods -n stage -w
kubectl get pods -n prod -w
```

## 🏷️ Tags

- `latest` - Auto-built from `main` branch (dev + stage)
- `v1.x.x` - Release tags (production)

---

**Maintained by**: DevOps Team  
**Project**: ScoutFlow NBA Player Tracking