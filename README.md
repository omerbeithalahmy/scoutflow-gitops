# ScoutFlow GitOps Repository

> **GitOps deployment configuration for the ScoutFlow NBA analytics platform**

Production-grade Kubernetes deployment using ArgoCD, Helm, and External Secrets Operator for secure secret management.

---

## 📋 Overview

This repository manages ScoutFlow's Kubernetes deployments across three environments using **GitOps principles**:

- **Development** - Fully automated deployment with automatic image tracking
- **Staging** - QA testing with production-like configuration and automatic updates
- **Production** - Manual approval for controlled releases

**Key Technologies:**
- ✅ ArgoCD (GitOps continuous delivery)
- ✅ Helm (Kubernetes package management)
- ✅ External Secrets Operator (AWS Secrets Manager integration)
- ✅ Automated image tracking (dev/stage)

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Developer Push to scoutflow-app Repository                      │
│  └─ GitHub Actions builds images with commit SHA tag             │
└──────────────────────────────────────────────────────────────────┘
                         │
                         ↓ (update-gitops.yaml workflow)
┌──────────────────────────────────────────────────────────────────┐
│  scoutflow-gitops Repository (this repo)                         │
│  └─ environments/dev/values.yaml    ← Auto-updated with new SHA  │
│  └─ environments/stage/values.yaml  ← Auto-updated with new SHA  │
│  └─ environments/prod/values.yaml   ← Manual update only         │
└──────────────────────────────────────────────────────────────────┘
                         │
                         ↓ (ArgoCD watches)
┌──────────────────────────────────────────────────────────────────┐
│  Kubernetes Cluster (AWS EKS)                                    │
│                                                                  │
│  ArgoCD ──→ Detects Git change ──→ Deploys ScoutFlow App        │
│                                                                  │
│  External Secrets Operator ──→ AWS Secrets Manager              │
│              ↓                                                   │
│         Database Credentials (no passwords in Git!)              │
└──────────────────────────────────────────────────────────────────┘
```

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
│       ├── scoutflow-dev.yaml    # Dev ArgoCD app (auto-sync enabled)
│       ├── scoutflow-stage.yaml  # Stage ArgoCD app (auto-sync enabled)
│       └── scoutflow-prod.yaml   # Prod ArgoCD app (manual sync only)
│
└── environments/
    ├── dev/values.yaml           # Dev config (auto-updated, 1 replica)
    ├── stage/values.yaml         # Stage config (auto-updated, 2 replicas)
    └── prod/values.yaml          # Prod config (manual updates, 3 replicas)
```

---

## 🚀 Getting Started from Zero

### Prerequisites

You need the following before deploying applications:

