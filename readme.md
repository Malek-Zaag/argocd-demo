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

---

## 📊 Helm vs Kustomize Comparison

| Feature                     | Helm                       | Kustomize               |
| --------------------------- | -------------------------- | ----------------------- |
| **Approach**                | Templating (Go templates)  | Patching/Overlays       |
| **Complexity**              | Moderate (template syntax) | Lower (declarative)     |
| **Reusability**             | High (charts, repos)       | Medium (local overlays) |
| **Learning Curve**          | Steeper                    | Gentler                 |
| **Package Management**      | Yes (Helm repos)           | No (Git-based)          |
| **Configuration Overrides** | Values files               | Strategic merge patches |
| **YAML Duplication**        | Less (templates)           | More (base + patches)   |

### When to Use Each

**Choose Helm if you:**

- Need reusable charts to share across teams
- Want package management and versioning
- Prefer a template-based approach
- Leverage the Helm ecosystem

**Choose Kustomize if you:**

- Have project-specific configuration needs
- Prefer staying closer to native Kubernetes YAML
- Want minimal learning curve
- Prefer strategic merge patches over templating

---

## 🔗 ArgoCD Commands Reference

```bash
argocd app create <appname>                 # Create a new application
argocd app list                             # List all applications
argocd app logs <appname>                   # Get application logs
argocd app get <appname>                    # Get application details
argocd app diff <appname>                   # Compare desired vs actual
argocd app sync <appname>                   # Manually sync with source
argocd app history <appname>                # View deployment history
argocd app rollback <appname>               # Rollback to previous version
argocd app set <appname> <flag>             # Set application configuration
argocd app delete <appname>                 # Delete an application
argocd app wait <appname>                   # Wait for app sync to complete
```

---

## 💡 GitOps Workflow

### With Helm

1. Update `values-*.yaml` files in Git
2. Commit and push changes
3. ArgoCD detects changes
4. Auto-syncs deployment (if automated)
5. Helm handles rolling updates

### With Kustomize

1. Update base YAML or overlay patches
2. Update `config.properties` or patches
3. Commit and push changes
4. ArgoCD detects changes
5. Kustomize renders manifests
6. kubectl applies updated resources

---

## 🧪 Testing & Validation

### Dry-Run Before Applying

**Helm**:

```bash
helm template webapp helm-webapp/ -f values-prod.yaml
helm install --dry-run --debug webapp helm-webapp/
```

**Kustomize**:

```bash
kubectl kustomize kustom-webapp/overlays/prod
kubectl apply -k kustom-webapp/overlays/prod --dry-run=client
```

## Happy experimenting with ArgoCD! 🚀

🧠 Argo CD + Helm + Kustomize — What to Remember
1️⃣ Helm repo vs Git repo (root cause of your first error)

❌ This is wrong for a Git repo:

```
chart: webapp
repoURL: https://github.com/...
```

✅ Correct for Git:

```
repoURL: https://github.com/...
path: webapp
```

Rule:

chart: → Helm repository (needs index.yaml)

path: → Git repository

2️⃣ Namespace creation in Argo CD

Argo does NOT create namespaces by default

You must opt in:

syncOptions:

- CreateNamespace=true

⚠️ Important:

It only works if the namespace does not already exist

default will never be created (it already exists)

✅ Best practice:

One app → one namespace

Manage the Namespace in Git when possible

3️⃣ automated, prune, selfHeal — what they really do
automated:
prune: true
selfHeal: true

automated → no manual sync

prune → delete resources removed from Git

selfHeal → revert manual kubectl changes

⚠️ Don’t mix with:

argocd.argoproj.io/sync-options: Prune=confirm

Pick automation OR confirmation, not both.

4️⃣ Helm + ReplicaSets (this one is critical)
revisionHistoryLimit truth
revisionHistoryLimit: 1

➡️ Keeps 1 old ReplicaSet on purpose
Seeing one old RS = expected behavior

When ReplicaSets become truly orphaned

Orphans happen when Deployment identity changes:

❌ Dangerous (never do this):

selector:
matchLabels:
env: {{ .Values.env }}

✅ Correct (selectors must be immutable):

selector:
matchLabels:
app.kubernetes.io/name: webapp
app.kubernetes.io/instance: {{ .Release.Name }}

Rule:
👉 Never template selectors from values.

5️⃣ One environment = one Argo Application

❌ Wrong:

values-dev.yaml
values-prod.yaml

✅ Correct:

app-dev → values-dev.yaml

app-prod → values-prod.yaml

Rule:
Never deploy multiple environments from one Argo Application.

6️⃣ Kustomize & namespace behavior

Kustomize does not auto-create namespaces

Argo won’t invent one for you

✅ Best practice:

apiVersion: v1
kind: Namespace
metadata:
name: my-app-dev

Commit it. Git owns it.

7️⃣ Kustomize + ConfigMaps not pruned (very important)
Why old ConfigMaps stick around

Kustomize uses name suffix hashing:

app-config-9c7b7t9f4k
app-config-kd82hd82j

This is intentional.

Argo will NOT prune them, even with:

prune: true

How to fix it
Option A (most common)
generatorOptions:
disableNameSuffixHash: true

Option B (pure GitOps)

Define ConfigMaps explicitly (no generator)

8️⃣ Big GitOps rules to live by

🔒 Immutability

Deployment selectors never change

📦 Ownership

Argo only prunes what Git explicitly owns

🌍 Isolation

One app

One env

One namespace

🧹 Prune is not garbage collection

Kubernetes controllers (RS, CM) have their own lifecycle

🔥 Final mental model (remember this)

Argo CD enforces Git state.
Helm & Kustomize generate Kubernetes behavior.
Kubernetes keeps history on purpose.

If something isn’t pruned, it’s usually:

Generated (not declared)

Controller-owned

Or intentionally preserved for safety
