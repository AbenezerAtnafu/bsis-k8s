# BSIS CCE Kubernetes Baseline

Baseline deployment template for Huawei CCE using Kustomize overlays and Argo CD GitOps.

## Structure

- `apps/<service>/base`: reusable manifests per service
- `apps/<service>/overlays/development|staging|production`: service environment overlays
- `apps/env-bootstrap/overlays/development|staging|production`: namespace and quota manifests
- `platform/argocd`: Argo CD project, platform apps, and service ApplicationSet
- `.github/workflows`: image build/push and promotion workflows
- `docs`: operational and security guidance

**Services:** bsis-app, core-api, identity-service, devices-integrations-api, main-client, facility-client, rabbitmq

## Prerequisites

- CCE cluster and Argo CD installed in namespace `argocd`
- GHCR package path available (`ghcr.io/habtec/bsis-app`)
- GitHub Actions permissions include `packages: write`
- `kubectl` and `kustomize` for local validation

## How to Deploy

### 1. Bootstrap (one-time setup)

1. Confirm repo URL in `platform/argocd/*.yaml` points to your GitHub repo.
2. Confirm GHCR path (`ghcr.io/habtec/bsis-app`) in `apps/bsis-app/overlays/*/kustomization.yaml`.
3. Set your ACME email in `platform/cert-manager-config/clusterissuer-letsencrypt-prod.yaml`.
4. If needed, set CCE ELB annotations in `platform/argocd/app-ingress-nginx.yaml`.
5. Create namespace secrets (see `docs/secrets-and-rbac.md`).
6. Apply Argo CD resources:

```bash
kubectl apply -k platform/argocd
```

### 2. GitOps deployment flow

After bootstrap, deployments are driven by Git:

1. **Development**: Pushing to `main` triggers the build workflow, which builds the image, pushes to GHCR, and updates `apps/bsis-app/overlays/development/kustomization.yaml`. Argo CD syncs automatically.

2. **Staging / Production**: Use the **Promote Image Tag** workflow (see [How to Push to GitHub](#how-to-push-to-github)) to promote a specific image tag. Argo CD syncs after the commit.

3. Argo CD `ApplicationSet` discovers `apps/*/overlays/{development|staging|production}` and creates one Argo Application per service/environment.

4. Argo CD syncs generated applications to namespaces `app-development`, `app-staging`, `app-production`.

### 3. Manual sync (optional)

```bash
# Sync all applications
argocd app sync --async --prune

# Sync a specific service
argocd app sync bsis-app-development
```

## How to Push to GitHub

### Initial setup (first time)

1. **Create a new repository** on GitHub (e.g. `HABTec/bsis-k8s`).

2. **Initialize and push** (if starting from local):

```bash
git init
git remote add origin https://github.com/HABTec/bsis-k8s.git
git add .
git commit -m "chore: initial baseline"
git branch -M main
git push -u origin main
```

3. **Configure repository permissions** in GitHub:
   - Settings → Actions → General → Workflow permissions → "Read and write permissions"

### Regular workflow

#### Deploy to Development (automatic on push)

```bash
git add .
git commit -m "feat: your changes"
git push origin main
```

The **Build Push and Update Development Overlay** workflow runs on push to `main`:
- Builds the image (context: repo root; requires `Dockerfile`)
- Pushes to GHCR with tags: `<SHA>`, `development-latest`
- Updates `apps/bsis-app/overlays/development/kustomization.yaml` with the new tag
- Commits and pushes the overlay update (triggers Argo CD sync)

#### Promote to Staging or Production (manual trigger)

1. Go to **Actions** → **Promote Image Tag** → **Run workflow**
2. Select **target_environment**: `staging` or `production`
3. Enter **image_tag**: the immutable tag to deploy (e.g. Git SHA from development)
4. Run the workflow

The workflow updates the target overlay and pushes the change; Argo CD syncs automatically.

## Local validation

```bash
# Validate all services
kustomize build apps/bsis-app/overlays/development
kustomize build apps/bsis-app/overlays/staging
kustomize build apps/bsis-app/overlays/production
kustomize build apps/identity-service/overlays/development
kustomize build apps/identity-service/overlays/staging
kustomize build apps/identity-service/overlays/production
kustomize build apps/core-api/overlays/development
kustomize build apps/core-api/overlays/staging
kustomize build apps/core-api/overlays/production
kustomize build apps/devices-integrations-api/overlays/development
kustomize build apps/devices-integrations-api/overlays/staging
kustomize build apps/devices-integrations-api/overlays/production
kustomize build apps/main-client/overlays/development
kustomize build apps/main-client/overlays/staging
kustomize build apps/main-client/overlays/production
kustomize build apps/facility-client/overlays/development
kustomize build apps/facility-client/overlays/staging
kustomize build apps/facility-client/overlays/production
kustomize build apps/rabbitmq/overlays/development
kustomize build apps/rabbitmq/overlays/staging
kustomize build apps/rabbitmq/overlays/production
kustomize build apps/env-bootstrap/overlays/development
kustomize build apps/env-bootstrap/overlays/staging
kustomize build apps/env-bootstrap/overlays/production
kustomize build platform/argocd
```

## References

- `docs/secrets-and-rbac.md`
- `docs/secrets-config-strategy.md`
- `docs/gitops-service-onboarding.md`
- `docs/vault-in-cluster-bootstrap.md`
- `docs/vault-rotation-runbook.md`
- `docs/operations-baseline.md`
- `docs/observability-retention-policy.md`
