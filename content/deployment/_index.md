---
title: Deployment Guide
weight: 4
---

Complete guide for deploying the AegisLab ecosystem components.

## Overview

> [!TIP]
> **Quick Mode Selection**:
>
> - 👨‍💻 Developing platform? → Use **Debug Mode**
> - 🧪 Running CI/CD tests? → Use **Test Mode**
> - 👥 Team collaboration? → Use **Staging Mode**
> - 🚀 Production deployment? → Use **Production Mode**

AegisLab supports four deployment modes:

```plain
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AegisLab Deployment Modes                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │   Debug Mode    │  │  Test/Staging   │  │       Production            │  │
│  │  (Local Dev)    │  │     Mode        │  │         Mode                │  │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────────────────┤  │
│  │ RCABench: Docker│  │RCABench: K8s    │  │  RCABench: K8s              │  │
│  │ Target: Kind/K8s│  │Target: Same K8s │  │  Target: Same K8s           │  │
│  │                 │  │                 │  │                             │  │
│  │1. make local-   │  │1. ENV_MODE=X    │  │  1. make install-rcabench   │  │
│  │   deploy        │  │   make run      │  │                             │  │
│  │2. make local-   │  │   (skaffold)    │  │                             │  │
│  │   debug         │  │                 │  │                             │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Components

#### AegisLab Components

| Component | Type        | Purpose                                        |
| --------- | ----------- | ---------------------------------------------- |
| Producer  | Deployment  | HTTP API server (port 8080)                    |
| Consumer  | Deployment  | Background task workers, K8s controllers       |
| MySQL     | StatefulSet | Primary database (port 3306)                   |
| Redis     | StatefulSet | Cache and task queue (port 6379)               |
| etcd      | StatefulSet | Distributed configuration (port 2379)          |
| Jaeger    | StatefulSet | Distributed tracing (port 16686 UI, 4318 OTLP) |
| BuildKit  | Deployment  | Container build service (port 1234)            |

#### Platform Dependencies (Must Install First)

| Component                | Type            | Purpose                                          | Test     | Staging | Production |
| ------------------------ | --------------- | ------------------------------------------------ | -------- | ------- | ---------- |
| Chaos Mesh               | Helm Chart      | Fault injection platform                         | ✅       | ✅      | ✅         |
| cert-manager             | Helm Chart      | Certificate management                           | ✅       | ✅      | ✅         |
| OpenTelemetry Kube Stack | Helm Chart      | Distributed tracing and metrics                  | ✅       | ✅      | ✅         |
| OpenTelemetry Demo       | Helm Chart      | Target application for testing (detector issues) | ✅       | ✅      | ✅         |
| Train Ticket             | Helm Chart      | Target application (most mature)                 | ✅       | ✅      | ✅         |
| Other Apps               | -               | 6 more apps in adaptation                        | 🚧       | 🚧      | 🚧         |
| JuiceFS CSI Driver       | Helm Chart      | Distributed file system                          | ✅       | ✅      | ❌         |
| ClickHouse               | External / Helm | Log and metrics storage                          | External | ✅      | ✅         |

### Storage Options

Storage configuration varies by environment:

| Environment | Storage Type        | StorageClass | PVCs Enabled       |
| ----------- | ------------------- | ------------ | ------------------ |
| Local/Debug | Local filesystem    | N/A          | N/A                |
| Test        | JuiceFS (installed) | N/A          | **No** (ephemeral) |
| Staging     | JuiceFS             | `juicefs-sc` | **Yes**            |
| Production  | Volcengine NAS      | `rcabench`   | **Yes**            |

> [!WARNING]
> **Storage Considerations**:
>
> - Test mode uses ephemeral storage - all data is lost when pods restart
> - Staging requires JuiceFS for shared storage across pods
> - Production uses Volcengine NAS - ensure NAS is provisioned before deployment

## Prerequisites

> [!NOTE]
> **Version Compatibility**: This guide is tested with Kubernetes v1.28+ and Helm 3.13+. Older versions may work but are not officially supported.

Before deploying AegisLab, ensure the following prerequisites are met:

### Software Requirements

| Tool     | Version                      | Installation                                                                                     |
| -------- | ---------------------------- | ------------------------------------------------------------------------------------------------ |
| devbox   | Latest                       | `curl -fsSL https://nix-community.org/detect-devbox.sh`                                          |
| Docker   | 20.10+                       | `apt-get install docker.io`                                                                      |
| Helm     | 3.x                          | `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3`                       |
| kubectl  | Match K8s version            | Follow Kubernetes docs                                                                           |
| kubectx  | Latest                       | `git clone https://github.com/ahmetb/kubectx $HOME/kubectx`                                      |
| Skaffold | Latest                       | `curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64` |
| Kind     | Latest (for local testing)   | `go install sigs.k8s.io/kind@latest`                                                             |
| Minikube | Latest (alternative to Kind) | Follow Minikube docs                                                                             |

