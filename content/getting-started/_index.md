---
title: Getting Started
weight: 1
---

# Getting Started with AegisLab

Welcome to the AegisLab ecosystem! This guide helps you understand the system architecture and choose the right path for your needs.

## What is AegisLab?

AegisLab is a comprehensive fault injection and root cause analysis (RCA) evaluation platform for microservices research. It enables:

- **Automated chaos engineering experiments** in distributed systems
- **Intelligent fault scheduling** using genetic algorithms
- **Standardized RCA algorithm evaluation** with reproducible datasets
- **End-to-end observability** with traces, metrics, and logs

## Choose Your Role

AegisLab serves two primary user personas:

### Algorithm Developer

You want to develop and evaluate RCA algorithms using standardized datasets.

**You should choose this path if you:**
- Have ML or rule-based approaches for root cause analysis
- Want to benchmark your algorithm against existing solutions
- Need access to realistic fault injection datasets
- Want to contribute to the RCA research community

[Go to Algorithm Developer Guide →](../algorithm-developers)

### Dataset Creator

You want to create datasets through fault injection experiments in microservices systems.

**You should choose this path if you:**
- Need to generate training/evaluation data for RCA algorithms
- Want to study failure patterns in distributed systems
- Need to test system resilience under various fault conditions
- Want to use genetic algorithms for intelligent fault scheduling

[Go to Dataset Creator Guide →](../dataset-creators)

## System Architecture

The AegisLab ecosystem consists of six interconnected components:

```
┌─────────────────────────────────────────────────────────────┐
│                      AegisLab (RCABench)                     │
│  Central orchestration platform for fault injection & RCA   │
│         Producer (API) + Consumer (Background Workers)       │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ orchestrates
                              ▼
┌─────────────────┐    ┌──────────────────┐    ┌──────────────┐
│  Chaos Experiment│───▶│   TrainTicket    │◀───│LoadGenerator │
│  Fault injection │    │  40+ services    │    │ Traffic gen  │
│  wrapper         │    │  Target system   │    │              │
└─────────────────┘    └──────────────────┘    └──────────────┘
                              │
                              │ observability
                              ▼
                    ┌──────────────────┐
                    │  OTEL Collector  │
                    │  Traces/Metrics  │
                    └──────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
│   Pandora    │    │ RCABench Platform│    │   Datasets   │
│  Genetic     │    │  Algorithm eval  │    │   Parquet    │
│  scheduler   │    │  framework       │    │   files      │
└──────────────┘    └──────────────────┘    └──────────────┘
```

### Component Overview

**AegisLab (RCABench)**
- REST API for experiment management
- Redis-based task queue for async processing
- Kubernetes job scheduler for algorithm execution
- Runs in Producer (API server) and Consumer (worker) modes

**TrainTicket**
- Target microservices application (40+ Spring Boot services)
- Simulates train ticket booking system
- Instrumented with OpenTelemetry for distributed tracing
- Each service has independent database and REST API

**Chaos Experiment**
- Programmatic wrapper for Chaos Mesh CRDs
- Three-layer architecture: specs → orchestration → intelligent generation
- System-aware fault generation using service metadata
- Supports network, pod, stress, JVM, and HTTP faults

**LoadGenerator**
- Realistic traffic generation using chain-of-responsibility pattern
- Probabilistic behavior selection (booking, payment, admin operations)
- OpenTelemetry tracing integration
- Multi-level randomness for realistic user simulation

**Pandora**
- Genetic algorithm-based intelligent fault scheduler
- Evolves fault scenarios to maximize SLO violations
- Multi-objective fitness: latency + error rate + diagnostic difficulty
- Integrates with AegisLab API for automated experimentation

**RCABench Platform**
- Algorithm development framework with standardized interface
- Plugin-based registry for RCA algorithms and trace samplers
- Polars-based lazy evaluation for large-scale data processing
- Standardized parquet format with converters for multiple datasets

## Data Flow

Understanding how data flows through the system:

1. **Deployment**: TrainTicket deployed to Kubernetes, LoadGenerator generates traffic
2. **Fault Injection**: Pandora/Manual → AegisLab API → Chaos Experiment → Chaos Mesh → TrainTicket
3. **Observability**: TrainTicket (OTEL) → Traces/Metrics/Logs → Storage (ClickHouse/Prometheus)
4. **RCA Evaluation**: AegisLab → collects traces → RCABench Platform → algorithms → metrics
5. **Intelligent Scheduling**: Pandora → evaluates results → genetic algorithm → new scenarios

## Prerequisites

Before getting started, ensure you have:

- **For Algorithm Developers**:
  - Python 3.10+
  - Basic understanding of distributed tracing
  - Familiarity with microservices architecture

- **For Dataset Creators**:
  - Python 3.10+
  - Access to Kubernetes cluster
  - Basic understanding of chaos engineering
  - Familiarity with microservices and distributed systems

- **For Infrastructure Setup**:
  - Kubernetes cluster (1.20+)
  - Chaos Mesh installed
  - Sufficient resources (see [Deployment Guide](../deployment))

## Environment Configuration

AegisLab uses environment variables for flexible deployment across different environments. Before getting started, configure your environment:

### Create Environment File

Copy the example environment file and configure it for your setup:

```bash
# Copy example file
cp .env.example .env

# Edit with your values
nano .env
```

### Required Environment Variables

```bash
# AegisLab API Configuration
AEGISLAB_API_URL=http://10.10.10.220:8080

# TrainTicket Gateway Configuration
TRAINTICKET_GATEWAY_URL=http://10.10.10.220:30080

# JuiceFS Configuration (for dataset storage)
JUICEFS_REDIS_URL=redis://10.10.10.119:6379/1

# Environment Mode (dev, test, prod)
ENV_MODE=prod
```

### Environment Modes

- **dev**: Local development with localhost endpoints
- **test**: Development cluster (10.10.10.161)
- **prod**: Production cluster (10.10.10.220)

### Using Environment Variables

In Python code:

```python
import os
from rcabench.openapi import Configuration

config = Configuration(
    host=os.getenv("AEGISLAB_API_URL", "http://10.10.10.220:8080")
)
```

In shell scripts:

```bash
# Mount JuiceFS
sudo juicefs mount ${JUICEFS_REDIS_URL} /mnt/jfs -d

# Access API
curl ${AEGISLAB_API_URL}/api/v1/health
```

## Next Steps

1. **Configure environment**: Set up your `.env` file with appropriate values
2. **Choose your path**: Select [Algorithm Developer](../algorithm-developers) or [Dataset Creator](../dataset-creators)
3. **Follow the quickstart**: Complete the 10-15 minute quickstart guide for your role
4. **Explore examples**: Review example algorithms or fault injection scenarios
5. **Deploy infrastructure**: If needed, follow the [Deployment Guide](../deployment)

## Getting Help

- **Documentation**: Browse the guides for your chosen path
- **Examples**: Check the examples section for code snippets and walkthroughs
- **Troubleshooting**: Refer to troubleshooting sections in each guide
- **Community**: Contribute to the ecosystem via GitHub

Ready to get started? Choose your path above!
