# ScoutFlow GitOps Repository

> **GitOps deployment configuration for the ScoutFlow NBA analytics platform**

This repository implements GitOps principles using ArgoCD to manage Kubernetes deployments across three environments (dev, stage, prod) with automated image tracking and secure secret management.

---

## 📋 Overview

This repository serves as the **single source of truth** for ScoutFlow's Kubernetes deployments. It defines three isolated environments with different deployment strategies:

- **Development** - Fully automated deployments with auto-sync and self-healing
- **Staging** - Production-like environment with automated deployments for QA validation  
- **Production** - Manual-approval deployments for controlled releases

**Core Technologies:**
- **ArgoCD** - Continuous delivery tool that watches this Git repository and syncs changes to Kubernetes
- **Helm** - Package manager that templates Kubernetes manifests (charts stored in `scoutflow-app` repo)
- **External Secrets Operator** - Syncs secrets from AWS Secrets Manager to Kubernetes
- **GitHub Actions** - CI/CD pipeline for validating YAML syntax and security

---

## 🏗️ Architecture

### GitOps Flow

```
┌──────────────────────────────────────────────────────────────────┐
│  Developer pushes code to scoutflow-app repository               │
└──────────────────────────────────────────────────────────────────┘
                         │
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  GitHub Actions CI/CD (in scoutflow-app)                         │
│  • Builds Docker images tagged with commit SHA                   │
│  • Pushes to AWS ECR                                             │
│  • Triggers update-gitops.yaml workflow                          │
└──────────────────────────────────────────────────────────────────┘
                         │
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  THIS REPOSITORY (scoutflow-gitops)                              │
│  • update-gitops workflow commits new SHA to values files        │
│  • Updates environments/dev/values.yaml                          │
│  • Updates environments/stage/values.yaml                        │
│  • Production (prod) is NOT auto-updated                         │
└──────────────────────────────────────────────────────────────────┘
                         │
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  ArgoCD (running in EKS cluster)                                 │
│  • Watches this Git repository for changes                       │
│  • Detects new commit SHA in values files                        │
│  • Auto-syncs dev and stage (syncPolicy.automated: true)         │
│  • Waits for manual sync on prod (no automated policy)           │
└──────────────────────────────────────────────────────────────────┘
                         │
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  Kubernetes Cluster (AWS EKS)                                    │
│  • Pods restart with new images                                  │
│  • External Secrets Operator syncs DB credentials from AWS       │
│  • Application deployed across namespaces (dev, stage, prod)     │
└──────────────────────────────────────────────────────────────────┘
```

### Secret Management Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  AWS Secrets Manager                                        │
│  • scoutflow/dev/database                                   │
│  • scoutflow/stage/database                                 │
│  • scoutflow/prod/database                                  │
│  (Created by scoutflow-infra Terraform)                     │
└─────────────────────────────────────────────────────────────┘
                         │
                         ↓ (IRSA authentication)
┌─────────────────────────────────────────────────────────────┐
│  External Secrets Operator (in EKS)                         │
│  • ClusterSecretStore configured with AWS region            │
│  • ExternalSecret resources defined in Helm chart           │
│  • Reads from AWS Secrets Manager using IAM role            │
└─────────────────────────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  Kubernetes Secrets                                         │
│  • Created automatically: <release>-db-secret               │
│  • Contains: DB_USER, DB_PASSWORD, DB_NAME                  │
│  • Mounted as environment variables in pods                 │
└─────────────────────────────────────────────────────────────┘
```

**Key Security Feature:** Zero database credentials stored in Git. All secrets managed externally via AWS Secrets Manager.

---

## 📁 Repository Structure

```
scoutflow-gitops/
├── .github/
│   └── workflows/
│       └── pr-validation.yml          # Validates YAML syntax and security on PRs
│
├── argocd/
│   └── apps/
│       ├── scoutflow-dev.yaml         # ArgoCD Application for dev namespace
│       ├── scoutflow-stage.yaml       # ArgoCD Application for stage namespace
│       └── scoutflow-prod.yaml        # ArgoCD Application for prod namespace
│
└── environments/
    ├── dev/values.yaml                # Helm values for dev (auto-updated)
    ├── stage/values.yaml              # Helm values for stage (auto-updated)
    └── prod/values.yaml               # Helm values for prod (manual updates)
