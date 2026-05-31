# GitOps-Driven Kubernetes Delivery (ArgoCD & Helm)

## 📌 Project Overview
Architected a GitOps continuous delivery pipeline using ArgoCD and Helm on a local Kubernetes cluster with Git as a single source of truth, eliminating configuration drift and achieving automated self-healing deployments with zero manual intervention.

## 🛠️ Prerequisites
- **Docker:** To run the local Kubernetes cluster.
- **kubectl:** Kubernetes command-line tool.
- **Helm:** Kubernetes package manager.
- **Git:** Version control system.
- **kind (Kubernetes IN Docker):** To provision the local cluster.

## 🚀 Step-by-Step Implementation Guide

### Step 1: Provision the Local Kubernetes Cluster
Spin up a local multi-node cluster using `kind`.
```bash
kind create cluster --name gitops-cluster
kubectl cluster-info --context kind-gitops-cluster

```

### Step 2: Establish the Git Source of Truth

Create a standard Helm chart and push it to your GitHub repository.

```bash
# Generate a boilerplate Nginx application chart
helm create my-webapp

# Commit and push to the main branch
git add .
git commit -m "Initial Helm chart setup"
git push origin main

```

### Step 3: Install and Configure ArgoCD

Deploy ArgoCD directly into the local cluster to monitor the Git repository.

```bash
# Create a dedicated namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Port-forward the UI to localhost (Run in a separate terminal)
kubectl port-forward svc/argocd-server -n argocd 8080:443

```

**Login Credentials:**

* **Username:** `admin`
* **Password:** Retrieve the initial password using:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

```


*(Note: If using Windows CMD, omit the base64 decoding and decode it manually, or use a tool like Git Bash).*

### Step 4: Architect the GitOps Pipeline

Connect ArgoCD to the Git repository using a declarative `Application` Custom Resource.

Create a file named `application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-webapp-delivery
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/RohanShingmode/gitops-helm-config.git'
    targetRevision: HEAD
    path: my-webapp
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true      # Removes resources that are no longer in Git
      selfHeal: true   # Eliminates configuration drift

```

Apply the configuration:

```bash
kubectl apply -f application.yaml -n argocd

```

### Step 5: Optimize Local Polling Interval (Windows CMD)

By default, ArgoCD polls the Git repository every 3 minutes. For local development without webhooks, reduce the reconciliation timeout to 30 seconds to detect changes faster.

Run this specifically in **Windows CMD**:

```cmd
kubectl patch configmap argocd-cm -n argocd --type merge -p "{\"data\":{\"timeout.reconciliation\":\"30s\"}}"

# Restart the repo-server to apply changes
kubectl rollout restart deploy argocd-repo-server -n argocd

```

### Step 6: Validate Zero Manual Intervention & Self-Healing

**1. Test Automated Deployments:**

* Modify `replicaCount` in `my-webapp/values.yaml` (e.g., from 1 to 5).
* Commit and push to GitHub.
* ArgoCD will automatically detect the change within 30 seconds and scale the pods without manual `kubectl` intervention.

**2. Test Configuration Drift Elimination (Self-Healing):**

* Delete the running deployment manually: `kubectl delete deployment my-webapp`
* ArgoCD will instantly detect that the cluster state deviates from the Git state and automatically recreate the deployment.