### Kubernetes Cluster Requirements

- **Version**: v1.28 or later
- **Resources**:
  - Development: 8 CPU, 16GB RAM minimum
  - Production: 16 CPU, 32GB RAM minimum
- **Capabilities**: StatefulSets, PVCs, CRDs (Chaos Mesh)
- **Network**: Pod-to-Pod communication, Service exposure

---

## Pre-Deployment Dependencies

> [!CAUTION]
> **Critical Setup Step**: AegisLab will **NOT** function without these dependencies. The system requires Chaos Mesh for fault injection and cert-manager for certificate management. Install all required dependencies before proceeding.

Before deploying AegisLab, install the required dependencies based on your environment:

### Required Dependencies

| Dependency                         | Purpose                         | Test    | Staging | Production |
| ---------------------------------- | ------------------------------- | ------- | ------- | ---------- |
| Chaos Mesh                         | Fault injection platform        | **Yes** | **Yes** | **Yes**    |
| cert-manager                       | Certificate management          | **Yes** | **Yes** | **Yes**    |
| OpenTelemetry Kube Stack(Optional) | Distributed tracing and metrics | **Yes** | **Yes** | **Yes**    |
| OpenTelemetry Demo(Optional)       | Target application for testing  | **Yes** | **Yes** | **Yes**    |
| JuiceFS CSI Driver(Optional)       | Distributed file system         | **Yes** | **Yes** | **No**     |
| ClickHouse(Optional)               | Log and metrics storage         | **Yes** | **No**  | **Yes**    |

**Storage Difference**:

- **Test**: JuiceFS installed but PVCs disabled (ephemeral storage for CI/CD)
- **Staging**: JuiceFS with full PVC support
- **Production**: Volcengine NAS (no JuiceFS needed)

**Why These Dependencies Are Required**:

- **Chaos Mesh**: Required for fault injection. Without it, AegisLab cannot inject failures into target applications.
- **cert-manager**: Required for TLS certificates in Chaos Mesh and other services.
- **OpenTelemetry Kube Stack**: Provides observability. Without it, you cannot trace fault injection results.
- **OpenTelemetry Demo**: Serves as the target application for fault injection experiments.
- **JuiceFS**: Provides distributed storage. Test/Staging use JuiceFS (PVCs disabled in Test for CI/CD speed), Production uses Volcengine NAS.
- **ClickHouse**: Stores logs and metrics for analysis.

### Quick Start Script

> [!TIP]
> The `start.sh` script automatically detects your environment and installs all required dependencies in the correct order. It handles Chaos Mesh, cert-manager, and optional components like JuiceFS.

Use the provided script to install all dependencies:

```bash
# For debug environment (local Docker + Kubernetes)
make setup-dev-env

# For test environment (with Kind cluster + JuiceFS)
./scripts/start.sh test

# For production environment (without JuiceFS, uses Volcengine NAS)
./scripts/start.sh prod
```

### Install Target Applications

> [!IMPORTANT]
>
> AegisLab requires target applications for fault injection experiments.

