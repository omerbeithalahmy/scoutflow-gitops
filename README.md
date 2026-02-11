# ScoutFlow GitOps Repository

> **GitOps deployment configuration for the ScoutFlow NBA analytics platform**

Production-grade Kubernetes deployment using ArgoCD, Helm, and External Secrets Operator for secure secret management.

---

## 📋 Overview

This repository manages ScoutFlow's Kubernetes deployments across three environments using **GitOps principles**:

- **Development** - Automated deployment with latest images
- **Staging** - QA testing with production-like configuration
- **Production** - Manual approval for controlled releases

**Key Technologies:**
- ✅ ArgoCD (GitOps continuous delivery)
- ✅ Helm (Kubernetes package management)
- ✅ External Secrets Operator (AWS Secrets Manager integration)
- ✅ Multi-environment configuration

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│  GitHub Repository (scoutflow-gitops)               │
│  └─ environments/                                   │
│     ├─ dev/values.yaml                              │
│     ├─ stage/values.yaml                            │
│     └─ prod/values.yaml                             │
└─────────────────────────────────────────────────────┘
                        │
                        ↓ (ArgoCD watches)
┌─────────────────────────────────────────────────────┐
│  Kubernetes Cluster (AWS EKS)                       │
│                                                     │
│  ArgoCD ──→ Deploys ──→ ScoutFlow Application      │
│                                                     │
│  External Secrets Operator ──→ AWS Secrets Manager │
│              ↓                                      │
│         Database Credentials                        │
└─────────────────────────────────────────────────────┘
```

**Secret Management Flow:**
1. Terraform creates secrets in AWS Secrets Manager
2. External Secrets Operator syncs to Kubernetes
3. Application pods consume secrets securely
4. **No passwords stored in Git** ✅

---

## 📁 Repository Structure

```
scoutflow-gitops/
├── .github/
│   └── workflows/
│       └── pr-validation.yml     # CI/CD validation workflow
│
├── argocd/
│   └── apps/
│       ├── scoutflow-dev.yaml      # Dev environment ArgoCD application
│       ├── scoutflow-stage.yaml    # Stage environment ArgoCD application
│       └── scoutflow-prod.yaml     # Prod environment ArgoCD application
│
└── environments/
    ├── dev/values.yaml             # Dev configuration (latest images, minimal resources)
    ├── stage/values.yaml           # Stage configuration (2 replicas, testing)
    └── prod/values.yaml            # Prod configuration (3 replicas, versioned images)
```

---

## 🚀 Quick Start

### Prerequisites

1. **Infrastructure deployed** - [`scoutflow-infra`](https://github.com/omerbh7/scoutflow-infra) terraform applied
2. **ArgoCD installed** - Running in the EKS cluster
3. **External Secrets Operator** - Installed via infra repo
4. **Application repository** - [`scoutflow-app`](https://github.com/omerbh7/scoutflow-app) available on GitHub

### Deploy Applications

```bash
# 1. Deploy dev environment
kubectl apply -f argocd/apps/scoutflow-dev.yaml

# 2. Deploy stage environment
kubectl apply -f argocd/apps/scoutflow-stage.yaml

# 3. Deploy prod environment (manual sync required)
kubectl apply -f argocd/apps/scoutflow-prod.yaml
```

### Verify Deployment

```bash
# Check ArgoCD applications
kubectl get applications -n argocd

# Check application status
kubectl get pods -n dev
kubectl get pods -n stage
kubectl get pods -n prod

# Verify External Secrets
kubectl get externalsecret -n dev
kubectl describe externalsecret -n dev
```

---

## 🤖 Automatic Image Tracking

**Automatic deployment updates for dev and stage environments.**

### How It Works

When you push code to the `scoutflow-app` repository:

1. CI builds Docker images tagged with the commit SHA (e.g., `backend:abc123def`)
2. Automated workflow updates this GitOps repo with the new image tag
3. Changes `dev/values.yaml` and `stage/values.yaml` to reference the new SHA
4. ArgoCD detects the Git change and automatically syncs
5. New images deploy to dev and stage clusters

### Setup Required

Create a GitHub Personal Access Token (classic) with `repo` scope and add it as `GITOPS_PAT` secret in the scoutflow-app repository.

**Production remains manual** for safety - you control when production updates happen.

---

## 🔄 CI/CD Pipeline

<details>
<summary><b>📖 Automated Validation Workflow (Click to expand)</b></summary>

### PR Validation

Runs automatically on every Pull Request and push to main:

**Validation Steps:**

1. **YAML Lint** - Checks code style and formatting (warnings only)
2. **YAML Syntax Validation** - Ensures all YAML files are parseable (fails on errors)
3. **Security Scan** - Scans for security misconfigurations using Checkov (warnings only)

**Workflow:** [pr-validation.yml](.github/workflows/pr-validation.yml)

### How It Works

```
1. Developer creates PR
   ↓