1. **AWS Account** with appropriate IAM permissions
2. **Infrastructure Deployed** - [`scoutflow-infra`](https://github.com/omerbh7/scoutflow-infra) Terraform applied
3. **kubectl CLI** installed ([Install Guide](https://kubernetes.io/docs/tasks/tools/))
4. **ArgoCD CLI** installed (optional, for manual operations) ([Install Guide](https://argo-cd.readthedocs.io/en/stable/cli_installation/))

<details>
<summary><b>📖 Step-by-Step Setup from Scratch (Click to expand)</b></summary>

### Step 1: Configure AWS Access

```bash
# Configure AWS CLI with your credentials
aws configure
# Enter: Access Key ID, Secret Access Key, Region (us-east-1), Output (json)

# Verify access
aws sts get-caller-identity
```

### Step 2: Deploy Infrastructure

```bash
# Clone and deploy the infrastructure repository
git clone https://github.com/omerbh7/scoutflow-infra
cd scoutflow-infra/environments/dev

# Initialize and apply Terraform
terraform init
terraform plan
terraform apply

# This creates:
# - EKS cluster
# - ArgoCD installation
# - External Secrets Operator
# - AWS Secrets Manager secrets
# - Monitoring stack (Prometheus + Grafana)
```

### Step 3: Configure kubectl for EKS

```bash
# From the infra repo directory
cd environments/dev  # or stage/prod

# Configure kubectl (output from Terraform)
$(terraform output -raw configure_kubectl)

# Alternatively, manual configuration:
aws eks update-kubeconfig --name scoutflow-dev --region us-east-1
```

### Step 4: Verify Prerequisites

Run these commands to ensure everything is ready:

```bash
# Verify EKS cluster is accessible
kubectl cluster-info

# Verify ArgoCD is running
kubectl get pods -n argocd
# Expected: All ArgoCD pods in Running state

# Verify External Secrets Operator
kubectl get pods -n external-secrets-system
# Expected: external-secrets pod(s) in Running state

# Verify namespaces exist
kubectl get namespaces | grep -E 'dev|stage|prod'

# Check AWS Secrets Manager secrets exist
aws secretsmanager describe-secret --secret-id scoutflow/dev/database --region us-east-1
```

### Step 5: Deploy ArgoCD Applications

```bash
# Clone this repository
git clone https://github.com/omerbh7/scoutflow-gitops
cd scoutflow-gitops

# Deploy applications to ArgoCD
kubectl apply -f argocd/apps/scoutflow-dev.yaml
kubectl apply -f argocd/apps/scoutflow-stage.yaml
kubectl apply -f argocd/apps/scoutflow-prod.yaml

# Verify ArgoCD applications created
kubectl get applications -n argocd
# Expected: scoutflow-dev, scoutflow-stage, scoutflow-prod
```

### Step 6: Monitor Initial Sync

```bash
# Watch the sync status
kubectl get applications -n argocd -w

# Check pods being created
kubectl get pods -n dev -w
kubectl get pods -n stage -w
kubectl get pods -n prod -w

# View ArgoCD UI for visual feedback
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open: https://localhost:8080
# Username: admin
# Password: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

</details>

---

## 🤖 Automatic Image Tracking

**Dev and stage environments automatically update when you push code to `scoutflow-app`.**

<details>
<summary><b>📖 How Automatic Updates Work (Click to expand)</b></summary>

### Complete Workflow

```
1. Developer pushes code to scoutflow-app (main branch)
   ↓
2. GitHub Actions CI runs (backend-ci.yaml, frontend-ci.yaml, db-ci.yaml)
   ↓
3. Docker images built and tagged with commit SHA
   Example: 279987127424.dkr.ecr.us-east-1.amazonaws.com/scoutflow-app-backend:e7c24f013e837f438f66a6e039e673f880e2cb89
   ↓
4. Images pushed to AWS ECR
   ↓
5. update-gitops.yaml workflow triggers (workflow_run event)
   ↓
6. Workflow checks out THIS repository (scoutflow-gitops)
   ↓
7. Uses yq to update image tags in values files:
   • environments/dev/values.yaml
   • environments/stage/values.yaml
   
   Command: yq eval -i '.global.imageTag = "e7c24f01..."' environments/dev/values.yaml
   ↓
8. Commits and pushes changes to scoutflow-gitops
   Commit message: "Update dev/stage to image e7c24f01..."
   ↓
9. ArgoCD detects Git repository change
   ↓
10. ArgoCD auto-syncs dev and stage (syncPolicy.automated: true)
   ↓
11. Kubernetes pods restart with new images
   ↓
12. Deployment complete! 🎉
```

### Why Production is NOT Auto-Updated

**Production values are NOT updated by automation** for safety and control:
- Production requires manual approval
- Gives you time to validate in stage
- Prevents accidental production deployments
- You control exactly when production updates

To update production, see [Promoting to Production](#promoting-stage-to-production).

### Setup Requirements

The automatic update workflow requires a GitHub Personal Access Token (PAT):

**In scoutflow-app repository:**
1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Create secret named `GITOPS_PAT`
3. Value: GitHub Personal Access Token (classic) with `repo` scope

**Why needed:** The workflow needs permission to push commits to this repository.

### Workflow Code Reference

The automation is handled by [`scoutflow-app/.github/workflows/update-gitops.yaml`](https://github.com/omerbh7/scoutflow-app/blob/main/.github/workflows/update-gitops.yaml):

```yaml
- name: Update image tags
  env:
    IMAGE_TAG: ${{ github.event.workflow_run.head_sha }}
  run: |
    cd gitops-repo
    yq eval -i '.global.imageTag = env(IMAGE_TAG)' environments/dev/values.yaml
    yq eval -i '.global.imageTag = env(IMAGE_TAG)' environments/stage/values.yaml
```

</details>

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

**Workflow:** [.github/workflows/pr-validation.yml](.github/workflows/pr-validation.yml)

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
5. ArgoCD automatically detects changes and syncs
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

1. **Terraform creates secrets** in AWS Secrets Manager (via scoutflow-infra)
   - Random passwords generated automatically
   - Stored as: `scoutflow/{environment}/database`
   
2. **External Secrets Operator reads** from AWS Secrets Manager
   - Uses IRSA (IAM Roles for Service Accounts) for authentication
   - No AWS credentials stored in Kubernetes
   
3. **Creates Kubernetes secret**: `<release>-db-secret`
   - Contains keys: `DB_USER`, `DB_PASSWORD`, `DB_NAME`
   
4. **Application pods mount the secret**
   - Environment variables injected automatically
   - Pods reference secret via Helm templates

### Security Benefits

- ✅ Zero passwords in Git
- ✅ IAM-based authentication (IRSA)
- ✅ Automatic secret rotation support
- ✅ Audit trail in AWS CloudTrail
- ✅ Encrypted at rest (AWS KMS)

### Troubleshooting External Secrets

```bash
# Check if ExternalSecret resource exists
kubectl get externalsecret -n dev

# Describe for detailed status
kubectl describe externalsecret -n dev

# Check if Kubernetes secret was created
kubectl get secret -n dev | grep db-secret

# View External Secrets Operator logs
kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets --tail=50

# Common issues:
# 1. SecretNotFound → Check secret name matches AWS Secrets Manager
# 2. AccessDenied → Verify IRSA role has secretsmanager:GetSecretValue permission
# 3. Region mismatch → Ensure region in values.yaml matches secret location
```

</details>

---

## 🌍 Environment Configuration

<details>
<summary><b>📖 Detailed Environment Specs (Click to expand)</b></summary>

### Development

**Purpose:** Feature development and testing

- **Images:** Commit SHA tags (e.g., `e7c24f013e837f438f66a6e039e673f880e2cb89`)
- **Image Updates:** Fully automatic via CI/CD
- **Replicas:** 1 per service (database, backend, frontend)
- **Resources:** Minimal (cost-optimized)
- **Sync Policy:** Automated with self-healing and auto-pruning
- **Namespace:** `dev`
- **Database Storage:** 10Gi

### Staging

**Purpose:** QA testing and pre-production validation

- **Images:** Commit SHA tags (auto-updated by CI)
- **Image Updates:** Fully automatic via CI/CD
- **Replicas:** 2 per service (high availability testing)
- **Resources:** Production-like sizing
- **Sync Policy:** Automated with self-healing
- **Namespace:** `stage`
- **Database Storage:** 10Gi
- **Monitoring:** Prometheus + Grafana enabled

### Production

**Purpose:** Live user traffic

- **Images:** Commit SHA tags (manually updated)
- **Image Updates:** Manual only (controlled releases)
- **Replicas:** 3 per service (high availability)
- **Resources:** High CPU/memory limits
- **Sync Policy:** **Manual sync required** (no automated block)
- **Namespace:** `prod`
- **Database Storage:** 20Gi
- **Monitoring:** Full observability stack with alerting

</details>

---

## 📊 Resource Allocation

<details>
<summary><b>📖 Resource Limits by Environment (Click to expand)</b></summary>

| Environment | Backend CPU | Backend Memory | Frontend CPU | Frontend Memory | DB Storage |
|-------------|-------------|----------------|--------------|-----------------|------------|
| **Dev**     | 100m-500m   | 128Mi-256Mi    | 50m-200m     | 64Mi-128Mi      | 10Gi       |
| **Stage**   | 200m-1000m  | 256Mi-512Mi    | 100m-500m    | 128Mi-256Mi     | 10Gi       |
| **Prod**    | 500m-2000m  | 512Mi-1Gi      | 200m-1000m   | 256Mi-512Mi     | 20Gi       |

</details>

---

## 🔄 Day-2 Operations

<details>
<summary><b>📖 Promoting Stage to Production (Click to expand)</b></summary>

### Promotion Workflow

Once you've validated changes in stage, promote to production:

```bash
# 1. Check what's running in stage
kubectl get pods -n stage -l app.kubernetes.io/component=backend -o jsonpath='{.items[0].spec.containers[0].image}'
# Output: 279987127424.dkr.ecr.us-east-1.amazonaws.com/scoutflow-app-backend:e7c24f013e837f438f66a6e039e673f880e2cb89

# 2. Extract the commit SHA
# Note the SHA: e7c24f013e837f438f66a6e039e673f880e2cb89

# 3. Or check the stage values file
cat environments/stage/values.yaml | grep imageTag
# Output: imageTag: e7c24f013e837f438f66a6e039e673f880e2cb89

# 4. Update production values file
vi environments/prod/values.yaml
# Change line 8:
#   imageTag: "e7c24f013e837f438f66a6e039e673f880e2cb89"

# 5. Commit and push
git add environments/prod/values.yaml
git commit -m "prod: promote stage image e7c24f01 to production"
git push origin main

# 6. ArgoCD will detect the change (shows OutOfSync)
kubectl get application scoutflow-prod -n argocd

# 7. Review the diff in ArgoCD UI or CLI
argocd app diff scoutflow-prod

# 8. Manually sync production
argocd app sync scoutflow-prod
# Or via UI: https://localhost:8080 → scoutflow-prod → Sync

# 9. Monitor the deployment
kubectl rollout status deployment -n prod -l app.kubernetes.io/name=scoutflow

# 10. Verify pods are running new version
kubectl get pods -n prod -o wide
```

</details>

<details>
<summary><b>📖 Rolling Back to Previous Version (Click to expand)</b></summary>

### Quick Rollback

If production deployment has issues:

```bash
# 1. Check Git history for previous working version
cd scoutflow-gitops
git log --oneline environments/prod/values.yaml
# Example output:
#   a1b2c3d prod: promote stage image e7c24f01 to production
#   d4e5f6g prod: update to image abc123de
#   h7i8j9k prod: initial production deployment

# 2. Revert to previous commit
git revert HEAD --no-edit
# This creates a new commit that undoes the last change

# 3. Push the revert
git push origin main

# 4. ArgoCD detects change
kubectl get application scoutflow-prod -n argocd

# 5. Sync production with reverted config
argocd app sync scoutflow-prod

# 6. Verify rollback
kubectl get pods -n prod
kubectl logs -n prod -l app.kubernetes.io/component=backend --tail=20
```

### Alternative: Point to Specific SHA

```bash
# If you know the exact working SHA:
vi environments/prod/values.yaml
# Change imageTag to known good version

git add environments/prod/values.yaml
git commit -m "prod: rollback to known good image abc123de"
git push origin main

argocd app sync scoutflow-prod
```

</details>

<details>
<summary><b>📖 Emergency Procedures (Click to expand)</b></summary>

### ScaleDown to Zero (Emergency Stop)

```bash
# Temporarily scale down all replicas
kubectl scale deployment -n prod --replicas=0 --all

# Verify
kubectl get pods -n prod
```

### Emergency Rollback via ArgoCD

```bash
# View application history
argocd app history scoutflow-prod

# Rollback to specific revision
argocd app rollback scoutflow-prod <REVISION_NUMBER>

# Example:
argocd app rollback scoutflow-prod 5
```

### Force Refresh ArgoCD

```bash
# If ArgoCD is stuck or not detecting changes
argocd app get scoutflow-prod --refresh

# Hard refresh (clears cache)
argocd app get scoutflow-prod --hard-refresh
```

</details>

<details>
<summary><b>📖 Verifying Deployed Versions (Click to expand)</b></summary>

### Check Currently Running Images

```bash
# Backend version
kubectl get pods -n prod -l app.kubernetes.io/component=backend -o jsonpath='{.items[0].spec.containers[0].image}'

# Frontend version
kubectl get pods -n prod -l app.kubernetes.io/component=frontend -o jsonpath='{.items[0].spec.containers[0].image}'

# Database version
kubectl get pods -n prod -l app.kubernetes.io/component=database -o jsonpath='{.items[0].spec.containers[0].image}'

# All services at once
kubectl get pods -n prod -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}' | column -t
```

### Compare Git vs Cluster

```bash
# What Git says should be deployed
cat environments/prod/values.yaml | grep imageTag

# What's actually running
kubectl get pods -n prod -o jsonpath='{.items[0].spec.containers[0].image}'

# Check ArgoCD sync status
argocd app get scoutflow-prod | grep -E 'Sync Status|Health Status'
```

</details>

---

## 🛠️ Common Operations

<details>
<summary><b>📖 Updating Application Configuration (Click to expand)</b></summary>

### Change Environment Variables

```bash
# 1. Edit the values file
vi environments/stage/values.yaml

# 2. Add or modify under the relevant service (backend/frontend/database)
backend:
  env:
    LOG_LEVEL: debug
    FEATURE_FLAG_XYZ: enabled

# 3. Commit and push
git add environments/stage/values.yaml
git commit -m "stage: enable debug logging for backend"
git push origin main

# 4. ArgoCD auto-syncs (for stage), monitor
kubectl rollout status deployment/scoutflow-backend -n stage
```

### Scale Replicas

```bash
# 1. Edit values file
vi environments/stage/values.yaml

# 2. Update replicas count
backend:
  replicas: 3  # Changed from 2

# 3. Commit and push
git add environments/stage/values.yaml
git commit -m "scale: increase stage backend to 3 replicas"
git push origin main

# 4. Verify scaling
kubectl get pods -n stage -l app.kubernetes.io/component=backend -w
```

### Modify Resource Limits

```bash
# 1. Edit values file
vi environments/prod/values.yaml

# 2. Update resources
backend:
  resources:
    requests:
      cpu: 1000m      # Increased from 500m
      memory: 1Gi     # Increased from 512Mi
    limits:
      cpu: 3000m      # Increased from 2000m
      memory: 2Gi     # Increased from 1Gi

# 3. Commit, push, and manually sync (production)
git add environments/prod/values.yaml
git commit -m "prod: increase backend resource limits"
git push origin main
argocd app sync scoutflow-prod

# 4. Monitor rolling update
kubectl rollout status deployment/scoutflow-backend -n prod
```

</details>

<details>
<summary><b>📖 ArgoCD Management Commands (Click to expand)</b></summary>

### View Application Status

```bash
# Summary status
argocd app get scoutflow-dev

# Detailed sync status
argocd app sync-status scoutflow-dev

# Check health of all apps
argocd app list
```

### Manual Sync Operations

```bash
# Sync specific application
argocd app sync scoutflow-prod

# Sync with prune (remove deleted resources)
argocd app sync scoutflow-prod --prune

# Sync only specific resource
argocd app sync scoutflow-prod --resource apps:Deployment:scoutflow-backend
```

### Access ArgoCD UI

```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open browser to: https://localhost:8080
# Username: admin
# Password: (from above)
```

</details>

<details>
<summary><b>📖 Monitoring and Logs (Click to expand)</b></summary>

### View Application Logs

```bash
# Backend logs
kubectl logs -n dev -l app.kubernetes.io/component=backend --tail=100 -f

# Frontend logs
kubectl logs -n dev -l app.kubernetes.io/component=frontend --tail=100 -f

# Database logs
kubectl logs -n dev -l app.kubernetes.io/component=database --tail=100 -f

# All pod logs in namespace
kubectl logs -n dev --all-containers=true --tail=50
```

### Access Monitoring (Stage/Prod Only)

**Grafana:**

```bash
# Get Grafana admin password (from infra repo)
cd ~/scoutflow-infra/environments/stage
terraform output -raw grafana_admin_password

# Port forward Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Open: http://localhost:3000
# Username: admin
# Password: (from terraform output)
```

**Prometheus:**

```bash
# Port forward Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Open: http://localhost:9090
```

### Check Resource Usage

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods -n prod

# Specific deployment
kubectl top pods -n prod -l app.kubernetes.io/component=backend
```

</details>

---

## 🚨 Troubleshooting

<details>
<summary><b>📖 ArgoCD Application Issues (Click to expand)</b></summary>

### ArgoCD Shows "OutOfSync"

**Symptoms:**
- Application status shows `OutOfSync`
- Resources not being applied to cluster

**Diagnosis:**

```bash
# Check what's out of sync
argocd app diff scoutflow-dev

# View application details
kubectl describe application scoutflow-dev -n argocd
```

**Resolution:**

```bash
# For dev/stage (should auto-sync):
# Check if auto-sync is enabled
kubectl get application scoutflow-dev -n argocd -o yaml | grep -A 5 syncPolicy

# Force refresh
argocd app get scoutflow-dev --refresh

# For production (manual sync required):
argocd app sync scoutflow-prod
```

### ArgoCD Application Stuck in "Progressing"

**Symptoms:**
- Sync says "Progressing" for extended time
- Resources not reaching healthy state

**Diagnosis:**

```bash
# Check application status
argocd app get scoutflow-dev

# Check events
kubectl get events -n dev --sort-by='.lastTimestamp'

# Check pod status
kubectl get pods -n dev
kubectl describe pod <pod-name> -n dev
```

**Common Causes:**
1. **Image pull errors** - See [Pods in ImagePullBackOff](#pods-stuck-in-imagepullbackoff)
2. **Resource constraints** - See [Pods Pending/Not Scheduling](#pods-pending-or-not-scheduling)
3. **External Secrets not syncing** - See [External Secrets Not Creating Kubernetes Secret](#external-secrets-not-creating-kubernetes-secret)

</details>

<details>
<summary><b>📖 Pod and Deployment Issues (Click to expand)</b></summary>

### Pods Stuck in ImagePullBackOff

**Symptoms:**
```
NAME                                  READY   STATUS             RESTARTS
scoutflow-backend-5d7c8b9f4-xyz       0/1     ImagePullBackOff   0
```

**Diagnosis:**

```bash
# Check exact error
kubectl describe pod <pod-name> -n dev | grep -A 10 Events

# Common errors:
# 1. "no basic auth credentials" → ECR authentication issue
# 2. "manifest unknown" → Image doesn't exist in ECR
# 3. "unauthorized" → ECR registry secret incorrect
```

**Resolution:**

```bash
# Option 1: Check ECR secret exists
kubectl get secret ecr-registry-secret -n dev

# If missing, recreate (done by Helm, check External Secrets)
kubectl get externalsecret -n dev

# Option 2: Verify image exists in ECR
aws ecr describe-images \
  --repository-name scoutflow-app-backend \
  --image-ids imageTag=e7c24f013e837f438f66a6e039e673f880e2cb89 \
  --region us-east-1

# Option 3: Check image tag in values file
cat environments/dev/values.yaml | grep imageTag

# Option 4: Manual pod delete (force restart)
kubectl delete pod <pod-name> -n dev
```

### Pods in CrashLoopBackOff

**Symptoms:**
```
NAME                                  READY   STATUS             RESTARTS
scoutflow-backend-5d7c8b9f4-xyz       0/1     CrashLoopBackOff   5
```

**Diagnosis:**

```bash
# Check logs
kubectl logs <pod-name> -n dev --previous

# Common issues:
# 1. Missing database credentials
# 2. Cannot connect to database
# 3. Application code error
```

**Resolution:**

```bash
# Check database secret exists
kubectl get secret -n dev | grep db-secret

# Describe the secret (don't decode, just check keys)
kubectl get secret scoutflow-db-secret -n dev -o jsonpath='{.data}' | jq 'keys'
# Expected: ["DB_NAME", "DB_PASSWORD", "DB_USER"]

# Check database pod is running
kubectl get pods -n dev -l app.kubernetes.io/component=database

# Test database connection from backend pod
kubectl exec -it <backend-pod> -n dev -- env | grep DB_
```

### Pods Pending or Not Scheduling

**Symptoms:**
```
NAME                                  READY   STATUS    RESTARTS
scoutflow-backend-5d7c8b9f4-xyz       0/1     Pending   0
```

**Diagnosis:**

```bash
# Check why pod is pending
kubectl describe pod <pod-name> -n dev | grep -A 10 Events

# Common messages:
# 1. "Insufficient cpu" → Node has no available CPU
# 2. "Insufficient memory" → Node has no available memory
# 3. "No nodes available" → All nodes are full or tainted
```

**Resolution:**

```bash
# Check node resources
kubectl top nodes
kubectl describe nodes | grep -A 10 "Allocated resources"

# Option 1: Scale down other workloads
kubectl scale deployment <other-deployment> -n dev --replicas=0

# Option 2: Reduce resource requests (edit values file)
vi environments/dev/values.yaml
# Reduce backend.resources.requests.cpu and memory

# Option 3: Scale cluster nodes (via scoutflow-infra Terraform)
cd ~/scoutflow-infra/environments/dev
vi variables.tf  # Increase node_desired_size
terraform apply
```

</details>

<details>
<summary><b>📖 External Secrets Issues (Click to expand)</b></summary>

### External Secrets Not Creating Kubernetes Secret

**Symptoms:**
- ExternalSecret resource exists but secret not created
- Pods failing with missing secret errors

**Diagnosis:**

```bash
# Check ExternalSecret status
kubectl get externalsecret -n dev
kubectl describe externalsecret -n dev

# Look for status conditions
kubectl get externalsecret scoutflow-external-secret -n dev -o yaml | grep -A 10 status

# Check External Secrets Operator logs
kubectl logs -n external-secrets-system \
  -l app.kubernetes.io/name=external-secrets \
  --tail=100
```

**Common Error Messages and Fixes:**

**Error: "SecretNotFound"**
```bash
# Secret name in values file doesn't match AWS Secrets Manager
cat environments/dev/values.yaml | grep secretName
# Should be: "scoutflow/dev/database"

# Verify in AWS
aws secretsmanager describe-secret \
  --secret-id scoutflow/dev/database \
  --region us-east-1
```

**Error: "AccessDenied"**
```bash
# IRSA role doesn't have GetSecretValue permission
# Check service account
kubectl get sa -n dev

# Verify annotation
kubectl get sa default -n dev -o yaml | grep eks.amazonaws.com/role-arn

# Fix in scoutflow-infra Terraform (IAM policy)
```

**Error: "Region mismatch"**
```bash
# Check region in values file
cat environments/dev/values.yaml | grep region
# Should be: us-east-1

# Verify secret region in AWS
aws secretsmanager describe-secret \
  --secret-id scoutflow/dev/database \
  --region us-east-1
```

</details>

<details>
<summary><b>📖 GitOps Sync Issues (Click to expand)</b></summary>

### Changes Not Appearing in Cluster

**Symptoms:**
- Pushed Git changes but ArgoCD not syncing
- Application still running old configuration

**Diagnosis:**

```bash
# Check ArgoCD application status
argocd app get scoutflow-dev

# Check last sync time
kubectl get application scoutflow-dev -n argocd -o yaml | grep -A 5 "lastSyncedAt"

# Verify Git repo is reachable
argocd app get scoutflow-dev | grep -E 'Repo|Path'
```

**Resolution:**

```bash
# Option 1: Force refresh
argocd app get scoutflow-dev --refresh

# Option 2: Ensure auto-sync enabled (dev/stage only)
kubectl get application scoutflow-dev -n argocd -o yaml | grep -A 5 syncPolicy

# Option 3: Manual sync
argocd app sync scoutflow-dev

# Option 4: Check ArgoCD application controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=100
```

### Automatic Image Updates Not Working

**Symptoms:**
- Pushed code to scoutflow-app but dev/stage not updating
- imageTag in values files still showing old SHA

**Diagnosis:**

```bash
# Check update-gitops workflow in scoutflow-app
# Go to: https://github.com/omerbh7/scoutflow-app/actions
# Look for failed "Update GitOps Repository" workflows

# Check Git history of this repo
cd ~/scoutflow-gitops
git log --oneline environments/dev/values.yaml
# Should show recent "Update dev/stage to image ..." commits
```

**Common Causes:**

1. **GITOPS_PAT secret not configured**
   ```bash
   # In scoutflow-app repository settings:
   # Settings → Secrets → Actions → Check for GITOPS_PAT
   ```

2. **CI build failed before update-gitops**
   ```bash
   # Check scoutflow-app GitHub Actions
   # Backend CI, Frontend CI, or Database CI failed
   ```

3. **update-gitops workflow failed**
   ```bash
   # Check workflow logs
   # Common error: "remote: Permission denied"
   # Fix: Regenerate GITOPS_PAT with correct permissions
   ```

</details>

---

## 🔗 Integration with Other Repositories

### [scoutflow-infra](https://github.com/omerbh7/scoutflow-infra)

**Provides:**
- AWS EKS Kubernetes cluster
- ArgoCD installation and configuration
- External Secrets Operator deployment
- AWS Secrets Manager secrets with database credentials
- IAM roles and IRSA for service accounts
- Prometheus + Grafana monitoring stack (stage/prod)
- Load Balancer Controller
- VPC networking and security groups

**Required before deploying this repo:** ✅ Yes - Infrastructure must be deployed first

### [scoutflow-app](https://github.com/omerbh7/scoutflow-app)

**Provides:**
- Helm chart templates
- Docker images (backend, frontend, database)
- Application source code
- CI/CD workflows that build and push images
- `update-gitops.yaml` workflow that updates this repository

**Referenced by:** ArgoCD Application manifests in this repo pull Helm charts from scoutflow-app

---

## 🚨 Important Notes

### Production Deployment

> [!WARNING]
> **Production requires manual sync** - This is by design for controlled releases

```bash
# Never assume production auto-syncs
# Always review changes before syncing
argocd app diff scoutflow-prod
argocd app sync scoutflow-prod
```

### Secret Updates

> [!CAUTION]
> **Secrets are managed by infrastructure repo (scoutflow-infra)**

- Database passwords: AWS Secrets Manager (created by Terraform)
- ECR credentials: Managed by External Secrets Operator
- **Never commit passwords to this repository**
- To rotate secrets: Update in AWS Secrets Manager, then restart pods

### Image Tag Strategy

> [!IMPORTANT]
> **All environments use commit SHA tags**

- **Dev/Stage:** Automatically updated by scoutflow-app CI/CD with commit SHA
- **Production:** Manually updated with validated commit SHA
- **Format:** 40-character Git commit hash (e.g., `e7c24f013e837f438f66a6e039e673f880e2cb89`)
- **Why SHAs:** Immutable, traceable, and matches Git commit history

```bash
# Check current image tag in any environment
cat environments/prod/values.yaml | grep imageTag

# Trace SHA back to Git commit
cd ~/scoutflow-app
git show e7c24f013e837f438f66a6e039e673f880e2cb89
```

---

## ✅ Production Deployment Checklist

Before deploying to production, verify:

- [ ] **Validated in Stage:** Run `kubectl get pods -n stage` - all pods healthy
- [ ] **Image SHA Identified:** Extract SHA from stage: `cat environments/stage/values.yaml | grep imageTag`
- [ ] **ArgoCD Diff Reviewed:** Run `argocd app diff scoutflow-prod` after updating prod values
- [ ] **Database Secrets Exist:** Run `kubectl get secret -n prod | grep db-secret`
- [ ] **Resource Limits Appropriate:** Review `environments/prod/values.yaml` resources section
- [ ] **Backup Plan Ready:** Document rollback SHA in PR or commit message
- [ ] **Manual Sync Performed:** Run `argocd app sync scoutflow-prod`
- [ ] **Deployment Monitored:** Run `kubectl rollout status deployment -n prod -l app.kubernetes.io/name=scoutflow`
- [ ] **Health Checks Passing:** Run `kubectl get pods -n prod` - all Running
- [ ] **Logs Reviewed:** Run `kubectl logs -n prod -l app.kubernetes.io/component=backend --tail=50`
- [ ] **Monitoring Verified:** Check Grafana dashboards for anomalies

---

## 📚 Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Helm Documentation](https://helm.sh/docs/)
- [External Secrets Operator](https://external-secrets.io/)
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)
- [GitOps Principles](https://www.gitops.tech/)

---

## 🤝 Contributing

1. Create feature branch from main
2. Update appropriate environment values files
3. Test changes in dev environment first
4. Run PR validation (automatic on PR creation)
5. Deploy to stage for QA validation
6. After approval, manually promote to prod

**Workflow:**
```bash
git checkout -b feature/update-backend-replicas
vi environments/dev/values.yaml  # Make changes
git add environments/dev/values.yaml
git commit -m "dev: increase backend replicas to 2"
git push origin feature/update-backend-replicas
# Create PR, wait for validation, merge
```

---
