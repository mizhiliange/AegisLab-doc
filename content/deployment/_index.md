---
title: Deployment Guide
weight: 4
---

# Deployment Guide

Complete guide for deploying the AegisLab ecosystem components.

## Overview

Deploying the AegisLab ecosystem requires setting up multiple interconnected components. This guide walks you through the complete deployment process.

## Prerequisites

Before starting deployment, ensure you have:

### Infrastructure Requirements

- **Kubernetes Cluster**: Version 1.20 or higher
  - Minimum 3 worker nodes
  - 8 CPU cores and 16GB RAM per node
  - Storage class for persistent volumes

- **Container Registry**: Docker Hub, Harbor, or similar
  - For storing custom images
  - Push/pull access configured

- **Network Access**:
  - Cluster accessible from your workstation
  - Internet access for pulling images
  - NodePort or LoadBalancer support for external access

### Software Requirements

- `kubectl` (1.20+)
- `helm` (3.0+)
- `docker` or `podman`
- `git`
- Python 3.10+ (for SDK and tools)
- Go 1.21+ (for building AegisLab)

### Access Requirements

- Kubernetes cluster admin access
- Container registry credentials
- Git repository access

## Deployment Order

Deploy components in this order to satisfy dependencies:

1. **Prerequisites**: Chaos Mesh, Redis, MySQL, monitoring stack
2. **TrainTicket**: Target microservices application
3. **AegisLab**: Core orchestration platform
4. **LoadGenerator**: Traffic generation
5. **Pandora**: Intelligent fault scheduler (optional)

## Quick Start

For a minimal deployment to get started:

```bash
# 1. Install Chaos Mesh
kubectl create ns chaos-mesh
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh

# 2. Deploy TrainTicket
cd train-ticket
helm dependency build manifests/helm/generic_service
helm install ts manifests/helm/generic_service -n ts --create-namespace

# 3. Deploy AegisLab
cd AegisLab
docker compose up -d redis mysql
make deploy-k8s

# 4. Start LoadGenerator
cd loadgenerator
go build -o loadgenerator
./loadgenerator --threads 5 --sleep 1000
```

## Deployment Guides

{{< cards >}}
  {{< card link="prerequisites" title="Prerequisites" icon="document" >}}
  {{< card link="kubernetes-cluster" title="Kubernetes Cluster" icon="document" >}}
  {{< card link="aegislab-deployment" title="AegisLab Deployment" icon="document" >}}
  {{< card link="train-ticket-deployment" title="TrainTicket Deployment" icon="document" >}}
  {{< card link="chaos-mesh-deployment" title="Chaos Mesh Deployment" icon="beaker" >}}
  {{< card link="loadgenerator-deployment" title="LoadGenerator Deployment" icon="arrow-circle-right" >}}
{{< /cards >}}

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  AegisLab    │  │ TrainTicket  │  │  Chaos Mesh  │     │
│  │  Namespace   │  │  Namespace   │  │  Namespace   │     │
│  │              │  │              │  │              │     │
│  │ - Producer   │  │ - 40+ svcs   │  │ - Controller │     │
│  │ - Consumer   │  │ - Databases  │  │ - Dashboard  │     │
│  │ - Redis      │  │ - Gateway    │  │              │     │
│  │ - MySQL      │  │              │  │              │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Monitoring Stack (Optional)                 │  │
│  │  - Prometheus  - Grafana  - Jaeger  - ClickHouse    │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ External Access
                              ▼
                    ┌──────────────────┐
                    │  LoadGenerator   │
                    │  (External)      │
                    └──────────────────┘
```

## Environment Configurations

The ecosystem supports multiple environment configurations:

### Development Environment

- Single-node Kubernetes (minikube, kind, k3s)
- Minimal resource allocation
- Local container registry
- Debug logging enabled

### Test Environment

- Multi-node Kubernetes cluster
- Moderate resource allocation
- Shared container registry
- Standard logging

### Production Environment

- Production Kubernetes cluster
- Full resource allocation
- High availability configuration
- Production logging and monitoring

## Configuration Management

Each component uses environment-specific configuration:

- **AegisLab**: `src/config.{dev,test,prod}.toml`
- **TrainTicket**: Helm values files
- **Pandora**: `config/config_{dev,prod}.toml`
- **RCABench Platform**: Environment variables

Set the environment mode:

```bash
export ENV_MODE=dev  # or test, prod
```

## Network Configuration

### Service Exposure

- **AegisLab API**: NodePort 30080 or LoadBalancer
- **TrainTicket Gateway**: NodePort 30080
- **Chaos Mesh Dashboard**: NodePort 31111
- **Monitoring Dashboards**: NodePort or Ingress

### Internal Communication

- Services communicate via Kubernetes DNS
- Example: `ts-order-service.ts.svc.cluster.local`

## Storage Configuration

### Persistent Volumes

Required for:
- MySQL databases (AegisLab, TrainTicket services)
- Redis persistence
- Dataset storage
- Log aggregation

### JuiceFS (Optional)

For large-scale dataset storage:

```bash
sudo juicefs mount redis://10.10.10.119:6379/1 /mnt/jfs -d
```

## Monitoring and Observability

### OpenTelemetry Stack

- **Collector**: Receives traces, metrics, logs
- **Backend**: ClickHouse for traces, Prometheus for metrics
- **Visualization**: Jaeger UI, Grafana dashboards

### Health Checks

Monitor component health:

```bash
# AegisLab health
curl http://aegislab-api:8080/health

# TrainTicket services
kubectl get pods -n ts

# Chaos Mesh status
kubectl get pods -n chaos-mesh
```

## Troubleshooting

### Common Issues

**Pods not starting**: Check resource limits and node capacity

```bash
kubectl describe pod <pod-name> -n <namespace>
```

**Image pull errors**: Verify registry credentials

```bash
kubectl get events -n <namespace>
```

**Service communication failures**: Check network policies and DNS

```bash
kubectl exec -it <pod-name> -n <namespace> -- nslookup <service-name>
```

**Chaos injection not working**: Verify Chaos Mesh installation

```bash
kubectl get chaosengine -A
```

## Next Steps

1. Follow the [Prerequisites](prerequisites) guide to prepare your environment
2. Set up your [Kubernetes Cluster](kubernetes-cluster)
3. Deploy [AegisLab](aegislab-deployment) as the core platform
4. Deploy [TrainTicket](train-ticket-deployment) as the target system
5. Install [Chaos Mesh](chaos-mesh-deployment) for fault injection
6. Start [LoadGenerator](loadgenerator-deployment) for traffic generation

## Getting Help

- Check component-specific logs for errors
- Review Kubernetes events for deployment issues
- Consult individual component documentation
- Report issues on GitHub repositories
