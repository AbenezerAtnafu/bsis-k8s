# Secrets and Configuration Strategy

This document defines how BSIS handles non-sensitive configuration and sensitive secrets in a GitOps model.

## Scope

- Environments: `app-development`, `app-staging`, `app-production`
- Platform namespaces: `argocd`, `ingress-nginx`, `cert-manager`, `monitoring`, `external-secrets`, `vault`
- Applies to application workloads and platform integrations managed from this repository

## Principles

- Keep sensitive values out of Git.
- Keep non-sensitive configuration in Git and version it per environment.
- Use least privilege for service accounts and human access.
- Prefer auditable rotation over ad-hoc changes.

## Configuration management (non-sensitive)

Store non-sensitive configuration in environment overlays as Kubernetes manifests.

Recommended examples:

- Feature flags
- Public endpoints
- Timeouts and tuning values
- Log levels

Guidelines:

- Keep defaults in base only when safe for all environments.
- Override environment-specific values in `apps/overlays/development|staging|production`.
- Use explicit keys and avoid embedding opaque JSON blobs when possible.

## Secret management (sensitive)

Sensitive values include:

- Database credentials and connection secrets
- API keys and tokens
- JWT signing keys
- Third-party webhook secrets
- TLS private keys (when not managed by cert-manager)

Current baseline in this repo:

- Workloads read `bsis-app-secrets` via `envFrom.secretRef`.
- Secrets are created directly in cluster namespaces (documented in `docs/secrets-and-rbac.md`).

Target model (phase 2):

- Use self-hosted HashiCorp Vault in-cluster as the central secret backend.
- Use External Secrets Operator (ESO) to sync from Vault into Kubernetes Secrets.
- Keep only `SecretStore` / `ExternalSecret` metadata in Git.
- Keep real secret values in Vault.
- Default in-cluster endpoint: `http://vault.vault.svc:8200` (bootstrap mode).

## GitOps boundaries

- Allowed in Git:
  - `ConfigMap` values that are non-sensitive
  - secret references (`secretRef`, `ExternalSecret` spec)
  - placeholders and templates with no real credentials
- Not allowed in Git:
  - plaintext production credentials
  - long-lived private tokens
  - private keys outside approved secret systems

## Naming conventions

- App secret consumed by workload: `bsis-app-secrets`
- TLS secret: `bsis-app-<env>-tls`
- Config map (recommended): `bsis-app-config`

Keep names stable across environments to simplify manifests and automation.

## Rotation policy

- Rotate only a controlled subset by default:
  - `DB_URL` credential material: every 90 days (or use dynamic DB credentials)
- `JWT_SECRET` is excluded from scheduled rotation and rotates only on trigger events.
- Other secrets rotate on change trigger (provider key rollover, incident, or compliance event).
- Trigger rollout/restart of affected workloads after rotation.
- Record rotation owner, date, and rollback plan in change ticket or release notes.

## RBAC and access control

- Application service accounts should not have API permissions to list/read arbitrary secrets.
- If API-based secret read is required, grant `get` only for named resources.
- Restrict production secret access to a small ops group and enforce approval workflow.

## Operational checks

- Validate secret presence before deployment sync windows.
- Alert on pod crash loops caused by missing secret keys.
- Periodically scan Git history and CI logs for accidental secret disclosure.
- Use `docs/vault-in-cluster-bootstrap.md` for initial setup and Kubernetes auth wiring.
- Use `docs/vault-rotation-runbook.md` for execution steps.
