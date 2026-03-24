# Day-1 Secrets and RBAC

This baseline uses Kubernetes Secrets for fast onboarding. Keep secret values out of Git and create them directly in each environment namespace.

For long-term direction (config + secrets model), see `docs/secrets-config-strategy.md`.
For Vault rotation steps, see `docs/vault-rotation-runbook.md`.

## 1) Create secrets per environment

Update keys to match your application contract.

```bash
kubectl -n app-development create secret generic bsis-app-secrets \
  --from-literal=DB_URL='postgres://user:pass@db-development:5432/app' \
  --from-literal=JWT_SECRET='replace-me'

kubectl -n app-staging create secret generic bsis-app-secrets \
  --from-literal=DB_URL='postgres://user:pass@db-staging:5432/app' \
  --from-literal=JWT_SECRET='replace-me'

kubectl -n app-production create secret generic bsis-app-secrets \
  --from-literal=DB_URL='postgres://user:pass@db-production:5432/app' \
  --from-literal=JWT_SECRET='replace-me'
```

If the secret exists, update with `kubectl apply -f -` from a generated manifest or delete/recreate during controlled maintenance windows.

## 2) Bind workload identity

The deployment runs as service account `bsis-app`. Pods consume secret values via `envFrom.secretRef` and do not need direct Kubernetes API permissions to read secrets.

## 3) RBAC boundary recommendations

- Do not grant default service accounts any secret permissions.
- If a job/tool must read secrets through API, create a dedicated service account and least-privilege Role for only required secret names.
- Restrict human access to production secrets using cluster RBAC groups.

Example least-privilege Role (only if API reads are required):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader-bsis-app
  namespace: app-development
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["bsis-app-secrets"]
    verbs: ["get"]
```

## 4) Rotation policy

- Rotate high-impact DB credentials on a fixed schedule.
- Rotate `JWT_SECRET` only on explicit trigger events (incident response, compromise suspicion, or cryptographic policy change).
- Roll deployments after rotation to ensure new values are loaded.
- Plan migration to External Secrets in phase 2.
