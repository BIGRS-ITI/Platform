# Task Manager Application - Multi-Image GitOps Deployment

## Architecture Overview

This application consists of **3 components** deployed on EKS via GitOps:

```
┌─────────────────┐
│   Frontend      │  ← Nginx serving static HTML/CSS/JS
│  (Port 8080)    │  ← Image: taskmanager-frontend
└────────┬────────┘
         │
         ↓
┌─────────────────┐     ┌─────────────┐     ┌──────────────┐
│   Backend API   │────→│    Redis    │     │  RDS MySQL   │
│  (Port 3000)    │     │  (Cache)    │     │  (Database)  │
│  Node.js/Express│     │  Port 6379  │     │  Port 3306   │
└─────────────────┘     └─────────────┘     └──────────────┘
  taskmanager-backend    redis:7-alpine     AWS RDS (Terraform)
```

## Infrastructure Components

### 1. **ECR Repositories** (Terraform Managed)
- `taskmanager-backend` - Node.js API container
- `taskmanager-frontend` - Nginx static file server

### 2. **RDS MySQL Database** (Terraform Managed)
- **Endpoint**: Managed by Terraform (output: `rds_endpoint`)
- **Database**: `mydb`
- **User**: `admin`
- **Password**: `password123`
- **Instance**: `db.t3.micro`
- **Storage**: 20GB encrypted

### 3. **Redis Cache** (Kubernetes Managed)
- **Image**: `redis:7-alpine`
- **Persistence**: AWS EBS gp2 5Gi PVC
- **Service**: `redis.taskmanager.svc.cluster.local:6379`

## Deployment Flow

```
┌──────────────┐
│  Developer   │
│  git push    │
└──────┬───────┘
       │
       ↓
┌──────────────────┐
│  Jenkins Build   │ ← Jenkinsfile (dual-image pipeline)
│  - Build Backend │
│  - Build Frontend│
│  - Push to ECR   │
└──────┬───────────┘
       │
       ↓
┌────────────────────────┐
│  ECR Repositories      │
│  - Backend: build #N   │
│  - Frontend: build #N  │
└──────┬─────────────────┘
       │
       ↓
┌────────────────────────┐
│ ArgoCD Image Updater   │ ← Detects new images
│ - Watches ECR          │
│ - Updates manifests    │
└──────┬─────────────────┘
       │
       ↓
┌────────────────────────┐
│  ArgoCD Applications   │
│  - backend-app.yaml    │
│  - frontend-app.yaml   │
│  - redis-app.yaml      │
└──────┬─────────────────┘
       │
       ↓
┌────────────────────────┐
│  EKS Cluster           │
│  Namespace: taskmanager│
└────────────────────────┘
```

## Setup Instructions

### Step 1: Deploy Infrastructure with Terraform

```bash
cd infrastructure/terraform/environment/dev

# Initialize Terraform
terraform init

# Plan (review changes)
terraform plan

# Apply infrastructure
terraform apply -auto-approve

# Get RDS endpoint
terraform output rds_endpoint
# Output example: bigrs-rds.cxxxxxxxxxx.us-east-1.rds.amazonaws.com:3306
```

### Step 2: Update Backend ConfigMap with RDS Endpoint

Edit `platform/apps/Backend/configmap.yaml`:

```yaml
data:
  DB_HOST: "bigrs-rds.cxxxxxxxxxx.us-east-1.rds.amazonaws.com"  # ← Update this
```

### Step 3: Push Platform Configuration to Git

```bash
cd platform
git add .
git commit -m "Add taskmanager backend, frontend, and redis apps"
git push origin main
```

### Step 4: Bootstrap ArgoCD

```bash
cd infrastructure/scripts

# Set environment variables
export CLUSTER_NAME="bigrs-cluster"
export CLUSTER_ENDPOINT=$(terraform -chdir=../terraform/environment/dev output -raw cluster_endpoint)
export AWS_REGION="us-east-1"
export GITHUB_PLATFORM_REPO="BIGRS-ITI/Platform"
export GITHUB_TOKEN="your_github_token"
export ARGOCD_NAMESPACE="argocd"

# Run bootstrap script
bash bootstrap-argocd.sh
```

### Step 5: Configure Jenkins Pipeline

1. **Access Jenkins**:
   ```bash
   kubectl port-forward -n jenkins svc/jenkins 8080:8080
   ```
   
2. **Get admin password**:
   ```bash
   kubectl get secret -n jenkins jenkins -o jsonpath='{.data.jenkins-admin-password}' | base64 -d
   ```

3. **Create Pipeline Job**:
   - New Item → Pipeline
   - Name: `taskmanager-build`
   - Pipeline from SCM → Git
   - Repository: Your nodejs_app repo
   - Script Path: `Jenkinsfile`

### Step 6: Trigger First Build

```bash
# Manually trigger Jenkins job or push to git
# This will build BOTH images in parallel
```

### Step 7: Verify Deployment

```bash
# Check ArgoCD applications
kubectl get applications -n argocd

# Check pods
kubectl get pods -n taskmanager

# Get frontend LoadBalancer URL
kubectl get svc -n taskmanager frontend
```

## Directory Structure

