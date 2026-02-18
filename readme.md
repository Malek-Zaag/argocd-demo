# ArgoCD Experimentation: Helm vs Kustomize

This project demonstrates how to use ArgoCD with two different Kubernetes manifest management approaches: **Helm** and **Kustomize**. Both methods are implemented for deploying the same web application across different environments (dev, prod).

**Reference**: [ArgoCD Boss Tutorial](https://youtu.be/JLrR9RV9AFA)

---

## 📁 Project Structure

```
argo-examples/
├── helm-webapp/                    # Helm chart approach
│   ├── Chart.yaml                 # Helm chart metadata
│   ├── values.yaml                # Base values for all environments
│   ├── values-dev.yaml            # Dev environment overrides
│   ├── values-prod.yaml           # Prod environment overrides
│   └── templates/                 # Kubernetes resource templates
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── configmap.yaml
│       └── NOTES.txt
│
├── kustom-webapp/                 # Kustomize approach
│   ├── base/                      # Base layer (shared by all envs)
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   │
│   └── overlays/                  # Environment-specific patches
│       ├── dev/
│       │   ├── config.properties
│       │   ├── kustomization.yaml
│       │   └── replicas.yaml
│       │
│       └── prod/
│           ├── config.properties
│           ├── kustomization.yaml
│           └── replicas.yaml
│
├── 1-application.yaml             # ArgoCD Application manifest (Helm)
├── 2-application.yaml             # ArgoCD Application manifest (Kustomize)
└── readme.md                      # This file
```

---

## 🎯 Overview

### What This Demonstrates

- **Helm Approach**: Template-based configuration management with values overrides
- **Kustomize Approach**: Declarative patching system for managing environment variants
- **ArgoCD Integration**: GitOps deployment of both approaches to Kubernetes
- **Multi-Environment Setup**: Separate dev and prod configurations for each approach

### Application Details

The sample application (`my-sample-app`) is a simple web application with:

- Image: `my-sample-app:latest`
- Port: 8080 (containerPort) → 80 (service port)
- Resource requests: 50m CPU, 16Mi memory
- Resource limits: 100m CPU, 128Mi memory
- Configuration: Environment variables from ConfigMap

---

## 🚀 Quick Start: ArgoCD Setup

### 1. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Access the ArgoCD UI

```bash
# Forward ArgoCD server port
kubectl port-forward service/argocd-server -n argocd 8080:443

# In another terminal, get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Open in browser: https://127.0.0.1:8080
```

### 3. Install ArgoCD CLI (Optional)

```bash
brew install argocd

# Login via CLI
kubectl port-forward svc/argocd-server -n argocd 8080:443
argocd login 127.0.0.1:8080
```

---

## 📦 Approach 1: Helm

### Key Configuration Files

**Base Values** (`values.yaml`):

```yaml
appName: myhelmapp
port: 80
configmap:
  name: myhelmapp-configmap-v1
  data:
    COLOR: "RED"
image:
  name: my-sample-app
  tag: latest
```

**Production Override** (`values-prod.yaml`):

```yaml
replicas: 12
configmap:
  data:
    COLOR: "PROD v3 Color"
```

### Deploy with Helm

#### Locally Render Templates

```bash
# Render with base values
helm template webapp helm-webapp/

# Render with prod values
helm template -f helm-webapp/values-prod.yaml webapp helm-webapp/
```

#### Deploy Directly (without ArgoCD)

```bash
# Deploy to prod namespace
helm install webapp-prod helm-webapp/ -n prod --create-namespace -f helm-webapp/values-prod.yaml

# Upgrade deployment
helm upgrade webapp-prod helm-webapp/ -n prod -f helm-webapp/values-prod.yaml
```

---

## 🔧 Approach 2: Kustomize

### Key Configuration Files

**Base Kustomization** (`base/kustomization.yaml`):

```yaml
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  app: kustomwebapp
namePrefix: kustom-
nameSuffix: -v1
```

**Dev Overlay** (`overlays/dev/kustomization.yaml`):

```yaml
bases:
  - ../../base
namespace: app2
patches:
  - path: replicas.yaml
configMapGenerator:
  - name: mykustom-map
    env: config.properties
```

### Deploy with Kustomize

#### Locally Render Manifests

```bash
# Render dev overlay
kubectl kustomize kustom-webapp/overlays/dev

# Render prod overlay
kubectl kustomize kustom-webapp/overlays/prod
```

#### Deploy Directly (without ArgoCD)

```bash
# Deploy dev overlay
kubectl apply -k kustom-webapp/overlays/dev

# Deploy prod overlay
kubectl apply -k kustom-webapp/overlays/prod
```

### Deploy Applications to ArgoCD

#### Via CLI

```bash
# Create Helm application
argocd app create webapp-helm-prod \
  --repo https://github.com/YOUR-REPO/argo-examples.git \
  --path helm-webapp \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace prod

# Create Kustomize application
argocd app create webapp-kustom-dev \
  --repo https://github.com/YOUR-REPO/argo-examples.git \
  --path kustom-webapp/overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace app2
```

#### Via Manifest

```bash
# Apply ArgoCD applications
kubectl apply -f 1-application.yaml
kubectl apply -f 2-application.yaml
```

### Monitor Applications

```bash
# List all applications
argocd app list

# Get application details
argocd app get webapp-helm-prod
argocd app get kustomize-example

# Compare desired vs actual state
argocd app diff webapp-helm-prod

# Manually sync application
argocd app sync webapp-helm-prod

# View sync history
argocd app history webapp-helm-prod

# Rollback to previous version
argocd app rollback webapp-helm-prod
```