```

<details>
<summary><b>ArgoCD Application Manifests</b></summary>

Located in `argocd/apps/`, these manifests define how ArgoCD deploys the application:

**Multi-Source Configuration:**
- **Source 1:** Helm chart from `scoutflow-app` repository (`helm/scoutflow` path)
- **Source 2:** Values files from this repository (`environments/{env}/values.yaml`)

**Sync Policies:**
- **Dev/Stage:** `syncPolicy.automated` enabled with `prune: true` and `selfHeal: true`
- **Production:** No `automated` block - requires manual sync via ArgoCD UI or CLI

**Example (scoutflow-dev.yaml):**
```yaml
spec:
  sources:
    - repoURL: https://github.com/omerbh7/scoutflow-app
      path: helm/scoutflow
      helm:
        valueFiles:
          - $values/environments/dev/values.yaml
    - repoURL: https://github.com/omerbh7/scoutflow-gitops
      ref: values
  
  syncPolicy:
    automated:
      prune: true        # Delete resources removed from Git
      selfHeal: true     # Revert manual kubectl changes
```

</details>

<details>
<summary><b>Environment Values Files</b></summary>

Located in `environments/{env}/values.yaml`, these files configure environment-specific settings:

**Key Configuration:**

```yaml
global:
  registry: 279987127424.dkr.ecr.us-east-1.amazonaws.com
  imageTag: e7c24f013e837f438f66a6e039e673f880e2cb89  # Commit SHA

externalSecrets:
  enabled: true
  region: us-east-1
  secretName: "scoutflow/dev/database"

backend:
  replicas: 1  # Dev: 1, Stage: 2, Prod: 3
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
```

**Image Tag Strategy:**
- All environments use **commit SHA tags** (40-character Git hash)
- Dev/Stage: Auto-updated by `scoutflow-app` CI/CD workflow
- Production: Manually updated for controlled releases
- Format: `e7c24f013e837f438f66a6e039e673f880e2cb89`

</details>

---

## 🤖 Automatic Image Tracking

<details>
<summary><b>Implementation Details</b></summary>

The `scoutflow-app` repository contains a workflow (`update-gitops.yaml`) that automatically updates dev and stage environments when new images are built.

**Workflow Trigger:**
```yaml
on:
  workflow_run:
    workflows: ["Backend CI", "Frontend CI", "Database CI"]
    types: [completed]
    branches: [main]
```

**Update Mechanism:**
1. Checks out this repository using `GITOPS_PAT` secret
2. Uses `yq` to update image tags in values files:
   ```bash
   yq eval -i '.global.imageTag = env(IMAGE_TAG)' environments/dev/values.yaml
   yq eval -i '.global.imageTag = env(IMAGE_TAG)' environments/stage/values.yaml
   ```
3. Commits changes: `"Update dev/stage to image {SHA}"`
4. Pushes to main branch
5. ArgoCD detects Git change and auto-syncs

**Why Production is Excluded:**
Production values are intentionally NOT updated by automation. This provides:
- Manual approval gate for production changes
- Time to validate in stage before production deployment
- Explicit control over production release timing
- Reduced risk of accidental production deployments

</details>

---

## 🌍 Environment Configurations

<details>
<summary><b>Development Environment</b></summary>

**Purpose:** Rapid iteration and feature testing

**Configuration:**
- **Namespace:** `dev`
- **Replicas:** 1 per service (backend, frontend, database)
- **Image Updates:** Fully automatic via CI/CD
- **Sync Policy:** Automated with self-healing and pruning
- **Resources:** Minimal (100m CPU, 128Mi memory for backend)
- **Database Storage:** 10Gi PersistentVolume

**ArgoCD Sync Settings:**
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    allowEmpty: false
```

</details>

<details>
<summary><b>Staging Environment</b></summary>

**Purpose:** QA validation and pre-production testing

**Configuration:**
- **Namespace:** `stage`
- **Replicas:** 2 per service (HA testing)
- **Image Updates:** Fully automatic via CI/CD
- **Sync Policy:** Automated with self-healing
- **Resources:** Production-like (200m-1000m CPU, 256Mi-512Mi memory)
- **Database Storage:** 10Gi PersistentVolume
- **Monitoring:** Prometheus + Grafana enabled (via scoutflow-infra)

**Purpose of Stage:**
- Validate changes in production-like environment
- QA testing before production promotion
- Performance testing with realistic resource constraints

</details>

<details>
<summary><b>Production Environment</b></summary>

**Purpose:** Live user-facing workloads

**Configuration:**
- **Namespace:** `prod`
- **Replicas:** 3 per service (high availability)
- **Image Updates:** Manual only
- **Sync Policy:** Manual sync required (no `automated` block)
- **Resources:** High limits (500m-2000m CPU, 512Mi-1Gi memory)
- **Database Storage:** 20Gi PersistentVolume
- **Monitoring:** Full observability stack with alerting

