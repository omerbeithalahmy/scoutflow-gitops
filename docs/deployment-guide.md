# Deployment Guide

Step-by-step instructions for deploying ScoutFlow using ArgoCD.

## Prerequisites

✅ EKS cluster running  
✅ ArgoCD installed on cluster  
✅ `kubectl` configured  
✅ AWS ECR access  
✅ Docker images available in ECR

---

## Step 1: Access ArgoCD UI

### Get Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### Port Forward

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Login

- URL: https://localhost:8080
- Username: `admin`
- Password: (from command above)

---

## Step 2: Create ECR Credentials

### Generate ECR Password

```bash
aws ecr get-login-password --region us-east-1
```

### Create Secrets

```bash
# Staging
kubectl create secret generic ecr-credentials \
  --from-literal=password="<PASSWORD>" \
  -n staging

# Production
kubectl create secret generic ecr-credentials \
  --from-literal=password="<PASSWORD>" \
  -n production
```

> **Note**: ECR passwords expire every 12 hours. For production, use [external-secrets-operator](https://external-secrets.io/).

---

## Step 3: Deploy Staging

### Apply ArgoCD Application

```bash
kubectl apply -f argocd/apps/scoutflow-staging.yaml
```

### Verify in ArgoCD UI

1. Refresh ArgoCD UI
2. You should see `scoutflow-staging` application
3. It will auto-sync immediately
4. Status should show "Healthy" and "Synced"

### Check Pods

```bash
kubectl get pods -n staging
```

**Expected output**:
```
NAME                                 READY   STATUS    RESTARTS   AGE
scoutflow-backend-xxxxxx             1/1     Running   0          2m
scoutflow-frontend-xxxxxx            1/1     Running   0          2m
scoutflow-db-0                       1/1     Running   0          2m
```

### Access Application

```bash
# Get ALB hostname
kubectl get ingress -n staging

# Open in browser (may take 2-3 minutes for ALB to provision)
```

---

## Step 4: Deploy Production

### Apply ArgoCD Application

```bash
kubectl apply -f argocd/apps/scoutflow-production.yaml
```

### Manual Sync in UI

1. Open ArgoCD UI
2. Find `scoutflow-production` application
3. Click "SYNC" button
4. Review changes in the diff view
5. Click "SYNCHRONIZE"

### Verify Deployment

```bash
kubectl get pods -n production
```

**Expected output** (note 2 replicas):
```
NAME                                 READY   STATUS    RESTARTS   AGE
scoutflow-backend-xxxxxx             1/1     Running   0          2m
scoutflow-backend-yyyyyy             1/1     Running   0          2m
scoutflow-frontend-xxxxxx            1/1     Running   0          2m
scoutflow-frontend-yyyyyy            1/1     Running   0          2m
scoutflow-db-0                       1/1     Running   0          2m
```

---

## Troubleshooting

### Application Shows "OutOfSync"

**Cause**: Git state differs from cluster state

**Solution**:
- Click "SYNC" in ArgoCD UI
- Or wait (staging auto-syncs)

### Pods in "ImagePullBackOff"

**Cause**: Can't pull from ECR

**Solution**:
```bash
# Recreate ECR secret
ECR_PASSWORD=$(aws ecr get-login-password --region us-east-1)
kubectl delete secret ecr-credentials -n <namespace>
kubectl create secret generic ecr-credentials \
  --from-literal=password="$ECR_PASSWORD" \
  -n <namespace>

# Delete pod to force restart
kubectl delete pod <pod-name> -n <namespace>
```

### Database Not Starting

**Cause**: PVC not created

**Solution**:
```bash
# Check PVC
kubectl get pvc -n <namespace>

# Check storage class exists
kubectl get storageclass

# Ensure gp2 (staging) or gp3 (production) exists
```

---

## Next Steps

- [Promotion Workflow](promotion-workflow.md) - Deploy new versions to production
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
