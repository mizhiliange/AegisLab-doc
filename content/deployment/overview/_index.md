---
title: Overview
weight: 1
---

## Deployment Modes

> [!TIP]
> **Quick Mode Selection**:
>
> - 👨‍💻 Developing platform? → Use **Debug Mode**
> - 🧪 Running CI/CD tests? → Use **Test Mode**
> - 👥 Team collaboration? → Use **Staging Mode**
> - 🚀 Production deployment? → Use **Production Mode**

AegisLab supports four deployment modes:

{{< img "deployment_modes.svg" "AegisLab 部署模式图" >}}

## Components

### AegisLab Components

| Component | Type        | Purpose                                        |
| --------- | ----------- | ---------------------------------------------- |
| Producer  | Deployment  | HTTP API server (port 8080)                    |
| Consumer  | Deployment  | Background task workers, K8s controllers       |
| MySQL     | StatefulSet | Primary database (port 3306)                   |
| Redis     | StatefulSet | Cache and task queue (port 6379)               |
| etcd      | StatefulSet | Distributed configuration (port 2379)          |
| Jaeger    | StatefulSet | Distributed tracing (port 16686 UI, 4318 OTLP) |
| BuildKit  | Deployment  | Container build service (port 1234)            |

### Platform Dependencies (Must Install First)

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

## Storage Options

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