**Manual Sync Requirement:**
```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
    - Validate=true
  # No 'automated' block - requires manual argocd app sync
```

</details>

---

## 🔐 Secret Management

<details>
<summary><b>External Secrets Operator Integration</b></summary>

This repository configures External Secrets Operator to sync database credentials from AWS Secrets Manager.

**Configuration in Values Files:**
```yaml
externalSecrets:
  enabled: true
  region: us-east-1
  secretName: "scoutflow/dev/database"
```

**How It Works:**

1. **Helm chart** (in scoutflow-app) creates `ExternalSecret` resource
2. **ExternalSecret** references AWS Secrets Manager secret name
3. **ClusterSecretStore** configured with AWS region and IRSA authentication
4. **External Secrets Operator** reads from AWS using IAM role
5. **Kubernetes Secret** created automatically with keys: `DB_USER`, `DB_PASSWORD`, `DB_NAME`
6. **Application pods** mount secret as environment variables

**Security Benefits:**
- ✅ Zero credentials in Git
- ✅ IAM-based authentication (IRSA - no AWS keys in cluster)
- ✅ Secrets encrypted at rest in AWS (KMS)
- ✅ Audit trail via AWS CloudTrail
- ✅ Centralized secret management

**Secret Naming Convention:**
- Dev: `scoutflow/dev/database`
- Stage: `scoutflow/stage/database`
- Prod: `scoutflow/prod/database`

</details>

---

## 🔄 CI/CD Pipeline

<details>
<summary><b>PR Validation Workflow</b></summary>

**File:** `.github/workflows/pr-validation.yml`

**Purpose:** Validate YAML syntax and security before merging changes

**Triggers:**
- Pull requests to main branch
- Pushes to main branch

**Validation Steps:**

1. **YAML Lint** (warnings only)
   - Uses `yamllint` with relaxed configuration
   - Checks code style and formatting
   - Does not fail build

2. **YAML Syntax Validation** (required)
   - Parses all `.yaml` files with Python
   - Ensures valid YAML structure
   - Fails build on syntax errors

3. **Security Scan** (warnings only)
   - Uses Checkov for security misconfigurations
   - Identifies potential security issues
   - Does not fail build (`soft_fail: true`)

**Why This Matters:**
- Prevents broken YAML from reaching ArgoCD
- Catches syntax errors before deployment
- Maintains code quality standards
- Identifies security issues early

</details>

---

## 📊 Resource Allocation

### Per-Environment Resource Limits

| Environment | Backend CPU | Backend Memory | Frontend CPU | Frontend Memory | DB Storage |
|-------------|-------------|----------------|--------------|-----------------|------------|
| **Dev**     | 100m-500m   | 128Mi-256Mi    | 50m-200m     | 64Mi-128Mi      | 10Gi       |
| **Stage**   | 200m-1000m  | 256Mi-512Mi    | 100m-500m    | 128Mi-256Mi     | 10Gi       |
| **Prod**    | 500m-2000m  | 512Mi-1Gi      | 200m-1000m   | 256Mi-512Mi     | 20Gi       |

**Resource Strategy:**
- **Dev:** Minimal resources for cost optimization
- **Stage:** Production-like sizing for realistic testing
- **Prod:** High limits for performance and availability

**Replica Strategy:**
- **Dev:** 1 replica (single pod per service)
- **Stage:** 2 replicas (HA testing)
- **Prod:** 3 replicas (high availability)

---

## 🔗 Integration with Other Repositories

### scoutflow-infra

