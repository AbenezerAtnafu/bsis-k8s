# CCE Operations Baseline

## Namespace strategy

- Use one namespace per environment: `app-development`, `app-staging`, `app-production`.
- Keep environment-scoped resources in each namespace (Deployments, Services, HPAs, Secrets).
- Keep platform resources in dedicated namespaces (for example, `argocd`, `ingress-nginx`, `monitoring`).

## Labeling strategy

- Follow `docs/labeling-strategy.md` for canonical app labels.
- Follow the naming conventions in `docs/labeling-strategy.md` for service slugs, instance names, and resource names.
- Keep `app.kubernetes.io/name` stable because Service selectors and policy selectors depend on it.
- Use `app.kubernetes.io/environment` on workload pods for environment-aware filtering and dashboards.

## Autoscaling baseline

- Use both HPA and cluster autoscaler.
- Initial HPA targets are configured in base:
  - CPU average utilization: 70%
  - Memory average utilization: 75%
- Environment max replica defaults:
  - development: max 6
  - staging: max 8
  - production: max 12
- Keep `requests`/`limits` realistic because HPA and scheduler depend on them.

## Resource quota baseline

`ResourceQuota` is defined per environment overlay:

- `apps/overlays/development/resourcequota.yaml`
- `apps/overlays/staging/resourcequota.yaml`
- `apps/overlays/production/resourcequota.yaml`

Baseline hard limits:

- development: `requests.cpu=2`, `requests.memory=3Gi`, `limits.cpu=8`, `limits.memory=6Gi`
- staging: `requests.cpu=3`, `requests.memory=4Gi`, `limits.cpu=12`, `limits.memory=8Gi`
- production: `requests.cpu=6`, `requests.memory=8Gi`, `limits.cpu=20`, `limits.memory=16Gi`

Object count caps are also set (`pods`, `services`, `configmaps`, `secrets`) to prevent namespace sprawl.

## Node pools

- Keep at least one general-purpose node pool for stateless workloads.
- Add separate pools when traffic or compliance requirements diverge (for example, production-only workloads).
- Use taints/tolerations only when needed to avoid scheduling complexity on day 1.

## Network policy baseline

Baseline policy is included at `apps/base/networkpolicy.yaml`:

- Denies implicit broad ingress by selecting only app pods.
- Allows ingress traffic only from `ingress-nginx` namespace to app port `8080`.
- Keeps east-west traffic controlled by default.

Recommended next additions:

- Add explicit egress policies for DNS, database, and external APIs.
- Add namespace-level default deny policies once dependencies are fully mapped.

## Rollout and rollback

- Argo CD drives sync from Git overlays.
- Validate health with readiness/liveness probes and Argo application health.
- Roll back by reverting the image tag commit for the target overlay and re-syncing.

## Monitoring and observability baseline

Cluster-level monitoring is deployed in a dedicated `monitoring` namespace via Argo CD + Helm:

- `platform/argocd/app-monitoring-base.yaml` deploys namespace guardrails and shared secrets.
- `platform/argocd/app-monitoring-stack.yaml` deploys `kube-prometheus-stack` from the Prometheus community Helm repo.
- `platform/argocd/app-monitoring-integrations.yaml` deploys external `ServiceMonitor` integrations after the stack is available.

Configured defaults:

- Stack components: Prometheus, Grafana, Alertmanager.
- Grafana ingress host: `grafana.example.com` with `cert-manager` cluster issuer `letsencrypt-production`.
- Grafana dashboards: open-source Kubernetes dashboards bundled by `kube-prometheus-stack`.
- Alert channel: Slack `#grafana`, webhook read from secret `alertmanager-slack` key `webhook-url`.
- Retention: metrics `15d` (Prometheus), alert data `120h` (Alertmanager).
- Storage class default: `csi-disk` (replace if your cluster uses a different class).

Log and trace retention targets are defined in `docs/observability-retention-policy.md` and should be enforced when Loki/Tempo are added.

### ServiceMonitor integration pattern

- Place integration monitors in `platform/monitoring-integrations/`.
- Keep them in namespace `monitoring`.
- Use a later Argo CD sync wave than the monitoring stack so CRDs already exist.
- Example included: `platform/monitoring-integrations/cert-manager-servicemonitor.yaml`.

## Ingress and TLS baseline

Platform ingress and certificate management are deployed via Argo CD:

- `platform/argocd/app-ingress-nginx.yaml` deploys `ingress-nginx` in namespace `ingress-nginx`.
- `platform/argocd/app-cert-manager.yaml` deploys `cert-manager` in namespace `cert-manager`.
- `platform/argocd/app-cert-manager-issuers.yaml` deploys shared issuers from Git.
- `platform/cert-manager-config/clusterissuer-letsencrypt-production.yaml` defines the default `ClusterIssuer` used by ingress objects.

Default assumptions:

- Ingress service type is `LoadBalancer` (CCE ELB); add your cluster-specific ELB annotations in `app-ingress-nginx.yaml`.
- ACME HTTP-01 solver uses ingress class `nginx`.
- Base application ingress uses annotation `cert-manager.io/cluster-issuer: letsencrypt-production`.