**Available Target Applications**:

| Application        | Status           | Notes                                |
| ------------------ | ---------------- | ------------------------------------ |
| Train Ticket       | ✅ Mature        | Most stable, recommended for testing |
| OpenTelemetry Demo | ⚠️ Partial       | `Detector` image is not compatible   |
| Other 6 Apps       | 🚧 In Adaptation | Currently being adapted              |

**Install Train Ticket** (Recommended):

```bash
helm repo add train-ticket https://lgu-se-internal.github.io/train-ticket --force-update
helm install train-ticket0 train-ticket/train-ticket \
  --namespace train-ticket0 \
  --create-namespace \
  --atomic \
  --timeout 10m
```

**Install OpenTelemetry Demo** (Optional - for tracing only):

```bash
helm repo add opentelemetry-demo https://lgu-se-internal.github.io/opentelemetry-demo --force-update
helm install otel-demo0 opentelemetry-demo/opentelemetry-demo \
  --namespace otel-demo0 \
  --create-namespace \
  -f data/initial_data/prod/otel-demo.yaml \
  --atomic \
  --timeout 10m
```

---

## Deployment Workflow

| Mode           | RCABench Platform   | Injection Target    | Deployment Method     | Command                           |
| -------------- | ------------------- | ------------------- | --------------------- | --------------------------------- |
| **Debug**      | Docker (local)      | Kind/Minikube/K8s   | Manual                | `make local-deploy + local-debug` |
| **Test**       | Kubernetes (Kind)   | Kind (same cluster) | Skaffold + auto hooks | `make test`                       |
| **Staging**    | Kubernetes (remote) | K8s (same cluster)  | Skaffold              | `make run`                        |
| **Production** | Kubernetes (remote) | K8s (same cluster)  | Helm                  | `make install-rcabench`           |

### Environment Setup

#### Environment Types

| Environment | Config                            | Registry      | Storage              | Cluster                 |
| ----------- | --------------------------------- | ------------- | -------------------- | ----------------------- |
| Debug       | `config.dev.toml`                 | Local         | Local filesystem     | Docker + K8s (separate) |
| Test        | `manifests/test/rcabench.yaml`    | cr.volces.com | Disabled (ephemeral) | Kind (local)            |
| Staging     | `manifests/staging/rcabench.yaml` | Harbor        | JuiceFS              | Remote K8s              |
| Production  | `manifests/prod/rcabench.yaml`    | cr.volces.com | Volcengine NAS       | Remote K8s              |

#### Configuration Management

Configuration is loaded in the following precedence (highest to lowest):

1. Environment variables
2. `$HOME/.rcabench/config.{ENV}.toml`
3. `/etc/rcabench/config.{ENV}.toml`
4. `./config.{ENV}.toml`

#### Initial Data Seeding

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

### Debug Mode (Local Development)

> [!NOTE]
> **Use Case**: Debug mode runs RCABench in Docker containers with your local Kubernetes cluster as the target. Perfect for algorithm development with fast iteration cycles and debugging capabilities.

For local development with hot-reload and debugging capabilities:

```bash
# Start RCABench in Docker
make local-deploy
make local-debug
```

### Test Deployment (Kind + Skaffold)

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

### Staging Deployment (Remote K8s + Skaffold)

For team testing on remote Kubernetes cluster:

```bash
# Set environment and deploy
# Equivalent to: ENV_MODE=staging devbox run skaffold run
ENV_MODE=staging make run
```

### Production Deployment (Kubernetes + Helm)

Production deploys to Kubernetes using Helm directly with Volcengine registry:

```bash
# Using Make
make install-rcabench
```

---

## Verification

> [!TIP]
> **Quick Verification**: Run the health check endpoint first. If it returns `"status": "healthy"`, your deployment is successful. Then proceed with detailed smoke tests.

### Health Check Endpoint

The system provides a comprehensive health check at `/system/health`:

```bash
# Get the API service endpoint
kubectl get svc -n exp rcabench-producer

# Check health (replace <api-service> with actual service IP/hostname)
curl http://<api-service>:8080/system/health
```

