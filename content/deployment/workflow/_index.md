---
title: Deployment Workflow
weight: 4
---

## Deployment Overview

| Mode           | RCABench Platform   | Injection Target    | Deployment Method     | Command                           |
| -------------- | ------------------- | ------------------- | --------------------- | --------------------------------- |
| **Debug**      | Docker (local)      | Kind/Minikube/K8s   | Manual                | `make local-deploy + local-debug` |
| **Test**       | Kubernetes (Kind)   | Kind (same cluster) | Skaffold + auto hooks | `make test`                       |
| **Staging**    | Kubernetes (remote) | K8s (same cluster)  | Skaffold              | `make run`                        |
| **Production** | Kubernetes (remote) | K8s (same cluster)  | Helm                  | `make install-rcabench`           |

## Environment Setup

### Environment Types

| Environment | Config                            | Registry      | Storage              | Cluster                 |
| ----------- | --------------------------------- | ------------- | -------------------- | ----------------------- |
| Debug       | `config.dev.toml`                 | Local         | Local filesystem     | Docker + K8s (separate) |
| Test        | `manifests/test/rcabench.yaml`    | cr.volces.com | Disabled (ephemeral) | Kind (local)            |
| Staging     | `manifests/staging/rcabench.yaml` | Harbor        | JuiceFS              | Remote K8s              |
| Production  | `manifests/prod/rcabench.yaml`    | cr.volces.com | Volcengine NAS       | Remote K8s              |

### Configuration Management

Configuration is loaded in the following precedence (highest to lowest):

1. Environment variables
2. `$HOME/.rcabench/config.{ENV}.toml`
3. `/etc/rcabench/config.{ENV}.toml`
4. `./config.{ENV}.toml`

### Initial Data Seeding

The system loads initial data from YAML files on startup:

```plain
data/initial_data/
├── staging/
│   ├── data.json || data.yaml       # Containers, datasets, projects, users
│   ├── otel-demo.yaml  # OpenTelemetry Demo config
│   └── ts.yaml         # Train Ticket config
└── prod/
    ├── data.json || data.yaml
    ├── otel-demo.yaml
    └── ts.yaml
```

**Data Files by Environment**:

- **Debug**: Uses `data/initial_data/staging/*.yaml` (same as staging)
- **Test**: Uses `data/initial_data/prod/*.yaml` (same as production)
- **Staging**: Uses `data/initial_data/staging/*.yaml`
- **Production**: Uses `data/initial_data/prod/*.yaml`

Default admin credentials:

> [!CAUTION]
> **Security Risk**: These are default credentials visible in public documentation. **Change immediately after first login** to prevent unauthorized access.

- **Username**: `admin`
- **Password**: `admin123`

## Debug Mode (Local Development)

> [!NOTE]
> **Use Case**: Debug mode runs RCABench in Docker containers with your local Kubernetes cluster as the target. Perfect for algorithm development with fast iteration cycles and debugging capabilities.

For local development with hot-reload and debugging capabilities:

```bash
# Start RCABench in Docker
make local-deploy
make local-debug
```

## Test Deployment (Kind + Skaffold)

For CI/CD testing on local Kind cluster:

```bash
# Using Make test
# This does:
# 1. Create Kind cluster (if needed)
# 2. Deploy via Skaffold with ENV_MODE=test
# 3. Trigger regression-test via Skaffold hooks
# 4. Run regression tests
make test
```

## Staging Deployment (Remote K8s + Skaffold)

For team testing on remote Kubernetes cluster:

```bash
# Set environment and deploy
# Equivalent to: ENV_MODE=staging devbox run skaffold run
ENV_MODE=staging make run
```

## Production Deployment (Kubernetes + Helm)

Production deploys to Kubernetes using Helm directly with Volcengine registry:

```bash
# Using Make
make install-rcabench
```

## Observability

### Logging

> [!NOTE]
> **Log Persistence**: Failed job logs are automatically saved to persistent storage. In production, these logs are retained for 7 days for compliance and debugging purposes.

Job failed run logs are stored in `/app/logs/jobs/` with the following structure:

```plain
/app/logs/jobs/
└── {namespace}/
    └── {date}/
        └── {timestamp}-{jobName}.json
```

**View logs**:

```bash
# Producer logs
kubectl logs -f deployment/rcabench-producer -n exp

# Consumer logs
kubectl logs -f deployment/rcabench-consumer -n exp

# MySQL logs
kubectl logs -f statefulset/rcabench-mysql -n exp

# All pods logs
kubectl logs -f -l app=rcabench -n exp
```

### Distributed Tracing

AegisLab uses Jaeger for distributed tracing.

**Access Jaeger UI**:

```bash
kubectl port-forward statefulset/rcabench-jaeger 16686:16686 -n exp
# Open browser to http://localhost:16686
```

**Trace Propagation**:

Traces use hierarchical span contexts:

- **Group context**: Top-level trace (grandfather span)
- **Trace context**: Task type-specific span (father span)
- **Task context**: Individual task execution span

Trace context is propagated via Kubernetes annotations:

- `taskCarrier`: OpenTelemetry trace context
- `traceCarrier`: Additional trace metadata
- Labels: `taskID`, `traceID`, `taskType`, `groupID`, `projectID`, `userID`
