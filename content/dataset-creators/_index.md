---
title: Dataset Creator Guide
weight: 2
---

Welcome to the Dataset Creator path! This guide helps you create high-quality datasets through intelligent fault injection experiments in microservices systems.

## What You'll Learn

- How to execute fault injection via AegisLab API
- Understanding fault types and parameters
- Using genetic algorithms for intelligent scheduling
- Monitoring data collection progress
- Retrieving and managing generated datasets

## Quick Navigation

{{< cards >}}
  {{< card link="quickstart" title="Quickstart" icon="play" >}}
  {{< card link="fault-injection-guide" title="Fault Injection Guide" icon="beaker" >}}
  {{< card link="using-aegislab" title="Using AegisLab" icon="document" >}}
  {{< card link="monitoring-collection" title="Monitoring & Collection" icon="document" >}}
  {{< card link="dataset-management" title="Dataset Management" icon="document" >}}
  {{< card link="advanced-topics" title="Advanced Topics" icon="sparkles" >}}
{{< /cards >}}

## Typical Workflow

1. **Setup Environment**: Install AegisLab SDK and configure credentials
2. **Design Experiment**: Choose fault types and target services
3. **Submit Injection**: Use AegisLab API to execute fault injection
4. **Monitor Collection**: Track trace events and data collection progress
5. **Retrieve Dataset**: Download standardized parquet files
6. **Quality Check**: Validate collected data completeness

## Key Concepts

### Fault Injection Lifecycle

```
Design → Submit → Execute → Collect → Store → Retrieve
```

1. **Design**: Define fault types, parameters, and target services
2. **Submit**: Send injection request via AegisLab API
3. **Execute**: Chaos Mesh applies faults to Kubernetes pods
4. **Collect**: OpenTelemetry captures traces, metrics, and logs
5. **Store**: Data converted to standardized parquet format
6. **Retrieve**: Download datasets for algorithm evaluation

### Fault Types

AegisLab supports multiple fault types:

- **Network Faults**: Delay, loss, duplicate, corrupt, partition
- **Pod Faults**: Kill, failure, container kill
- **Stress Faults**: CPU stress, memory stress
- **JVM Faults**: Exception injection, latency injection, return value modification
- **HTTP Faults**: Abort, delay, replace

### HandlerNode Format

Fault injection requests use the `HandlerNode` structure:

```python
{
    "handler": "network_delay",  # Fault type
    "params": {
        "delay": "100ms",
        "target_service": "ts-order-service"
    }
}
```

### Intelligent Scheduling with Pandora

Pandora uses genetic algorithms to evolve fault scenarios that:
- Maximize SLO violations (latency, error rate)
- Increase diagnostic difficulty (lower MRR scores)
- Discover edge cases and complex failure modes

## Prerequisites

- Python 3.10+
- Access to AegisLab API endpoint
- Basic understanding of microservices and Kubernetes
- Familiarity with distributed tracing concepts

## Next Steps

Start with the [Quickstart Guide](quickstart) to submit your first fault injection in under 10 minutes.
