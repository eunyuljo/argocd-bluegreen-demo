# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Blue-Green deployment demonstration project using ArgoCD and Argo Rollouts. The project showcases GitOps-based deployments with visual confirmation (Blue vs Green colored HTML pages) to make deployment strategies easy to understand, especially for Kubernetes beginners.

**Key Features:**
- No build required (uses public nginx:1.25-alpine image)
- ArgoCD-based GitOps workflow
- Argo Rollouts Blue-Green deployment strategy
- Visual color-based version identification
- Simple, educational structure

## Architecture

### Deployment Flow

```
Git Repository (manifests)
    ↓
ArgoCD (GitOps sync)
    ↓
Kubernetes Cluster
    ↓
Argo Rollout (Blue-Green strategy)
    ↓
Services (Active & Preview)
```

### Key Components

**Rollout Resource** (`manifests/rollout.yaml`):
- Controls the Blue-Green deployment strategy
- Manages 3 replicas with nginx containers
- Uses ConfigMaps to mount version-specific HTML
- References two services: `rollouts-demo-active` (production) and `rollouts-demo-preview` (testing)
- `autoPromotionEnabled: false` requires manual promotion via `kubectl argo rollouts promote`

**Services** (`manifests/services.yaml`):
- `rollouts-demo-active` (NodePort 30080): Production traffic endpoint
- `rollouts-demo-preview` (NodePort 30081): New version testing endpoint
- Both services route to pods with `app: rollouts-demo` label, but Argo Rollouts dynamically manages which pods each service targets

**ConfigMaps**:
- `rollouts-demo-blue`: Blue version HTML (blue background)
- `rollouts-demo-green`: Green version HTML (green background)
- Both contain self-reloading HTML pages (3-second auto-refresh)

**ArgoCD Application** (`argocd-app.yaml`):
- Defines the ArgoCD Application resource
- Points to Git repository path: `argocd-bluegreen-demo/manifests`
- Automated sync with prune and selfHeal enabled
- Must update `repoURL` to your own repository

## Common Commands

### Prerequisites Setup

```bash
# Install Argo Rollouts
helm repo add argo https://argoproj.github.io/argo-helm
helm install argo-rollouts argo/argo-rollouts -n argo-rollouts --create-namespace

# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Deploy Application

```bash
# Update repoURL in argocd-app.yaml first, then:
kubectl apply -f argocd-bluegreen-demo/argocd-app.yaml

# Verify deployment
kubectl get all -n rollouts-demo
kubectl argo rollouts get rollout rollouts-demo -n rollouts-demo
```

### Access Services

```bash
# Access production (Active) service
kubectl port-forward svc/rollouts-demo-active -n rollouts-demo 8888:80
# Browser: http://localhost:8888

# Access preview service
kubectl port-forward svc/rollouts-demo-preview -n rollouts-demo 8889:80
# Browser: http://localhost:8889
```

### Blue-Green Deployment Workflow

```bash
# 1. Deploy Green version (update rollout to use green ConfigMap)
kubectl patch rollout rollouts-demo -n rollouts-demo --type merge -p '{
  "spec": {
    "template": {
      "metadata": {
        "labels": {
          "version": "green"
        }
      },
      "spec": {
        "volumes": [{
          "name": "html",
          "configMap": {
            "name": "rollouts-demo-green"
          }
        }]
      }
    }
  }
}'

# 2. Test on preview service (port 8889)
# Active service still shows Blue

# 3. Promote to production
kubectl argo rollouts promote rollouts-demo -n rollouts-demo

# 4. Verify active service now shows Green (port 8888)
# Old Blue pods are scaled down after 30 seconds (scaleDownDelaySeconds)
```

### Monitoring and Troubleshooting

```bash
# Check rollout status
kubectl argo rollouts get rollout rollouts-demo -n rollouts-demo
kubectl argo rollouts status rollouts-demo -n rollouts-demo

# Watch rollout progress
kubectl argo rollouts get rollout rollouts-demo -n rollouts-demo --watch

# View rollout history
kubectl argo rollouts history rollout rollouts-demo -n rollouts-demo

# Rollout dashboard (visual UI)
kubectl argo rollouts dashboard
# Browser: http://localhost:3100

# Check ArgoCD sync status
kubectl get application rollouts-bluegreen-demo -n argocd

# View pod logs
kubectl logs -n rollouts-demo -l app=rollouts-demo

# Describe rollout for debugging
kubectl describe rollout rollouts-demo -n rollouts-demo

# Restart rollout if needed
kubectl argo rollouts restart rollouts-demo -n rollouts-demo

# Abort an in-progress rollout
kubectl argo rollouts abort rollouts-demo -n rollouts-demo

# Rollback to previous version
kubectl argo rollouts undo rollouts-demo -n rollouts-demo
```

## Important Configuration Notes

### ArgoCD Application Configuration

Before deploying, you MUST update `argocd-bluegreen-demo/argocd-app.yaml`:
- Change `repoURL` from `https://github.com/YOUR_USERNAME/argo-rollouts-demo.git` to your actual Git repository
- Ensure the `path` matches your repository structure
- For this specific setup, the path should be: `argocd-bluegreen-demo/manifests`

### Switching Between Versions

To switch versions, modify the `rollout.yaml` configMap reference:
- Blue: `name: rollouts-demo-blue`
- Green: `name: rollouts-demo-green`

The GitOps way is to commit changes to Git and let ArgoCD sync automatically. Alternatively, use `kubectl patch` for immediate testing.

### Blue-Green Strategy Settings

In `manifests/rollout.yaml`, the strategy section controls behavior:
- `autoPromotionEnabled: false` - Requires manual promotion (recommended for production)
- `scaleDownDelaySeconds: 30` - Time before old version pods are terminated
- To enable auto-promotion: Set `autoPromotionEnabled: true` and add `autoPromotionSeconds: 60`

## Development Workflow

### GitOps-Based Updates

1. Modify manifests in Git repository
2. Commit and push changes
3. ArgoCD automatically detects changes (if syncPolicy is automated)
4. New version deploys to preview service
5. Test on preview endpoint
6. Manually promote to active service

### Local Testing Without Git

For quick testing without Git commits:
1. Use `kubectl patch` to modify the Rollout directly
2. Changes are temporary and will be overridden by ArgoCD sync
3. Useful for rapid iteration during development

## Language and Documentation

This project contains Korean language documentation in README files. The key technical terms:
- 네임스페이스 = namespace
- 배포 = deployment
- 운영 = production
- 테스트 = testing

When modifying documentation, maintain bilingual context where present.

## Cleanup

```bash
# Delete ArgoCD application
kubectl delete -f argocd-bluegreen-demo/argocd-app.yaml

# Delete namespace (removes all resources)
kubectl delete namespace rollouts-demo

# Uninstall Argo Rollouts
helm uninstall argo-rollouts -n argo-rollouts

# Uninstall ArgoCD
kubectl delete namespace argocd
```
