# Platform GitOps Repository

## Overview

This repository contains all Kubernetes manifests and ArgoCD applications for the platform. It follows GitOps principles where the Git repository is the single source of truth for the desired state of the cluster.

## Architecture

```text
┌────────────────────────────────────────────────────────┐
│                   ArgoCD Bootstrap                     │
│  (Deployed by Terraform from Infrastructure repo)      │
└──────────────────────┬─────────────────────────────────┘
                       │
                       ↓ (Monitors Platform repo)
┌────────────────────────────────────────────────────────┐
│              Platform Repository (This Repo)           │
│  ┌────────────────────────────────────────────────┐    │
│  │ argo-apps/ - Application Definitions           │    │
│  │  ├── cert-manager.yaml                         │    │
│  │  ├── nginx-ingress-controller.yaml             │    │
│  │  ├── jenkins-app.yaml                          │    │
│  │  ├── external-secrets-app.yaml                 │    │
│  │  ├── external-secrets-secret-store.yaml        │    │
│  │  ├── image-updater-app.yaml                    │    │
│  │  ├── ecr-token-refresher.yaml                  │    │
│  │  └── nodejs-app.yaml                           │    │
│  └────────────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────────────┐    │
│  │ helm-values/ - Helm Chart Customizations       │    │
│  │  ├── cert-manager-values.yaml                  │    │
│  │  ├── cluster_issuer.yaml                       │    │
│  │  ├── nginx-values.yaml                         │    │
│  │  ├── jenkins-values.yaml                       │    │
│  │  ├── external-secrets-values.yaml              │    │
│  │  └── image-updater-values.yaml                 │    │
│  └────────────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────────────┐    │
│  │ apps/ - Application Manifests                  │    │
│  │  ├── ecr-token-refresher/                      │    │
│  │  └── nodejs-app/                               │    │
│  └────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────┘
                       │
                       ↓ (ArgoCD syncs and deploys)
┌───────────────────────────────────────────────────────┐
│                    EKS Cluster                        │
│  ┌────────┐  ┌────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Cert   │  │ Nginx  │  │ Jenkins  │  │ External │   │
│  │Manager │  │Ingress │  │          │  │ Secrets  │   │
│  └────────┘  └────────┘  └──────────┘  └──────────┘   │
│  ┌────────┐  ┌────────┐  ┌──────────┐                 │
│  │ Image  │  │  ECR   │  │ NodeJS   │                 │
│  │Updater │  │ Token  │  │   App    │                 │
│  └────────┘  └────────┘  └──────────┘                 │
└───────────────────────────────────────────────────────┘
```

## Directory Structure

```text
Platform/
├── README.md                           # This file
│
├── argo-apps/                          # ArgoCD Application definitions
│   ├── cert-manager.yaml               # TLS certificate management
│   ├── nginx-ingress-controller.yaml   # Ingress controller
│   ├── jenkins-app.yaml                # CI/CD server
│   ├── external-secrets-app.yaml       # Secrets management operator
│   ├── external-secrets-secret-store.yaml  # Secret store config
│   ├── image-updater-app.yaml          # Automated image updates
│   ├── ecr-token-refresher.yaml        # ECR credentials refresh
│   └── nodejs-app.yaml                 # Task manager application
│
├── helm-values/                        # Helm chart values files
│   ├── cert-manager-values.yaml        # Cert-manager configuration
│   ├── cluster_issuer.yaml             # Let's Encrypt issuer
│   ├── nginx-values.yaml               # Nginx ingress configuration
│   ├── jenkins-values.yaml             # Jenkins configuration
│   ├── external-secrets-values.yaml    # External secrets config
│   └── image-updater-values.yaml       # Image updater configuration
│
└── apps/                               # Application manifests
    ├── ecr-token-refresher/            # ECR auth token refresh job
    │   ├── ecr-token-refresher.yaml
    │   └── kustomization.yaml
    │
    └── nodejs-app/                     # Task manager app
        ├── Backend/                    # Backend deployment
        ├── Frontend/                   # Frontend deployment
        ├── Redis/                      # Redis cache
        ├── kustomization.yaml
        └── serviceaccount.yaml
```

## Components

### Infrastructure Components (Sync Wave 0-3)

| Component | Sync Wave | Purpose | Type |
|-----------|-----------|---------|------|
| **cert-manager** | 0 | Manages TLS certificates for HTTPS | Helm Chart |
| **nginx-ingress** | 3 | Routes external traffic to services | Helm Chart |

### Platform Services (Sync Wave 4-5)

