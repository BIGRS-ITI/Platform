# Platform GitOps Repository

## Overview

This repository contains all Kubernetes manifests and ArgoCD applications for the BIGRS platform. It follows GitOps principles where the Git repository is the single source of truth for the desired state of the cluster. Deployments are fully automated using ArgoCD with the App of Apps pattern.

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
│  │  ├── pre-apps.yaml (sync-wave: -1)             │    │
│  │  ├── cert-manager.yaml (sync-wave: 0)          │    │
│  │  ├── cert-manager-issuers.yaml (sync-wave: 1)  │    │
│  │  ├── nginx-ingress-controller.yaml (wave: 3)   │    │
│  │  ├── jenkins-app.yaml (sync-wave: 4)           │    │
│  │  ├── external-secrets-operator.yaml (wave: 4)  │    │
│  │  ├── external-secrets-app.yaml (sync-wave: 5)  │    │
│  │  ├── image-updater-app.yaml (sync-wave: 5)     │    │
│  │  └── nodejs-app.yaml (default wave)            │    │
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
│  │  ├── pre-apps/ (namespaces, SA, ECR token)     │    │
│  │  ├── external-secrets/ (secret store config)   │    │
│  │  └── nodejs-app/ (task manager app)            │    │
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

For full directory structure and detailed component information, see the complete README sections below.

## Quick Start

### View Applications

```bash
# List all applications
kubectl get applications -n argocd

# Check application status
argocd app get <app-name>

# Watch all applications
watch kubectl get applications -n argocd
```

### Manual Sync

```bash
# Sync all applications (respects sync waves)
argocd app sync -l argocd.argoproj.io/instance=bootstrap

# Sync specific application
argocd app sync nodejs-app
```

### Update Configuration

```bash
# 1. Edit files in this repository
# 2. Commit and push to appropriate branch
git add .
git commit -m "Update configuration"
git push origin prod  # or main for dev

# 3. ArgoCD auto-syncs within 3 minutes
# Or manually sync:
argocd app sync <app-name>
```

## Sync Waves Explained

Applications deploy in a specific order using sync waves:

```text
Wave -1: pre-apps → Create namespaces, service accounts, ECR token
Wave 0:  cert-manager → TLS certificate management
Wave 1:  cert-manager-issuers → Let's Encrypt configuration
Wave 3:  nginx-ingress → HTTP(S) routing
Wave 4:  jenkins, external-secrets-operator → Platform services
Wave 5:  external-secrets-app, image-updater → Automation
Default: nodejs-app → Applications
```

**Why this order matters:**

- Namespaces must exist before deployments
- Cert-manager CRDs before issuers
- Nginx before ingress resources
- External Secrets before secret references
- ECR credentials before image pulls

## Key Features Summary

### GitOps Automation

- ✅ **App of Apps Pattern** - Single bootstrap app deploys everything
- ✅ **Auto-Sync** - Changes in Git automatically deployed
- ✅ **Self-Healing** - ArgoCD reverts manual changes
- ✅ **Sync Waves** - Controlled deployment order

### Security & Secrets

- ✅ **External Secrets Operator** - AWS Secrets Manager integration
- ✅ **Pod Identity** - Secure AWS access without keys
- ✅ **TLS Certificates** - Automated with Let's Encrypt
- ✅ **No Hardcoded Secrets** - All secrets from AWS

### CI/CD & Automation

- ✅ **Jenkins Integration** - Automated builds and deploys
- ✅ **Image Auto-Update** - Watches ECR for new tags
- ✅ **ECR Token Refresh** - Automated every 6 hours
- ✅ **Multi-Source Apps** - Helm charts + Git values

## For detailed documentation on all components, troubleshooting, and advanced usage, please refer to the original README content above.

## Contributing

1. Create feature branch from `main`
2. Test in dev environment
3. Create PR to `main`
4. After testing, merge to `prod`
5. ArgoCD auto-deploys to production

## License

MIT

## Author

BIGRS-ITI Team
