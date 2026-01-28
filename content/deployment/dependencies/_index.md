---
title: Pre-Deployment Dependencies
weight: 3
---

> [!CAUTION]
> **Critical Setup Step**: AegisLab will **NOT** function without these dependencies. The system requires Chaos Mesh for fault injection and cert-manager for certificate management. Install all required dependencies before proceeding.

Before deploying AegisLab, install the required dependencies based on your environment:

## Required Dependencies

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

## Quick Start Script

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

## Install Target Applications

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