2. GitHub Actions runs validation
   ↓
3. YAML syntax check must pass
   ↓
4. PR can be merged
   ↓
5. ArgoCD automatically deploys changes
```

### Benefits

- Catches YAML syntax errors before merge
- Identifies security misconfigurations
- Maintains code quality standards
- Prevents broken configurations from reaching cluster

</details>

---

## 🔐 Secret Management

<details>
<summary><b>📖 How External Secrets Work (Click to expand)</b></summary>

### Configuration

In values files:
```yaml
externalSecrets:
  enabled: true
  region: us-east-1
  secretName: "scoutflow/dev/database"
```

### What Happens

1. External Secrets Operator reads from AWS Secrets Manager
2. Creates Kubernetes secret: `<release>-db-secret`
3. Contains keys: `DB_USER`, `DB_PASSWORD`, `DB_NAME`
4. Application pods mount the secret

### Security Benefits

- ✅ Zero passwords in Git
- ✅ IAM-based authentication (IRSA)
- ✅ Automatic secret rotation support
- ✅ Audit trail in AWS CloudTrail

</details>

---

## 🌍 Environment Configuration

<details>
<summary><b>📖 Detailed Environment Specs (Click to expand)</b></summary>

### Development

**Purpose:** Feature development and testing

- **Images:** `latest` tag (auto-deploys on push)
- **Replicas:** 1 per service
- **Resources:** Minimal (cost-optimized)
- **Sync:** Automated with self-healing
- **Namespace:** `dev`

### Staging

**Purpose:** QA testing and pre-production validation

- **Images:** `latest` tag
- **Replicas:** 2 per service (HA testing)
- **Resources:** Production-like
- **Sync:** Automated
- **Namespace:** `stage`

### Production

**Purpose:** Live user traffic

- **Images:** Versioned tags (e.g., `v1.0.0`)
- **Replicas:** 3 per service (high availability)
- **Resources:** High limits
- **Sync:** **Manual approval required**
- **Namespace:** `prod`

</details>

---

## 🔄 Deployment Workflow

<details>
<summary><b>📖 Deployment Process Details (Click to expand)</b></summary>

### Automated Deployment (Dev/Stage)

```
1. Developer pushes code to [`scoutflow-app`](https://github.com/omerbh7/scoutflow-app)
   ↓
2. GitHub Actions builds & pushes Docker images
   ↓
3. ArgoCD detects change in GitOps repo
   ↓
4. ArgoCD syncs application to cluster
   ↓
5. Pods restart with new images
```

### Manual Deployment (Production)

```
1. Update prod/values.yaml with new version
   ↓
2. Commit and push changes
   ↓
3. ArgoCD detects out-of-sync state
   ↓
4. Manual sync via ArgoCD UI or CLI:
   argocd app sync scoutflow-prod
   ↓
5. Production deployment completes
```

</details>

---

## 🛠️ Common Operations

<details>
<summary><b>📖 Useful Commands (Click to expand)</b></summary>

### Update Application Version

```bash
# 1. Edit the values file
vi environments/prod/values.yaml

# 2. Change imageTag
global:
  imageTag: v1.1.0  # Change this line

# 3. Commit and push
git add environments/prod/values.yaml
git commit -m "chore: update prod to v1.1.0"
git push origin main

# 4. Sync in ArgoCD
argocd app sync scoutflow-prod
```

### Scale Replicas

```bash
# Edit values file
vi environments/stage/values.yaml

# Update replicas
backend:
  replicas: 3  # Changed from 2

# Push changes
git add environments/stage/values.yaml
git commit -m "scale: increase stage backend to 3 replicas"
git push
```

### Check Application Health

```bash
# ArgoCD application status
argocd app get scoutflow-dev

# Detailed sync status
argocd app sync-status scoutflow-dev

# View application in UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open: https://localhost:8080
```

### Troubleshoot Sync Issues

```bash
# Check ArgoCD application
kubectl describe application scoutflow-dev -n argocd

# Check External Secrets
kubectl get externalsecret -n dev
kubectl describe externalsecret -n dev

# Check if secret was created
kubectl get secret -n dev | grep db-secret

# View External Secrets Operator logs
kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets
```

</details>

---

## 📊 Resource Allocation

<details>
<summary><b>📖 Resource Limits by Environment (Click to expand)</b></summary>

| Environment | Backend CPU | Backend Memory | Frontend CPU | Frontend Memory |
|-------------|-------------|----------------|--------------|-----------------|
| **Dev**     | 100m-500m   | 128Mi-256Mi    | 50m-200m     | 64Mi-128Mi      |
| **Stage**   | 200m-1000m  | 256Mi-512Mi    | 100m-500m    | 128Mi-256Mi     |
| **Prod**    | 500m-2000m  | 512Mi-1Gi      | 200m-1000m   | 256Mi-512Mi     |

</details>

---

## 🔍 Monitoring & Access

<details>
<summary><b>📖 Access and Monitoring Commands (Click to expand)</b></summary>

### ArgoCD UI

```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access: https://localhost:8080
# Username: admin
# Password: (from above)
```

### Application Logs

```bash
# Backend logs
kubectl logs -n dev -l app.kubernetes.io/component=backend

# Frontend logs
kubectl logs -n dev -l app.kubernetes.io/component=frontend

# Database logs
kubectl logs -n dev -l app.kubernetes.io/component=database
```

</details>

---

## 🔗 Integration with Other Repositories

### [scoutflow-infra](https://github.com/omerbh7/scoutflow-infra)

**Provides:**
- EKS cluster
- ArgoCD installation
- External Secrets Operator
- AWS Secrets Manager secrets
- IAM roles and policies

**Required before deploying this repo**

### [scoutflow-app](https://github.com/omerbh7/scoutflow-app)

**Provides:**
- Helm chart templates
- Docker images
- Application code

**Referenced by ArgoCD applications**

---

## 🚨 Important Notes

### Production Deployment

⚠️ **Production requires manual sync** - This is by design for controlled releases

```bash
# Never push directly to prod without review
# Always sync manually via ArgoCD UI or CLI
argocd app sync scoutflow-prod
```

### Secret Updates

🔒 **Secrets are managed by infrastructure repo**

- Database passwords: AWS Secrets Manager
- ECR credentials: Set via CI/CD pipeline
- Never commit passwords to this repository

### Image Tags

📌 **Automatic tracking with commit SHAs**

- **Dev/Stage:** Automatically updated with commit SHA (e.g., `abc123def`)
- **Production:** Manually updated for controlled releases
- Automation workflow handles dev/stage - production remains manual for safety

```bash
# Dev and stage are auto-updated by CI
# Production requires manual update:
vi environments/prod/values.yaml
# Change imageTag to desired commit SHA or version tag
```

---

## 📝 Making Changes

<details>
<summary><b>📖 How to Modify Configuration (Click to expand)</b></summary>

### Adding New Environment Variables

1. Update `environments/<env>/values.yaml`
2. Commit and push
3. ArgoCD auto-syncs (or manual sync for prod)

### Modifying Resources

1. Edit resource limits in values files
2. Test in dev first
3. Promote to stage
4. Deploy to prod after validation

### Updating Secrets

1. Update secret in AWS Secrets Manager (via infra repo)
2. External Secrets Operator syncs automatically
3. Restart pods if needed:
   ```bash
   kubectl rollout restart deployment/<name> -n <namespace>
   ```

</details>

---

## ✅ Production Checklist

Before deploying to production:

- [ ] Code reviewed and tested in dev
- [ ] QA validation completed in stage
- [ ] Version tag created (e.g., v1.0.0)
- [ ] Values file updated with version tag
- [ ] Resource limits appropriate
- [ ] Secrets configured in AWS Secrets Manager
- [ ] ArgoCD application deployed
- [ ] Manual sync performed
- [ ] Health checks passing
- [ ] Logs reviewed for errors

---

## 📚 Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Helm Documentation](https://helm.sh/docs/)
- [External Secrets Operator](https://external-secrets.io/)
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)

---

## 🤝 Contributing

1. Create feature branch
2. Update appropriate environment values
3. Test in dev environment
4. Submit pull request
5. Deploy to stage for validation
6. Manually deploy to prod after approval

---
