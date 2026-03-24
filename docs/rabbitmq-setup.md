# RabbitMQ Setup Guide

This document describes how RabbitMQ is deployed in the BSIS Kubernetes cluster and how to configure services to use it.

## Overview

RabbitMQ is deployed **in-cluster** as a shared messaging service, with one instance per environment:

- **app-development** → RabbitMQ for development
- **app-staging** → RabbitMQ for staging  
- **app-production** → RabbitMQ for production

Each instance uses the [Bitnami RabbitMQ Helm chart](https://github.com/bitnami/charts/tree/main/bitnami/rabbitmq) and runs in the same namespace as the application services.

## Architecture

- **Deployment**: `platform/argocd/applicationset-rabbitmq.yaml` creates Argo CD Applications (`rabbitmq-development`, `rabbitmq-staging`, `rabbitmq-production`)
- **Credentials**: External Secrets Operator syncs `RABBITMQ_PASSWORD` from Vault (`bsis/<env>/shared`)
- **Connection**: Services in the same namespace use host `rabbitmq` and port `5672`

## Prerequisites

### 1. Vault secrets

Ensure these keys exist in `bsis/<env>/shared` for each environment:

| Key | Description |
|-----|-------------|
| `RABBITMQ_USERNAME` | Set to `user` (must match Helm `auth.username`) |
| `RABBITMQ_PASSWORD` | Secure password for the default user |

`RABBITMQ_HOST` and `RABBITMQ_PORT` are configured in ConfigMaps (rabbitmq / 5672 for in-cluster).

Example (using Vault CLI):

```bash
vault kv put kv/bsis/development/shared \
  RABBITMQ_USERNAME=user \
  RABBITMQ_PASSWORD="<your-secure-password>"
```

### 2. Vault Kubernetes auth

Create Vault roles for RabbitMQ's ServiceAccount:

- `rabbitmq-development` (bound to SA `rabbitmq` in `app-development`)
- `rabbitmq-staging` (bound to SA `rabbitmq` in `app-staging`)
- `rabbitmq-production` (bound to SA `rabbitmq` in `app-production`)

Each role must have policy access to read `bsis/<env>/shared`.

## Services using RabbitMQ

These services are already configured to use RabbitMQ via their ExternalSecrets:

| Service | Usage |
|---------|-------|
| core-api | Spring AMQP |
| identity-service | Spring AMQP |
| devices-integrations-api | Custom `RabbitMQPublisher` (exchange: `analyzer.results.exchange`) |

## Adding RabbitMQ to a new service

1. **ConfigMap**: Add static connection details (same for all envs):
   - `SPRING_RABBITMQ_HOST: "rabbitmq"`
   - `SPRING_RABBITMQ_PORT: "5672"`

2. **ExternalSecret**: Add `RABBITMQ_USERNAME` and `RABBITMQ_PASSWORD` from `bsis/<env>/shared`.

3. **Deployment**: Reference the secret for credentials; host/port come from ConfigMap via envFrom (see `core-api` or `identity-service` base deployment for the pattern).

## Management UI

The RabbitMQ management plugin is enabled by default. To access it:

```bash
kubectl port-forward -n app-development svc/rabbitmq 15672:15672
```

Then open http://localhost:15672 (default user: `user`, password: from Vault).

## Metrics

RabbitMQ metrics are exposed and a ServiceMonitor is created for Prometheus (when ServiceMonitor CRDs exist).
