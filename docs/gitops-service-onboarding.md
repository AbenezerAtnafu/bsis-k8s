# Service Onboarding Standard (ApplicationSet)

This repository uses an Argo CD `ApplicationSet` to generate one Argo `Application` per service and environment.

## Directory pattern

Every deployable service must follow this structure:

- `apps/<service-name>/base`
- `apps/<service-name>/overlays/development`
- `apps/<service-name>/overlays/staging`
- `apps/<service-name>/overlays/production`

Example:

- `apps/identity-service/base`
- `apps/identity-service/overlays/development`
- `apps/identity-service/overlays/staging`
- `apps/identity-service/overlays/production`

## Required overlay content

Each environment overlay should include:

- environment-specific image tag in `kustomization.yaml`
- deployment/ingress/hpa patches
- `SecretStore` and `ExternalSecret` for that service

Do not add service workloads under shared `apps/overlays/*` paths.

## Naming conventions

- Kubernetes app name: `app.kubernetes.io/name=<service-name>`
- Image: `ghcr.io/habtec/<service-image>`
- Environment tags:
  - `development-latest`
  - `staging-latest`
  - `production-latest`
- Immutable release tag: full Git SHA

## Argo CD behavior

- `platform/argocd/applicationset-services.yaml` discovers:
  - `apps/*/overlays/development`
  - `apps/*/overlays/staging`
  - `apps/*/overlays/production`
- Generated Argo app name: `<service>-<environment>`
- Destination namespace: `app-<environment>`

## Adding a new service checklist

1. Create `apps/<service-name>/base` manifests.
2. Create `apps/<service-name>/overlays/{development,staging,production}`.
3. Set image tags in each overlay `kustomization.yaml`.
4. Add service SecretStore and ExternalSecret per environment.
5. Run validation:
  - `kubectl kustomize apps/<service-name>/overlays/development`
  - `kubectl kustomize apps/<service-name>/overlays/staging`
  - `kubectl kustomize apps/<service-name>/overlays/production`
6. Commit and let ApplicationSet reconcile automatically.

## RabbitMQ (shared messaging)

In-cluster RabbitMQ is deployed per environment via `platform/argocd/applicationset-rabbitmq.yaml`. Each environment (development, staging, production) gets its own RabbitMQ instance in the corresponding `app-<env>` namespace.

- **Service DNS**: `rabbitmq` (same namespace) or `rabbitmq.app-<env>.svc.cluster.local`
- **AMQP port**: 5672
- **Management UI port**: 15672 (when management plugin is enabled)

**Vault bootstrap** (required before first sync): Ensure `bsis/<env>/shared` contains:
- `RABBITMQ_PASSWORD` – used by RabbitMQ and by all consuming services
- `RABBITMQ_USERNAME` – set to `user` (must match Helm `auth.username`)

Host and port (`rabbitmq` / `5672`) are configured in ConfigMaps.

**Vault Kubernetes auth**: Configure Vault roles `rabbitmq-development`, `rabbitmq-staging`, `rabbitmq-production` for the `rabbitmq` ServiceAccount in each namespace.

**Services using RabbitMQ**: core-api, identity-service, devices-integrations-api.

---

## devices-integrations

HL7 device interface service (Alinity CI, etc.) that receives lab results and forwards to Core API and RabbitMQ.

- **Image**: `ghcr.io/habtec/bsis-devices-integrations`
- **Port**: 5601 (HL7 MLLP)
- **Config**: `config.properties` built from ConfigMap + ExternalSecret (init container runs `envsubst`)
- **Vault paths** (create before deployment):
  - `bsis/<env>/devices-integrations`: `AUTH_CLIENT_SECRET` (analyzer OAuth client secret)
  - `bsis/<env>/shared`: `DB_PASSWORD`, `RABBITMQ_USERNAME`, `RABBITMQ_PASSWORD`
- **Kubernetes auth**: configure Vault role `devices-integrations-<env>` for the service account.