Response format:

```json
{
  "status": "healthy",
  "timestamp": "2025-01-28T10:00:00Z",
  "version": "1.1.55",
  "uptime": "5ms",
  "services": {
    "database": {
      "status": "healthy",
      "response_time": "5ms"
    },
    "redis": {
      "status": "healthy",
      "response_time": "2ms"
    },
    "jaeger": {
      "status": "healthy",
      "response_time": "10ms"
    },
    "kubernetes": {
      "status": "healthy",
      "response_time": "15ms"
    },
    "buildkit": {
      "status": "healthy",
      "response_time": "3ms"
    }
  }
}
```

### Smoke Tests

Run these commands after deployment:

```bash
# 1. Check platform dependencies are running
kubectl get pods -n chaos-mesh
kubectl get pods -n cert-manager
kubectl get pods -n monitoring
kubectl get pods -n train-ticket0  # Recommended target

# 2. Check AegisLab pods are running
kubectl get pods -n exp

# Expected output:
# NAME                        READY   STATUS    RESTARTS   AGE
# rcabench-producer-xxx       1/1     Running   0          2m
# rcabench-consumer-xxx       1/1     Running   0          2m
# rcabench-mysql-0            1/1     Running   0          3m
# rcabench-redis-0            1/1     Running   0          3m
# rcabench-etcd-0             1/1     Running   0          3m
# rcabench-jaeger-0           1/1     Running   0          3m

# 3. Check service health
curl http://<api-service>/system/health | jq '.status'
# Expected: "healthy"

# 4. Verify database connectivity
kubectl exec -it statefulset/rcabench-mysql -n exp -- \
  mysqladmin ping -h localhost
# Expected: mysqld is alive

# 5. Verify Redis connectivity
kubectl exec -it statefulset/rcabench-redis -n exp -- \
  redis-cli ping
# Expected: PONG

# 6. Test API authentication
curl -X POST http://<api-service>:8080/api/v2/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' | jq '.data.access_token'

# 7. Verify Chaos Mesh is ready
kubectl get workflows -n chaos-mesh
```

---

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

---

## Troubleshooting

### Common Failure Modes

| Failure Mode               | Symptoms                                   | Diagnosis                      | Resolution                                             |
| -------------------------- | ------------------------------------------ | ------------------------------ | ------------------------------------------------------ |
| Chaos Mesh Not Ready       | Injection failures, "workflow not found"   | Check Chaos Mesh pods          | `kubectl get pods -n chaos-mesh` and restart if needed |
| Database Connection Failed | 503 errors, "connection refused"           | Check MySQL StatefulSet, PVC   | `kubectl delete pod rcabench-mysql-0 -n exp`           |
| Redis Unavailable          | Task queue stalled                         | Health check returns unhealthy | `kubectl delete pod rcabench-redis-0 -n exp`           |
| BuildKit Failure           | Container build errors                     | Check BuildKit logs            | Restart BuildKit, verify Harbor certs                  |
| etcd Timeout               | Config not loading                         | Check etcd cluster health      | Reinitialize etcd StatefulSet                          |
| JuiceFS Mount Failed       | Dataset not accessible (Test/Staging only) | Check PV/PVC binding           | Verify JuiceFS secret, remount                         |
| OTEL Demo Not Found        | Cannot create injection targets            | Check OTEL Demo namespace      | `kubectl get pods -n otel-demo0`                       |
| High Memory Usage          | OOMKilled pods                             | Check container limits         | Increase resource limits                               |
| Stuck Tasks                | Tasks in Running state forever             | Check dead letter queue        | Manually cancel stuck tasks                            |

---

## Support

For issues and questions:

1. Check existing troubleshooting section
2. Review logs in `/app/logs/jobs/`
3. Open a GitHub issue with:
   - Environment (debug/test/staging/prod)
   - Error messages and logs
   - Steps to reproduce
   - Kubernetes version and cluster info
