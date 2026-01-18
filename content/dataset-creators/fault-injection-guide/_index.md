---
title: Fault Injection Guide
weight: 2
---

# Fault Injection Guide

This guide covers the fundamentals of fault injection in the AegisLab ecosystem, including fault types, parameters, and best practices for creating high-quality datasets.

## Topics

{{< cards >}}
  {{< card link="workflow-overview" title="Workflow Overview" icon="document" >}}
  {{< card link="fault-types" title="Fault Types" icon="beaker" >}}
  {{< card link="handler-node-format" title="HandlerNode Format" icon="code" >}}
  {{< card link="intelligent-scheduling" title="Intelligent Scheduling" icon="sparkles" >}}
  {{< card link="best-practices" title="Best Practices" icon="sparkles" >}}
{{< /cards >}}

## Overview

Fault injection in AegisLab follows a structured workflow:

```
Design Experiment → Submit via API → Execute Chaos → Collect Data → Generate Dataset
```

### Key Components

1. **Fault Specification**: Define fault type, target service, and parameters
2. **Execution**: Chaos Mesh applies faults to Kubernetes pods
3. **Observability**: OpenTelemetry captures traces, metrics, and logs
4. **Data Processing**: Convert raw observability data to standardized parquet format
5. **Dataset Storage**: Store datasets with ground truth metadata

## Fault Injection Lifecycle

### 1. Design Phase

Choose appropriate fault types based on your research goals:

- **Network faults** for studying communication failures
- **Pod faults** for testing resilience to service crashes
- **Stress faults** for resource exhaustion scenarios
- **JVM faults** for application-level failures
- **HTTP faults** for API-level issues

### 2. Submission Phase

Submit fault injection requests via AegisLab API:

```python
from rcabench.openapi import ApiClient, FaultInjectionApi
from rcabench.openapi.models import DtoSubmitInjectionReq

injection_req = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[
        {
            "handler": "network_delay",
            "params": {
                "delay": "100ms",
                "target_service": "ts-order-service"
            }
        }
    ],
    duration=60
)

response = api.submit_injection(injection_req)
```

### 3. Execution Phase

AegisLab orchestrates the fault injection:

1. Validates fault specification
2. Generates Chaos Mesh CRDs
3. Applies chaos to target pods
4. Monitors fault execution
5. Cleans up after duration expires

### 4. Collection Phase

Observability data is automatically collected:

- **Traces**: Distributed traces from OpenTelemetry
- **Metrics**: Resource usage, latency, error rates
- **Logs**: Application logs with context
- **Ground Truth**: Fault injection metadata

### 5. Dataset Generation

Raw data is processed into standardized format:

- Convert to parquet files
- Add ground truth labels
- Validate data completeness
- Store in dataset repository

## Fault Types Overview

AegisLab supports five categories of faults:

### Network Faults

Simulate network issues between services:

- **Delay**: Add latency to network packets
- **Loss**: Drop packets randomly
- **Duplicate**: Duplicate packets
- **Corrupt**: Corrupt packet data
- **Partition**: Isolate services from each other

### Pod Faults

Simulate pod-level failures:

- **Kill**: Terminate pod immediately
- **Failure**: Make pod fail health checks
- **Container Kill**: Kill specific container in pod

### Stress Faults

Simulate resource exhaustion:

- **CPU Stress**: Consume CPU resources
- **Memory Stress**: Consume memory resources

### JVM Faults

Inject faults at application level (Java services):

- **Exception**: Throw exceptions in specific methods
- **Latency**: Add delays to method execution
- **Return Value**: Modify method return values

### HTTP Faults

Inject faults at HTTP layer:

- **Abort**: Return error responses
- **Delay**: Add latency to HTTP requests
- **Replace**: Modify request/response content

## HandlerNode Structure

Fault injections use the `HandlerNode` format:

```python
{
    "handler": "fault_type",  # Type of fault to inject
    "params": {               # Fault-specific parameters
        "target_service": "service-name",
        "target_namespace": "namespace",
        # Additional parameters based on fault type
    }
}
```

### Example: Network Delay

```python
{
    "handler": "network_delay",
    "params": {
        "delay": "100ms",
        "target_service": "ts-order-service",
        "target_namespace": "ts",
        "jitter": "10ms"
    }
}
```

### Example: Pod Kill

```python
{
    "handler": "pod_kill",
    "params": {
        "target_service": "ts-payment-service",
        "target_namespace": "ts"
    }
}
```

### Example: CPU Stress

```python
{
    "handler": "cpu_stress",
    "params": {
        "target_service": "ts-user-service",
        "target_namespace": "ts",
        "workers": 2,
        "load": 80
    }
}
```

## Intelligent Scheduling with Pandora

Pandora uses genetic algorithms to evolve fault scenarios:

### How It Works

1. **Bootstrap**: Generate random fault scenarios
2. **Evaluate**: Measure SLO violations and diagnostic difficulty
3. **Select**: Choose high-fitness scenarios as parents
4. **Crossover**: Combine parent scenarios
5. **Mutate**: Randomly modify parameters
6. **Repeat**: Evolve over multiple generations

### Fitness Function

Pandora optimizes for:

- **SLO Score**: Maximize latency increases and error rates
- **Diagnosis Score**: Minimize MRR (harder to diagnose)

### Configuration

```toml
[algorithm.fitness]
slo_weight = 0.5
diagnosis_weight = 0.5
timeout = 300

[algorithm.ga]
population_size = 50
generations = 20
mutation_rate = 0.1
crossover_rate = 0.8
```

## Best Practices

### Experiment Design

1. **Start simple**: Begin with single-fault scenarios
2. **Isolate variables**: Change one parameter at a time
3. **Use realistic parameters**: Match production failure patterns
4. **Consider timing**: Inject faults during peak load

### Fault Selection

1. **Target critical paths**: Focus on services in request path
2. **Vary severity**: Test both minor and severe faults
3. **Combine faults**: Test cascading failures
4. **Include recovery**: Test system recovery behavior

### Data Quality

1. **Verify collection**: Check traces are captured during fault window
2. **Validate ground truth**: Ensure fault metadata is accurate
3. **Check completeness**: Verify all expected services are present
4. **Monitor load**: Ensure sufficient traffic during injection

### Safety Considerations

1. **Use dedicated environments**: Never inject faults in production
2. **Set timeouts**: Always specify fault duration
3. **Monitor impact**: Watch for unexpected cascading failures
4. **Have rollback plan**: Be ready to cancel injection if needed

## Next Steps

- [Fault Types](fault-types): Detailed reference for all fault types
- [HandlerNode Format](handler-node-format): Complete parameter reference
- [Using AegisLab](../using-aegislab): Submit injections via API
- [Intelligent Scheduling](intelligent-scheduling): Deep dive into Pandora
