# Observability Retention Policy

This policy defines data retention for the BSIS observability stack.

## Current baseline (implemented)

- Metrics retention: `15d` (Prometheus).
- Alert data retention: `120h` (Alertmanager).

Configured in `platform/argocd/app-monitoring-stack.yaml`.

## Logging retention policy (to apply with Loki phase)

- Application logs retention: `7d`.
- Cluster/system logs retention: `7d`.
- Security and audit logs retention: `30d` (recommended minimum).

When Loki is deployed, implement this in Helm values using:

- `loki.limits_config.retention_period`
- `loki.compactor.retention_enabled`
- object storage lifecycle policy aligned with the same durations

## Tracing retention target

- Traces retention target: `3d`.

## Review cadence

- Revalidate retention monthly for cost/compliance.
- Increase retention only with approved storage and budget impact.
