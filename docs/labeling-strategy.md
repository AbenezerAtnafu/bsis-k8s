# BSIS Kubernetes Labeling Strategy

This document defines a consistent Kubernetes labeling standard for BSIS workloads.

## Goals

- Make workload ownership and purpose clear.
- Keep selectors stable and predictable.
- Improve filtering in `kubectl`, Argo CD, and observability tools.

## Required labels

Apply these labels to every workload resource (`Deployment`, `Service`, `HPA`, `Ingress`, `PDB`, `ServiceAccount`, `NetworkPolicy`).

- `app.kubernetes.io/name`: workload name (for example `identity-service`).
- `app.kubernetes.io/component`: logical component (`identity`, `api`, `frontend`).
- `app.kubernetes.io/part-of`: platform name (`bsis`).
- `app.kubernetes.io/managed-by`: deployment manager (`argocd`).
- `app.kubernetes.io/environment`: deployment environment (`development`, `staging`, `production`).
- `app.kubernetes.io/instance`: unique workload instance in an environment (for example `identity-service-development`).

## BSIS service mapping

Use the following canonical names:

| Service | `app.kubernetes.io/name` | `app.kubernetes.io/component` |
|---|---|---|
| Identity Service | `identity-service` | `identity` |
| Core API | `core-api` | `api` |
| Main BSIS Frontend | `main-bsis-frontend` | `frontend` |
| Facility Frontend | `facility-frontend` | `frontend` |

## Naming conventions

Use lowercase kebab-case for all names. Avoid underscores, spaces, and uppercase letters.

- Service slug format: `<service-name>` (for example `core-api`).
- Environment format: `development`, `staging`, `production`.
- Instance format: `<service-name>-<env>` (for example `core-api-production`).

### Kubernetes resource names

- Namespace: `app-<env>` (for example `app-development`).
- Deployment/Service/HPA/Ingress/PDB/ServiceAccount base name: `<service-name>`.
- Secret consumed by workload: `<service-name>-secrets`.
- NetworkPolicy name: `<service-name>-default-deny-ingress`.

### Argo CD and path naming

- Argo CD application name: `<service-name>-<env>`.
- Overlay path: `apps/<service-name>/overlays/<env>`.
- Base path: `apps/<service-name>/base`.

### DNS and TLS naming

- Host format: `<service-name>-<env>.example.com` for non-production.
- Prod host format: `<service-name>.example.com`.
- TLS secret format: `<service-name>-<env>-tls` (production can be `<service-name>-production-tls` for consistency).

### Example mapping

| Service | development instance | staging instance | production instance |
|---|---|---|---|
| Identity Service | `identity-service-development` | `identity-service-staging` | `identity-service-production` |
| Core API | `core-api-development` | `core-api-staging` | `core-api-production` |
| Main BSIS Frontend | `main-bsis-frontend-development` | `main-bsis-frontend-staging` | `main-bsis-frontend-production` |
| Facility Frontend | `facility-frontend-development` | `facility-frontend-staging` | `facility-frontend-production` |

## Selector guidance

- Use `app.kubernetes.io/name` as the primary selector key.
- Do not use environment labels in `Service` selectors unless required.
- Keep selector labels immutable after first deployment.

## Example

```yaml
metadata:
  labels:
    app.kubernetes.io/name: core-api
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: bsis
    app.kubernetes.io/managed-by: argocd
    app.kubernetes.io/environment: production
    app.kubernetes.io/instance: core-api-production
```
