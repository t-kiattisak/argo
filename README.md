# Argo CD GitOps Production Configuration Repository

GitOps Configuration Repository for installing, customizing, and managing applications with Argo CD at a production-ready level.

---

## 📂 Repository Directory Structure

```bash
├── README.md                  # System guide and documentation
├── argocd-install/            # Manifests for installing and configuring Argo CD (HA/Local Mode)
│   ├── kustomization.yaml     # Kustomize bundling for Argo CD Helm Chart
│   ├── namespace.yaml         # Creates the 'argocd' namespace
│   ├── values-ha.yaml         # Helm Values for installing Argo CD in High Availability (HA) mode
│   └── values-local.yaml      # Lightweight Helm Values for local development and testing
├── infrastructure/            # Cluster-wide foundational services
│   ├── kustomization.yaml
│   ├── cert-manager/          # Automatic SSL/TLS Certificate management via Let's Encrypt
│   ├── ingress-nginx/         # Ingress Controller for routing external traffic
│   ├── external-secrets/      # Pulls Kubernetes Secrets from external Key Management Services (AWS/GCP/Vault)
│   └── sealed-secrets/        # Alternative: Encrypts secrets inside Git (for non-cloud environments)
└── apps/                      # Application deployment configurations
    ├── base/                  # Standard blueprint templates (Deployment, Service)
    │   └── my-api/
    ├── overlays/              # Environment-specific overlays (Staging, Production)
    │   └── my-api-prod/       # Custom overrides for Production (e.g. replicas, resource limits)
    └── templates/             # ApplicationSet for automatic application discovery and sync
        └── appset-git-generator.yaml
```

---

## 🚀 Installation & Deployment Steps

### 1. Installing Argo CD in High Availability (HA) Mode

We use Kustomize to create the namespace and install the official Argo CD Helm chart with configurations optimized for production (`values-ha.yaml`).

```bash
# Verify resource generation before applying to the cluster
kubectl kustomize argocd-install/ --enable-helm

# Install to your Kubernetes cluster
kubectl apply -k argocd-install/ --enable-helm
```

*Note: In `argocd-install/values-ha.yaml`, the default admin user is disabled (`admin.enabled: "false"`) as a production security recommendation. Users should log in via SSO. For initial testing, you can change this value to `"true"` or retrieve the generated admin password with:*
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

### 2. Installing Infrastructure Components

Install the Ingress Controller, Cert-Manager, and Secrets Manager using a single Kustomize command:

```bash
kubectl apply -k infrastructure/
```

> 💡 **Recommendation**: If using AWS, GCP, or HashiCorp Vault, verify the IAM Roles for Service Accounts (IRSA) settings in `infrastructure/external-secrets/` to grant secure access to keys.

---

### 3. Application Onboarding

To deploy applications into the cluster, we recommend using the **ApplicationSet Pattern**, which automatically generates Argo CD Application resources when a new folder is added to `apps/overlays/`.

1. Install the ApplicationSet controller manifest onto your management cluster:
   ```bash
   kubectl apply -f apps/templates/appset-git-generator.yaml
   ```
2. Create folders following the template structure in `apps/base/my-api` and `apps/overlays/my-api-prod`.
3. Argo CD will automatically detect folders under `apps/overlays/*` and sync the application.

---

## 🔒 Production Security Best Practices

1. **Do Not Commit Plaintext Secrets**: Use `External Secrets Operator` to pull credentials from Cloud Key Vaults or use `Sealed Secrets` to encrypt secrets before committing to Git.
2. **SSO Integration**: Set up OIDC under the `argocd-cm` configuration in `argocd-install/values-ha.yaml` (example config templates are included inside the file).
3. **RBAC Policy**: Enforce granular role policies (e.g. limit developers to Read/Sync inside their project namespaces while admins hold cluster-wide privileges).
4. **Network Policies**: Limit network traffic to the Argo CD API server and Redis Sentinel to prevent public connection exposures.
5. **Git Connection**: When connecting to Git Providers (e.g., GitHub), use **GitHub App Authentication** instead of sharing SSH Keys to enforce tighter scopes and audit capabilities.