| Component | Sync Wave | Purpose | Type |
|-----------|-----------|---------|------|
| **jenkins** | 4 | CI/CD pipeline automation | Helm Chart |
| **external-secrets** | 4 | Syncs secrets from AWS Secrets Manager | Helm Chart |
| **external-secrets-secret-store** | 5 | Configuration for secret provider | Manifest |
| **image-updater** | 5 | Auto-updates container images | Helm Chart |
| **ecr-token-refresher** | 5 | Refreshes ECR credentials (CronJob) | Manifest |

### Applications

| Component | Purpose | Type |
|-----------|---------|------|
| **nodejs-app** | Task manager application | Kustomize |

## Sync Waves

ArgoCD deploys applications in order based on sync wave annotations:

```text
Wave 0: cert-manager (TLS foundation)
  ↓
Wave 3: nginx-ingress (traffic routing)
  ↓
Wave 4: jenkins, external-secrets (platform services)
  ↓
Wave 5: image-updater, ecr-token-refresher, secret-store (automation)
  ↓
Default: nodejs-app (applications)
```

## Key Features

### Multi-Source Applications

Several applications use ArgoCD's multi-source feature to combine Helm charts with custom values from Git:

- **cert-manager**: Helm chart + custom values
- **nginx-ingress**: Helm chart + custom values
- **jenkins**: Helm chart + custom values
- **image-updater**: Helm chart + custom values

**Example Pattern:**

```yaml
sources:
  # Source 1: Helm Chart from upstream repo
  - repoURL: https://charts.jetstack.io
    chart: cert-manager
    targetRevision: v1.16.1
    helm:
      valueFiles:
        - $values/helm-values/cert-manager-values.yaml
  
  # Source 2: Values file from this Git repo
  - repoURL: https://github.com/BIGRS-ITI/Platform.git
    targetRevision: main
    ref: values
```

### Automated Image Updates

The **argocd-image-updater** watches ECR for new image tags and automatically updates application manifests:

- Checks ECR every 2 minutes
- Updates image tags in Git
- ArgoCD syncs changes automatically
- Enabled for nodejs-app backend/frontend

### ECR Token Refresh

The **ecr-token-refresher** ensures ArgoCD can pull images from ECR:

- Initial Job: Runs immediately on deployment
- CronJob: Refreshes token every 6 hours
- Creates `ecr-credentials` secret in argocd namespace
- Used by image-updater for ECR access

### External Secrets Operator

Syncs secrets from AWS Secrets Manager to Kubernetes:

- Uses EKS Pod Identity for AWS authentication
- Configured via SecretStore CR
- Automatically creates K8s secrets from AWS secrets
- No manual secret management needed

## Application Details

### cert-manager

**Purpose:** Automates TLS certificate issuance and renewal  
**Namespace:** cert-manager  
**Chart:** jetstack/cert-manager v1.16.1  
**Features:**

- CRD installation
- 2 replicas for HA
- Prometheus metrics enabled
- 10s webhook timeout

**Resources:**

- CPU: 100m request
- Memory: 128Mi request

### nginx-ingress-controller

**Purpose:** Routes HTTP/HTTPS traffic to services  
**Namespace:** ingress-nginx  
**Chart:** ingress-nginx/ingress-nginx v4.10.1  
**Features:**

- AWS NLB with IP targets
- 2-5 replicas (HPA enabled)
- Cross-zone load balancing
- Pod anti-affinity for HA
- TLSv1.2/1.3 only

**Resources:**

- CPU: 100m-500m
- Memory: 128Mi-512Mi

### jenkins

**Purpose:** CI/CD automation server  
**Namespace:** jenkins  
**Chart:** jenkins/jenkins v5.8.108  
**Features:**

- JDK 21 LTS image
- 20Gi persistent storage (gp2)
- JCasC configuration
- Kubernetes cloud with docker-in-docker
- Pod Identity for ECR access

**Resources:**

- CPU: 500m-2 cores
- Memory: 512Mi-2Gi
- Java heap: 512m-2Gi

**Agents:**

- Docker build capability
- AWS CLI included
- Ephemeral volumes

### external-secrets

**Purpose:** Syncs AWS Secrets Manager → Kubernetes  
**Namespace:** external-secrets  
**Chart:** external-secrets/external-secrets v0.9.9  
**Features:**

- CRD installation
- Pod Identity with IAM role
- Automatic secret sync
- Info-level logging

**Resources:**

- CPU: 100m request
- Memory: 128Mi request

### argocd-image-updater

**Purpose:** Auto-updates container images  
**Namespace:** argocd  
**Chart:** argo/argocd-image-updater v0.14.0  
**Features:**

- 2-minute check interval
- ECR registry configured
- Uses ecr-credentials secret
- Pod Identity for AWS access
- Health and metrics endpoints

**Resources:**

