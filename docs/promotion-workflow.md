# Promotion Workflow: Staging → Production

How to safely promote tested changes from staging to production.

---

## Overview

```
Staging (automatic)          Production (manual)
     ↓                              ↓
latest tag                      v1.0.0 tag
     ↓                              ↓
1 replica                       2 replicas
     ↓                              ↓
Test here first          ──→   Promote when ready
```

---

## Step 1: Test in Staging

### Push Code

```bash
cd scoutflow-app
git checkout -b feature/new-stats
# ... make changes ...
git commit -m "Add new player stats"
git push origin feature/new-stats
```

### Merge to Main

```bash
# Create PR, review, merge to main
# CI automatically:
# - Builds Docker image
# - Tags as 'latest'
# - Pushes to ECR
```

### Verify Auto-Deployment

```bash
# ArgoCD auto-syncs staging
# Check in ArgoCD UI or:
kubectl get pods -n staging -w

# Test the application
kubectl port-forward -n staging svc/scoutflow-frontend 3000:80
# Open: http://localhost:3000
```

---

## Step 2: Create Release Tag

### Tag in App Repository

```bash
cd scoutflow-app

# Create version tag
git tag v1.1.0
git push origin v1.1.0
```

### CI Builds Production Image

```
GitHub Actions:
✓ Detects v1.1.0 tag
✓ Builds Docker image
✓ Tags as: 279987127424.dkr.ecr.us-east-1.amazonaws.com/.../backend:v1.1.0
✓ Pushes to ECR
```

---

## Step 3: Update GitOps Repository

### Edit Production Values

```bash
cd scoutflow-gitops

# Create branch for release
git checkout -b release/v1.1.0

# Edit production values
vim environments/production/values.yaml
```

**Changes to make**:

```yaml
global:
  imageTag: v1.1.0  # Changed from v1.0.0

backend:
  image:
    tag: v1.1.0  # Changed from v1.0.0

frontend:
  image:
    tag: v1.1.0  # Changed from v1.0.0

database:
  image:
    tag: v1.1.0  # Changed from v1.0.0

ingest:
  image:
    tag: v1.1.0  # Changed from v1.0.0
```

### Commit and Push

```bash
git add environments/production/values.yaml
git commit -m "Release v1.1.0 to production

- Update all image tags to v1.1.0
- Includes new player stats feature
- Tested in staging"

git push origin release/v1.1.0
```

### Merge to Main

```bash
# Create PR on GitHub
# Review changes
# Merge to main
```

---

## Step 4: Deploy to Production

### Sync in ArgoCD UI

1. ArgoCD detects change in `scoutflow-gitops`
2. `scoutflow-production` shows "OutOfSync"
3. Open ArgoCD UI:
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
4. Click on `scoutflow-production`
5. Click "SYNC" button
6. Review the diff (shows image tag changes)
7. Click "SYNCHRONIZE"

### Watch Rolling Update

```bash
kubectl get pods -n production -w
```

**What happens**:
```
backend-old-xxxx    1/1  Running      # Old pod
backend-new-yyyy    0/1  Creating     # New pod starting
backend-new-yyyy    1/1  Running      # New pod ready
backend-old-xxxx    0/1  Terminating  # Old pod shutting down
```

### Verify Deployment

```bash
# Check all pods running
kubectl get pods -n production

# Check image versions
kubectl describe pod -n production <pod-name> | grep Image:
```

---

## Rollback (If Needed)

### Option 1: ArgoCD UI Rollback

1. Open ArgoCD UI
2. Click "History and rollback"
3. Select previous revision
4. Click "Rollback"

### Option 2: Git Revert

```bash
cd scoutflow-gitops

# Revert to previous version
git revert HEAD
git push origin main

# Sync in ArgoCD UI
```

### Option 3: Quick Fix

```bash
# Edit production values back to v1.0.0
vim environments/production/values.yaml
git commit -m "Rollback to v1.0.0"
git push origin main

# Sync in ArgoCD
```

---

## Best Practices

✅ **Test in staging first** - Always verify in staging before production  
✅ **Use semantic versioning** - v1.0.0, v1.1.0, v2.0.0  
✅ **Write detailed commit messages** - Explain what changed and why  
✅ **Monitor after deployment** - Watch logs and metrics  
✅ **Keep rollback ready** - Know how to quickly revert

---

## Deployment Checklist

- [ ] Feature tested in staging
- [ ] Release tag created in scoutflow-app
- [ ] CI built and pushed versioned images
- [ ] Production values updated with new version
- [ ] Changes reviewed in PR
- [ ] Merged to main
- [ ] Synced in ArgoCD UI
- [ ] Verified pods running
- [ ] Application tested
- [ ] Monitoring checked

---

**Next**: Monitor your production deployment! 🚀
