---
title: HandlerNode Format
weight: 3
---

Complete reference for the ChaosNode structure used in fault injection requests.

## Overview

ChaosNode is the core data structure for specifying fault injections in AegisLab. It uses a tree structure where:
- **name**: The type of fault or parameter name
- **children**: Nested parameters as child nodes
- **value**: Numeric value for parameters with ranges
- **range**: Valid range constraints for values

## Basic Structure

The SDK uses the `ChaosNode` model:

```python
from rcabench.openapi.models import ChaosNode

# Basic structure
node = ChaosNode(
    name="NetworkDelay",  # Fault type name (PascalCase)
    children={
        "service": ChaosNode(name="ts-order-service"),
        "delay": ChaosNode(name="100ms"),
        "namespace": ChaosNode(name="ts")
    }
)
```

## ChaosNode Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Fault type or parameter value |
| `children` | dict[str, ChaosNode] | Nested parameters |
| `value` | int | Numeric value (for ranged parameters) |
| `range` | list[int] | Valid range [min, max] for value |
| `description` | string | Human-readable description |

## Fault Type Names

The implementation uses PascalCase names for fault types:

| Fault Type | ChaosType Value | Description |
|------------|-----------------|-------------|
| `PodKill` | 0 | Kill pod |
| `PodFailure` | 1 | Pod failure |
| `ContainerKill` | 2 | Kill container |
| `MemoryStress` | 3 | Memory stress |
| `CPUStress` | 4 | CPU stress |
| `HTTPRequestAbort` | 5 | Abort HTTP request |
| `HTTPResponseAbort` | 6 | Abort HTTP response |
| `HTTPRequestDelay` | 7 | Delay HTTP request |
| `HTTPResponseDelay` | 8 | Delay HTTP response |
| `NetworkDelay` | 17 | Network delay |
| `NetworkLoss` | 18 | Network packet loss |
| `NetworkDuplicate` | 19 | Network packet duplication |
| `NetworkCorrupt` | 20 | Network packet corruption |
| `JVMException` | 25 | JVM exception injection |
| `JVMLatency` | 26 | JVM method latency |
| `JVMReturn` | 27 | JVM return value modification |

## Network Fault Examples

### Network Delay

```python
from rcabench.openapi.models import ChaosNode

network_delay = ChaosNode(
    name="NetworkDelay",
    children={
        "service": ChaosNode(name="ts-order-service"),
        "namespace": ChaosNode(name="ts"),
        "delay": ChaosNode(name="100ms"),
        "jitter": ChaosNode(name="10ms"),  # Optional
    }
)
```

### Network Loss

```python
network_loss = ChaosNode(
    name="NetworkLoss",
    children={
        "service": ChaosNode(name="ts-payment-service"),
        "namespace": ChaosNode(name="ts"),
        "loss": ChaosNode(value=10),  # 10% packet loss
    }
)
```

## Pod Fault Examples

### Pod Kill

```python
pod_kill = ChaosNode(
    name="PodKill",
    children={
        "service": ChaosNode(name="ts-order-service"),
        "namespace": ChaosNode(name="ts"),
    }
)
```

### Pod Failure

```python
pod_failure = ChaosNode(
    name="PodFailure",
    children={
        "service": ChaosNode(name="ts-payment-service"),
        "namespace": ChaosNode(name="ts"),
    }
)
```

## Stress Fault Examples

### CPU Stress

```python
cpu_stress = ChaosNode(
    name="CPUStress",
    children={
        "service": ChaosNode(name="ts-order-service"),
        "namespace": ChaosNode(name="ts"),
        "workers": ChaosNode(value=2),
        "load": ChaosNode(value=80),  # 80% CPU load
    }
)
```

### Memory Stress

```python
memory_stress = ChaosNode(
    name="MemoryStress",
    children={
        "service": ChaosNode(name="ts-payment-service"),
        "namespace": ChaosNode(name="ts"),
        "size": ChaosNode(name="512MB"),
    }
)
```

## JVM Fault Examples

### JVM Exception

```python
jvm_exception = ChaosNode(
    name="JVMException",
    children={
        "service": ChaosNode(name="ts-order-service"),
        "namespace": ChaosNode(name="ts"),
        "class": ChaosNode(name="com.example.OrderService"),
        "method": ChaosNode(name="createOrder"),
        "exception": ChaosNode(name="java.lang.RuntimeException"),
    }
)
```