- CPU: 100m-200m
- Memory: 128Mi-256Mi

**Security:**

- Non-root user (UID 1000)
- Read-only root filesystem
- No privilege escalation

### ecr-token-refresher

**Purpose:** Maintains ECR authentication  
**Namespace:** argocd  
**Type:** Job + CronJob  
**Features:**

- Initial job on deployment
- Runs every 6 hours via CronJob
- Creates docker-registry secret
- Uses Pod Identity
- Installs AWS CLI v2

**Resources:**

- CPU: 100m-500m
- Memory: 256Mi-512Mi

### nodejs-app

**Purpose:** Task manager application  
**Namespace:** taskmanager  
**Type:** Kustomize  
**Components:**

- Backend (Node.js + Express)
- Frontend (Static HTML/CSS/JS)
- Redis (cache)
- MySQL (database - managed by RDS)

**Features:**

- Pod Identity for ECR image pull
- Ingress for external access
- ConfigMaps and Secrets
- Health checks

## Usage

### Deploying Applications

Applications are automatically deployed by ArgoCD once the bootstrap app is configured. To manually sync:

```bash
# Sync all applications
argocd app sync -l argocd.argoproj.io/instance=bootstrap

# Sync specific application
argocd app sync cert-manager
argocd app sync nodejs-app
```

### Viewing Application Status

```bash
# List all applications
kubectl get applications -n argocd

# View specific application details
argocd app get jenkins

# Watch sync progress
argocd app wait jenkins --health
```

### Updating Configurations

1. Edit the appropriate file in this repository
2. Commit and push to main branch
3. ArgoCD auto-syncs within 3 minutes
4. Or manually sync: `argocd app sync <app-name>`

### Adding New Applications

1. Create application manifest in `argo-apps/`
2. Add sync wave annotation if order matters
3. Add helm values to `helm-values/` if using Helm
4. Commit and push
5. ArgoCD deploys automatically

## Image Update Automation

Applications can opt-in to automated image updates using annotations:

```yaml
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: backend=608713827966.dkr.ecr.us-east-1.amazonaws.com/nodejs-backend
    argocd-image-updater.argoproj.io/backend.update-strategy: latest
    argocd-image-updater.argoproj.io/write-back-method: git
```

## Secret Management

Secrets are managed via External Secrets Operator:

1. Store secret in AWS Secrets Manager
2. Create ExternalSecret CR referencing the AWS secret
3. Operator syncs → creates Kubernetes Secret
4. Pods consume via environment variables or volumes

**Example:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: mysql-password
spec:
  secretStoreRef:
    name: aws-secretsstore
  target:
    name: mysql-secret
  data:
    - secretKey: password
      remoteRef:
        key: prod/mysql/password
```

## Troubleshooting

### Application Not Syncing

```bash
# Check application status
argocd app get <app-name>

# View sync errors
kubectl describe application <app-name> -n argocd

# Force refresh
argocd app get <app-name> --refresh
```

### Image Updater Not Working

```bash
# Check updater logs
kubectl logs -n argocd deployment/argocd-image-updater

# Verify ECR credentials
kubectl get secret ecr-credentials -n argocd

# Check ECR token refresher
kubectl get jobs,cronjobs -n argocd
kubectl logs -n argocd job/ecr-token-refresher-init
```

### External Secrets Issues

```bash
# Check operator status
kubectl get pods -n external-secrets

# View secret store
kubectl get secretstore -n external-secrets

# Check external secret status
kubectl describe externalsecret <name> -n <namespace>
```

### Certificate Issues

```bash
# Check cert-manager pods
kubectl get pods -n cert-manager

# View certificate status
kubectl get certificate -A

# Check cluster issuer
kubectl describe clusterissuer letsencrypt-prod
```

## Technologies

- **ArgoCD** - GitOps continuous deployment
- **Helm** - Package manager for Kubernetes
- **Kustomize** - Template-free configuration
- **cert-manager** - Certificate management
- **External Secrets Operator** - Secret sync from AWS
- **NGINX Ingress** - Ingress controller
- **Jenkins** - CI/CD automation

## Best Practices

1. **Never commit secrets** - Use External Secrets Operator
2. **Use sync waves** - Control deployment order
3. **Enable auto-sync** - For continuous deployment
4. **Add health checks** - For proper status reporting
5. **Tag resources** - For organization and tracking
6. **Use multi-source** - Separate Helm charts from values
7. **Version Helm charts** - Pin to specific versions
8. **Monitor sync status** - Set up alerts for failed syncs

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [External Secrets Operator](https://external-secrets.io/)
- [cert-manager Documentation](https://cert-manager.io/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Jenkins Helm Chart](https://github.com/jenkinsci/helm-charts)
