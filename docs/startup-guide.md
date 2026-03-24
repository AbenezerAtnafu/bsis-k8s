# BSIS-K8S Fresh Startup Guide

This guide provides a step-by-step walkthrough for setting up the BSIS Kubernetes environment from scratch on an Ubuntu OS.

## 1. Local Machine Setup
Ensure you have `kubectl`, `kustomize`, and `git` installed on your local development machine.

### Clone the Repository
```bash
git clone https://github.com/AbenezerAtnafu/bsis-k8s.git
cd bsis-k8s
```

## 2. Remote Server (Ubuntu) Installation
If you are working on a fresh Ubuntu instance (e.g., Huawei CCE ECS), follow these steps to install the necessary tools.

### Install kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Install kustomize
```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo install -o root -g root -m 0755 kustomize /usr/local/bin/kustomize
```

## 3. Argo CD Installation
Argo CD is the heart of the GitOps deployment. It must be installed before bootstrapping.

### Create Namespace
```bash
kubectl create namespace argocd
```

### Apply Installation Manifests
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Wait for Ready State & Install ApplicationSet CRDs
```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# If ApplicationSet resources fail to apply, manually install the CRD with server-side apply:
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/applicationset-crd.yaml --server-side
```

## 4. Configuration & Customization
Before bootstrapping, ensure the following files are updated with your specific environment details:

1.  **Repository URL**: Update `platform/argocd/*.yaml` to point to `https://github.com/AbenezerAtnafu/bsis-k8s.git`.
2.  **ACME Email**: Set your email in `platform/cert-manager-config/clusterissuer-letsencrypt-prod.yaml`.
3.  **ELB/EIP Info**: Update `platform/argocd/app-ingress-nginx.yaml` with your `kubernetes.io/elb.id`.

## 5. Kubernetes Secrets
Create the required application secrets manually in the target namespaces.

```bash
kubectl -n app-development create secret generic bsis-app-secrets \
  --from-literal=DB_URL='postgres://user:pass@db-development:5432/app' \
  --from-literal=JWT_SECRET='your-secure-secret'
```

## 6. Bootstrap the Platform
Once tools are installed and configuration is updated, apply the bootstrap manifests:

```bash
kubectl apply -k platform/argocd
```

## 7. Syncing & CLI Installation
To manage and sync your applications easily, install the `argocd` CLI.

### Install argocd CLI (Ubuntu)
```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

### Sync All Applications
Once installed, you can sync all applications at once:
```bash
# Get the admin password from the secret
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)

# Login (use localhost if port-forwarding or your ELB IP)
argocd login --core

# Sync all
argocd app sync --all --async --prune
```

### Access the Argo CD UI
To access the dashboard from your local browser:

1. **Start Port-Forwarding** (Run this in a separate terminal on your local machine if using SSH tunnel, or on the Ubuntu server):
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```

2. **Get the Admin Password**:
   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
   ```

3. **Open Browser**:
   Navigate to `https://localhost:8080` (Username: `admin`).

## 8. Verification
Verify that the applications are created and syncing in Argo CD:

```bash
kubectl get applications -n argocd
```