**Repository:** [`github.com/omerbh7/scoutflow-infra`](https://github.com/omerbh7/scoutflow-infra)

**Provides:**
- AWS EKS Kubernetes cluster
- ArgoCD installation (Helm addon)
- External Secrets Operator deployment
- AWS Secrets Manager secrets (database credentials)
- IAM roles and IRSA configuration
- Prometheus + Grafana monitoring stack (stage/prod)
- VPC networking and security groups
- Load Balancer Controller

**Relationship:** Infrastructure must be deployed before this GitOps repository can function. ArgoCD running in the cluster watches this repository.

### scoutflow-app

**Repository:** [`github.com/omerbh7/scoutflow-app`](https://github.com/omerbh7/scoutflow-app)

**Provides:**
- Helm chart templates (`helm/scoutflow/`)
- Docker images (backend, frontend, database)
- Application source code
- CI/CD workflows (backend-ci, frontend-ci, db-ci)
- `update-gitops.yaml` workflow that updates this repository

**Relationship:** ArgoCD pulls Helm charts from scoutflow-app and values files from this repository (multi-source configuration).

---

## 🚨 Key Design Decisions

### Why GitOps?

**Benefits Implemented:**
- **Declarative:** Entire deployment state defined in Git
- **Versioned:** Full audit trail of all changes
- **Automated:** ArgoCD continuously reconciles desired vs actual state
- **Recoverable:** Can restore any previous state via Git history
- **Secure:** No kubectl access needed for deployments

### Why Commit SHA Tags Instead of "latest"?

**Rationale:**
- **Immutable:** SHA tags never change, ensuring reproducibility
- **Traceable:** Direct mapping to Git commit in scoutflow-app
- **Debuggable:** Can trace exact code version running in cluster
- **Rollback-friendly:** Easy to identify and revert to previous versions

**Example:**
```
imageTag: e7c24f013e837f438f66a6e039e673f880e2cb89
         └─ Maps to: git show e7c24f01 in scoutflow-app
```

### Why Manual Production Sync?

**Rationale:**
- **Control:** Explicit approval required for production changes
- **Validation:** Time to test in stage before production
- **Safety:** Prevents accidental production deployments
- **Compliance:** Audit trail of who approved production changes

**Implementation:**
```yaml
# scoutflow-prod.yaml has NO automated sync policy
syncPolicy:
  syncOptions:
    - CreateNamespace=true
  # No 'automated' block
```

### Why Multi-Source ArgoCD Applications?

**Rationale:**
- **Separation of Concerns:** Helm charts (code) separate from values (config)
- **Reusability:** Same Helm chart used across all environments
- **Flexibility:** Can update values without touching chart
- **Security:** Values repo can have different access controls

**Implementation:**
```yaml
sources:
  - repoURL: github.com/omerbh7/scoutflow-app  # Helm chart
    path: helm/scoutflow
  - repoURL: github.com/omerbh7/scoutflow-gitops  # Values
    ref: values
```

---

## 📈 Deployment Workflow

<details>
<summary><b>Development/Stage Deployment Flow</b></summary>

```
1. Developer pushes code to scoutflow-app (main branch)
2. GitHub Actions builds Docker images with commit SHA tag
3. Images pushed to AWS ECR
4. update-gitops.yaml workflow triggers
5. Workflow updates dev/values.yaml and stage/values.yaml with new SHA
6. Commits pushed to scoutflow-gitops (this repo)
7. ArgoCD detects Git change within 3 minutes (polling interval)
8. ArgoCD auto-syncs dev and stage applications
9. Kubernetes pods restart with new images
10. Deployment complete
```

**Time to Deployment:** ~5-10 minutes from code push to running in dev/stage

</details>

<details>
<summary><b>Production Deployment Flow</b></summary>

```
1. Validate changes in stage environment
2. Identify commit SHA running in stage
3. Manually update environments/prod/values.yaml with SHA
4. Commit and push to scoutflow-gitops
5. ArgoCD detects change (shows "OutOfSync")
6. Operator reviews diff in ArgoCD UI
7. Manual sync triggered: argocd app sync scoutflow-prod
8. Kubernetes performs rolling update
9. Monitor deployment status
10. Verify application health
```

**Manual Steps:** Update values file, review diff, trigger sync

</details>

---

## 🛡️ Security Features

### No Secrets in Git

- Database credentials stored in AWS Secrets Manager
- External Secrets Operator syncs to Kubernetes
- Zero passwords committed to this repository

### IRSA Authentication

- External Secrets Operator uses IAM role (no AWS keys)
- Service accounts annotated with IAM role ARN
- Configured in scoutflow-infra Terraform

### Automated Pruning

- ArgoCD removes resources deleted from Git
- Prevents orphaned resources in cluster
- Enabled via `syncPolicy.automated.prune: true`

### Self-Healing

- ArgoCD reverts manual kubectl changes
- Ensures Git remains source of truth
- Enabled via `syncPolicy.automated.selfHeal: true`

---

## 📝 Summary

This repository implements a production-grade GitOps workflow for the ScoutFlow platform with:

- ✅ **Automated deployments** for dev/stage environments
- ✅ **Manual approval** for production releases
- ✅ **Secure secret management** via AWS Secrets Manager
- ✅ **Immutable deployments** using commit SHA tags
- ✅ **Multi-environment isolation** (dev, stage, prod)
- ✅ **CI/CD validation** for YAML syntax and security
- ✅ **Full audit trail** via Git history
- ✅ **Zero credentials in Git** via External Secrets Operator

**Key Achievement:** Complete separation of application code (scoutflow-app) and deployment configuration (this repo) following GitOps best practices.
