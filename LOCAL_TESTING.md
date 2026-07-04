# Local Testing Guide for Argo CD (with Docker/Kubernetes)

Testing this GitOps repository on your local machine is easy using **Docker Desktop** or **Kind (Kubernetes in Docker)**. We have prepared a lightweight configuration file (`values-local.yaml`) for local testing.

---

## 🛠️ Prerequisites

1. **Docker Desktop** installed and running.
2. **Local Kubernetes Cluster**:
   * **Option 1 (Recommended for Mac/Windows)**: Enable Kubernetes in Docker Desktop (Go to Settings -> Kubernetes -> Check "Enable Kubernetes" -> Click "Apply & Restart").
   * **Option 2 (For Terminal users)**: Install [Kind](https://kind.sigs.k8s.io/) and create a cluster:
     ```bash
     kind create cluster --name argo-test
     ```
3. **kubectl**: CLI tool for managing Kubernetes.
4. **Helm**: CLI tool for downloading and managing Helm charts.

---

## 🚀 Testing Steps

### 1. Install Argo CD Locally (Reduced specs to save RAM)

Open your Terminal in this project directory and run:

```bash
# 1. Create a namespace for Argo CD
kubectl create namespace argocd

# 2. Add and update the Argo Helm Repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 3. Install Argo CD using the local config (disables HA and enables Admin User)
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  -f argocd-install/values-local.yaml
```

Verify that all Pods are up and running:
```bash
kubectl get pods -n argocd
```

---

## 🔓 Clean Up Resources

Once you are done testing and want to reclaim RAM/CPU resources:

```bash
# Delete the test application namespace
kubectl delete namespace my-api-prod

# Uninstall Argo CD
helm uninstall argocd -n argocd
kubectl delete namespace argocd

# If using Kind, you can delete the cluster directly
kind delete cluster --name argo-test
```