### JVM Latency

```python
jvm_latency = ChaosNode(
    name="JVMLatency",
    children={
        "service": ChaosNode(name="ts-payment-service"),
        "namespace": ChaosNode(name="ts"),
        "class": ChaosNode(name="com.example.PaymentService"),
        "method": ChaosNode(name="processPayment"),
        "latency": ChaosNode(value=5000),  # 5000ms delay
    }
)
```

## HTTP Fault Examples

### HTTP Request Abort

```python
http_abort = ChaosNode(
    name="HTTPRequestAbort",
    children={
        "service": ChaosNode(name="ts-order-service"),
        "namespace": ChaosNode(name="ts"),
        "code": ChaosNode(value=503),
        "path": ChaosNode(name="/api/orders"),
    }
)
```

### HTTP Response Delay

```python
http_delay = ChaosNode(
    name="HTTPResponseDelay",
    children={
        "service": ChaosNode(name="ts-payment-service"),
        "namespace": ChaosNode(name="ts"),
        "delay": ChaosNode(name="2s"),
        "path": ChaosNode(name="/api/payment"),
    }
)
```

## Submitting Fault Injections

Use the `SubmitInjectionReq` model with a 2D array of ChaosNodes:

```python
from rcabench.openapi import ApiClient, Configuration
from rcabench.openapi.api import InjectionsApi
from rcabench.openapi.models import SubmitInjectionReq, ChaosNode, ContainerSpec

config = Configuration(host="${AEGISLAB_API_URL}")
client = ApiClient(config)
api = InjectionsApi(client)

# Define fault specs
fault_spec = ChaosNode(
    name="NetworkDelay",
    children={
        "service": ChaosNode(name="ts-order-service"),
        "namespace": ChaosNode(name="ts"),
        "delay": ChaosNode(name="100ms"),
    }
)

# Submit injection
request = SubmitInjectionReq(
    project_name="my-project",
    benchmark=ContainerSpec(name="trainticket"),
    pedestal=ContainerSpec(name="loadgenerator"),
    specs=[[fault_spec]],  # 2D array: each sub-array is a batch
    interval=5,  # Total experiment interval in minutes
    pre_duration=1,  # Normal data collection before fault (minutes)
)

response = api.inject_fault(request)
```

## Multiple Faults (Parallel)

To inject multiple faults in parallel, include them in the same sub-array:

```python
# Parallel faults (injected simultaneously)
specs = [[
    ChaosNode(name="NetworkDelay", children={
        "service": ChaosNode(name="ts-order-service"),
        "delay": ChaosNode(name="100ms"),
    }),
    ChaosNode(name="CPUStress", children={
        "service": ChaosNode(name="ts-payment-service"),
        "workers": ChaosNode(value=2),
    }),
]]
```

## Sequential Faults (Batches)

To inject faults sequentially, use separate sub-arrays:

```python
# Sequential faults (injected one after another)
specs = [
    [ChaosNode(name="NetworkDelay", children={...})],  # Batch 1
    [ChaosNode(name="CPUStress", children={...})],     # Batch 2
    [ChaosNode(name="PodKill", children={...})],       # Batch 3
]
```

## Validation

The system validates ChaosNode structures against reference schemas. Common validation errors:

**Missing required child**:
```
Error: Missing required child 'service' for fault type 'NetworkDelay'
```

**Invalid value range**:
```
Error: Value 150 out of range [0, 100] for parameter 'loss'
```

**Unknown fault type**:
```
Error: Unknown fault type 'InvalidFault'
```

## Best Practices

1. **Use PascalCase**: Fault type names use PascalCase (e.g., `NetworkDelay`, not `network_delay`)
2. **Include namespace**: Always specify the target namespace
3. **Start simple**: Test single faults before combining
4. **Use realistic values**: Match production failure patterns
5. **Validate locally**: Test ChaosNode structure before submission

## Next Steps

- [Fault Types](../fault-types): Detailed fault type reference
- [Workflow Overview](../workflow-overview): End-to-end process
- [Quickstart](../../quickstart): Submit your first injection
