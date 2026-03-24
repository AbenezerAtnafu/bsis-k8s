# Vault Secret Rotation Runbook

This runbook covers limited-scope rotation for BSIS secrets sourced from HashiCorp Vault and synced with External Secrets Operator (ESO).

## Rotation scope

Only these keys are in scheduled rotation:

- `DB_URL` credential material

`JWT_SECRET` is not in scheduled rotation. Rotate it only on explicit trigger events (incident response, compromise suspicion, or cryptographic policy change).
All other keys also rotate only on trigger events (incident, vendor rollover, or policy requirement).

## Prerequisites

- Vault KV v2 paths exist:
  - `kv/bsis/development/app`
  - `kv/bsis/staging/app`
  - `kv/bsis/production/app`
- ESO is healthy in namespace `external-secrets`
- `ExternalSecret` resources are synced in each app namespace

## Standard rotation procedure

1. Generate new values in your secret management process.
2. Update Vault secret at the target path.
3. Wait for ESO refresh interval (`1h`) or force a refresh by annotating `ExternalSecret`.
4. Restart application rollout to ensure processes load new values.
5. Validate service health and authentication paths.

## Example commands

Update Vault values:

```bash
vault kv patch kv/bsis/development/app DB_URL='postgres://...'
vault kv patch kv/bsis/staging/app DB_URL='postgres://...'
vault kv patch kv/bsis/production/app DB_URL='postgres://...'
```

Force refresh (optional):

```bash
kubectl -n app-development annotate externalsecret bsis-app-secrets force-sync="$(date +%s)" --overwrite
kubectl -n app-staging annotate externalsecret bsis-app-secrets force-sync="$(date +%s)" --overwrite
kubectl -n app-production annotate externalsecret bsis-app-secrets force-sync="$(date +%s)" --overwrite
```

Restart rollout:

```bash
kubectl -n app-development rollout restart deploy/bsis-app
kubectl -n app-staging rollout restart deploy/bsis-app
kubectl -n app-production rollout restart deploy/bsis-app
```

## Validation checklist

- `kubectl -n <ns> get secret bsis-app-secrets -o yaml` shows updated resource version.
- `kubectl -n <ns> describe externalsecret bsis-app-secrets` shows `Ready=True`.
- Application health probes remain green after restart.
- No auth failures in logs related to rotated credentials.

## Emergency rotation

- Rotate affected key immediately in Vault.
- Force ESO sync and restart impacted workloads.
- Revoke old credential at source system (DB/user/token provider).
- Record incident timeline and remediation actions.
