# ArgoCD
ABC of ArgoCD | ArgoCD 101

# Complete ArgoCD Guide

## Table of Contents
- [What is ArgoCD?](#what-is-argocd)
- [Architecture](#architecture)
- [Installation](#installation)
- [Getting Started](#getting-started)
- [Application Management](#application-management)
- [Sync Strategies](#sync-strategies)
- [Troubleshooting](#troubleshooting)
- [CLI Reference](#cli-reference)
- [Resources](#resources)

## What is ArgoCD?

ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. ArgoCD automates the deployment of applications to specified target environments by continuously monitoring Git repositories and synchronizing the live state with the desired state.

### Key Benefits
- **Declarative**: Application definitions, configurations, and environments are declarative and version controlled
- **Automated**: Application deployment and lifecycle management is automated, auditable, and easy to understand
- **Scalable**: Supports multiple clusters and thousands of applications
- **Secure**: Built-in security features including RBAC, SSO, and audit logging
- **GitOps Native**: Follows GitOps principles with Git as the single source of truth

## Architecture

ArgoCD consists of several components:

### ArgoCD Server
- Web UI and API server
- Handles authentication and RBAC
- Manages application definitions
- Serves the web interface

### Repository Server
- Maintains local cache of Git repositories
- Generates Kubernetes manifests from various sources
- Handles Helm, Kustomize, and plain YAML

### Application Controller
- Kubernetes controller that monitors applications
- Compares live state vs desired state
- Performs sync operations
- Updates application status and health

### Redis
- Caching layer for repository data
- Session storage for the web UI

## Installation

### Prerequisites
- Kubernetes cluster (1.20+)
- kubectl configured to access your cluster
- At least 2GB RAM and 1 CPU core available

### Install ArgoCD

#### Method 1: Using kubectl
```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

#### Method 2: Using Helm
```bash
# Add ArgoCD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD
helm install argocd argo/argo-cd -n argocd --create-namespace
```

### Access ArgoCD UI

#### Port Forward (Development)
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Access at: https://localhost:8080

#### Load Balancer (Production)
```yaml
# argocd-server-service-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: argocd-server-lb
  namespace: argocd
spec:
  type: LoadBalancer
  ports:
  - name: https
    port: 443
    targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server
```

### Get Initial Admin Password
```bash
# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

## Getting Started

### 1. Login via CLI
```bash
# Install ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Login to ArgoCD
argocd login localhost:8080
```

### 2. Create Your First Application

#### Via CLI
```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

#### Via YAML
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 3. Sync Application
```bash
# Sync application
argocd app sync guestbook

# Check application status
argocd app get guestbook
```

## Application Management

### Application Lifecycle

#### Creating Applications
Applications can be created through:
- Web UI
- CLI commands
- YAML manifests
- ApplicationSets (for multiple applications)

#### Application States
- **Healthy**: All resources are running as expected
- **Progressing**: Application is being deployed or updated
- **Degraded**: Some resources are not healthy
- **Suspended**: Application sync is paused

#### Sync States
- **Synced**: Live state matches desired state
- **OutOfSync**: Live state differs from desired state
- **Unknown**: Sync state cannot be determined

### Managing Applications

#### View Applications
```bash
# List all applications
argocd app list

# Get detailed application info
argocd app get <app-name>

# View application history
argocd app history <app-name>
```

#### Sync Operations
```bash
# Manual sync
argocd app sync <app-name>

# Sync with prune (remove extra resources)
argocd app sync <app-name> --prune

# Dry run sync
argocd app sync <app-name> --dry-run

# Force sync (ignore differences)
argocd app sync <app-name> --force
```

#### Rollback Applications
```bash
# Rollback to previous version
argocd app rollback <app-name>

# Rollback to specific revision
argocd app rollback <app-name> <revision-id>
```

## Sync Strategies

### Manual Sync
Applications require manual intervention to sync changes.

```yaml
spec:
  syncPolicy: {}  # No automated sync policy
```

### Automated Sync
ArgoCD automatically syncs when changes are detected.

```yaml
spec:
  syncPolicy:
    automated:
      prune: true      # Remove resources not in Git
      selfHeal: true   # Revert manual changes
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
```

### Sync Hooks
Execute actions during sync operations.

```yaml
# Pre-sync hook example
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: migrate/migrate
        command: ["migrate", "-path", "/migrations", "-database", "$DATABASE_URL", "up"]
```

### Sync Waves
Control the order of resource deployment.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Deploy in wave 1
```

## Troubleshooting

### Common Issues

#### Application Stuck in Progressing State
```bash
# Check application events
kubectl describe application <app-name> -n argocd

# Check controller logs
kubectl logs deployment/argocd-application-controller -n argocd

# Force refresh
argocd app get <app-name> --hard-refresh
```

#### Sync Failures
```bash
# Check sync operation details
argocd app get <app-name> --show-operation

# View resource differences
argocd app diff <app-name>

# Check for resource hooks
kubectl get all -l app.kubernetes.io/instance=<app-name>
```

#### Repository Connection Issues
```bash
# Test repository connection
argocd repo get <repo-url>

# Update repository credentials
argocd repo add <repo-url> --username <user> --password <pass> --upsert
```

#### Performance Issues
```bash
# Check resource usage
kubectl top pods -n argocd

# Increase controller resources
kubectl patch deployment argocd-application-controller -n argocd -p '{"spec":{"template":{"spec":{"containers":[{"name":"application-controller","resources":{"requests":{"memory":"2Gi","cpu":"1000m"}}}]}}}}'
```

### Debug Commands
```bash
# Enable debug logging
kubectl patch configmap argocd-cmd-params-cm -n argocd --patch '{"data":{"controller.log.level":"debug"}}'

# View detailed application info
argocd app get <app-name> -o yaml

# Check application history
argocd app history <app-name> --output wide

# View live resources
argocd app resources <app-name> --orphaned
```

## CLI Reference

### Application Commands
```bash
# Create application
argocd app create <name> --repo <repo> --path <path> --dest-server <server>

# List applications
argocd app list

# Get application details
argocd app get <name>

# Sync application
argocd app sync <name>

# Delete application
argocd app delete <name>

# Set application parameters
argocd app set <name> --parameter key=value

# View application logs
argocd app logs <name>

# Wait for sync
argocd app wait <name>
```

### Repository Commands
```bash
# Add repository
argocd repo add <repo-url>

# List repositories
argocd repo list

# Remove repository
argocd repo rm <repo-url>
```

### Cluster Commands
```bash
# Add cluster
argocd cluster add <context>

# List clusters
argocd cluster list

# Remove cluster
argocd cluster rm <server>
```

### Project Commands
```bash
# Create project
argocd proj create <name>

# List projects
argocd proj list

# Add repository to project
argocd proj add-source <project> <repo>

# Add destination to project
argocd proj add-destination <project> <server> <namespace>
```

## Resources

### Official Documentation
- [ArgoCD Official Docs](https://argo-cd.readthedocs.io/)
- [ArgoCD GitHub Repository](https://github.com/argoproj/argo-cd)
- [ArgoCD Examples](https://github.com/argoproj/argocd-example-apps)

### Community Resources
- [ArgoCD Slack Community](https://argoproj.github.io/community/join-slack)
- [CNCF ArgoCD Landscape](https://landscape.cncf.io/)
- [ArgoCD Blog Posts and Tutorials](https://blog.argoproj.io/)

### Related Projects
- **Argo Workflows**: Container-native workflow engine
- **Argo Events**: Event-based dependency manager
- **Argo Rollouts**: Progressive delivery controller
- **ApplicationSet**: Multi-application management

### Books and Learning
- "GitOps and Kubernetes" by Billy Yuen, Alexander Matyushentsev, Todd Ekenstam, Jesse Suen
- "Learning ArgoCD" by Akram Riahi
- Kubernetes documentation on GitOps patterns

---

This README provides a comprehensive guide to ArgoCD. For the most up-to-date information, always refer to the official ArgoCD documentation and releases.