```
Platform/
├── apps/
│   ├── Backend/
│   │   ├── backend-deployment.yaml    ← Deployment, Service, HPA
│   │   ├── configmap.yaml              ← RDS connection config
│   │   └── secrets.yaml                ← DB password
│   ├── Frontend/
│   │   └── frontend-deployment.yaml    ← Deployment, Service, HPA
│   └── Redis/
│       └── redis-deployment.yaml       ← Deployment, Service, PVC
└── argo-apps/
    ├── backend-app.yaml               ← ArgoCD app with image-updater
    ├── frontend-app.yaml              ← ArgoCD app with image-updater
    └── redis-app.yaml                 ← ArgoCD app (static image)
```

## Jenkins Pipeline Details

The `Jenkinsfile` builds **both images in parallel**:

### Parallel Build Stages:
1. **Build Backend** (Dockerfile.backend)
2. **Build Frontend** (Dockerfile.frontend)

### Image Tags:
- `taskmanager-backend:${BUILD_NUMBER}`
- `taskmanager-backend:latest`
- `taskmanager-frontend:${BUILD_NUMBER}`
- `taskmanager-frontend:latest`

### ECR Push:
Both images pushed to:
- `608713827966.dkr.ecr.us-east-1.amazonaws.com/taskmanager-backend`
- `608713827966.dkr.ecr.us-east-1.amazonaws.com/taskmanager-frontend`

## ArgoCD Image Updater Configuration

### Backend App Annotations:
```yaml
argocd-image-updater.argoproj.io/image-list: backend=608713827966.dkr.ecr.us-east-1.amazonaws.com/taskmanager-backend
argocd-image-updater.argoproj.io/backend.update-strategy: latest
argocd-image-updater.argoproj.io/backend.allow-tags: regexp:^[0-9]+$
```

### Frontend App Annotations:
```yaml
argocd-image-updater.argoproj.io/image-list: frontend=608713827966.dkr.ecr.us-east-1.amazonaws.com/taskmanager-frontend
argocd-image-updater.argoproj.io/frontend.update-strategy: latest
argocd-image-updater.argoproj.io/frontend.allow-tags: regexp:^[0-9]+$
```

## Environment Variables (Backend)

The backend container receives configuration from **ConfigMap** and **Secrets**:

### From ConfigMap:
- `NODE_ENV=production`
- `PORT=3000`
- `DB_HOST=<RDS_ENDPOINT>`
- `DB_PORT=3306`
- `DB_USER=admin`
- `DB_NAME=mydb`
- `REDIS_HOST=redis.taskmanager.svc.cluster.local`
- `REDIS_PORT=6379`
- `CACHE_TTL=300`

### From Secrets:
- `DB_PASSWORD=password123`
- `REDIS_PASSWORD=` (empty, no auth)

## Database Connection

The backend connects to **AWS RDS MySQL**:
- **Host**: From Terraform output (`rds_endpoint`)
- **Port**: 3306
- **Security**: RDS security group only allows EKS nodes
- **Encryption**: Storage encrypted at rest

## Redis Configuration

- **Persistence**: Enabled (AOF + RDB)
- **Storage**: AWS EBS gp2 5Gi PVC
- **Backup**: Every 60 seconds if at least 1 write

## Scaling

### Backend:
- **Min**: 2 replicas
- **Max**: 10 replicas
- **Trigger**: CPU > 70% or Memory > 80%

### Frontend:
- **Min**: 2 replicas
- **Max**: 5 replicas
- **Trigger**: CPU > 70% or Memory > 80%

### Redis:
- **Replicas**: 1 (stateful, single instance)

## Health Checks

### Backend:
- **Liveness**: `GET /api/health` (30s delay, 10s interval)
- **Readiness**: `GET /api/health` (10s delay, 5s interval)

### Frontend:
- **Liveness**: `GET /` (10s delay, 10s interval)
- **Readiness**: `GET /` (5s delay, 5s interval)

### Redis:
- **Liveness**: `redis-cli ping` (30s delay, 10s interval)
- **Readiness**: `redis-cli ping` (5s delay, 5s interval)

## Troubleshooting

### Check Backend Logs:
```bash
kubectl logs -n taskmanager -l app=backend --tail=100 -f
```

### Check RDS Connection:
```bash
kubectl exec -it -n taskmanager deployment/backend -- sh
# Inside pod:
nc -zv $DB_HOST 3306
```

### Check Redis Connection:
```bash
kubectl exec -it -n taskmanager deployment/backend -- sh
# Inside pod:
nc -zv redis.taskmanager.svc.cluster.local 6379
```

### Verify ConfigMap Values:
```bash
kubectl describe configmap -n taskmanager backend-config
```

### Check ArgoCD Sync Status:
```bash
kubectl get application -n argocd taskmanager-backend -o yaml
kubectl get application -n argocd taskmanager-frontend -o yaml
```

## Important Notes

1. **Database Password**: Currently hardcoded in RDS module as `password123`. For production, use AWS Secrets Manager.

2. **Redis is NOT RedisInsight**: The production deployment uses standard Redis. RedisInsight is only in docker-compose for local development.

3. **Image Updates**: Automatic via ArgoCD Image Updater. Every Jenkins build triggers a deployment.

4. **RDS Endpoint**: Must be manually copied from Terraform outputs to `backend-config` ConfigMap.

5. **Namespace**: All app components deploy to `taskmanager` namespace (auto-created by ArgoCD).

## Next Steps

1. ✅ Update `DB_HOST` in configmap.yaml after Terraform apply
2. ✅ Push platform changes to GitHub
3. ✅ Run bootstrap-argocd.sh
4. ✅ Create Jenkins pipeline job
5. ✅ Trigger first build
6. ✅ Access application via Frontend LoadBalancer URL
