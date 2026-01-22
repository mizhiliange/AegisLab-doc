---
title: Fault Types
weight: 2
---

Complete reference for all fault types supported by AegisLab. Each fault type includes parameters, examples, and use cases.

## Fault Type Naming

The implementation uses PascalCase names for fault types. The table below shows all supported fault types:

| Category | Fault Type | ChaosType Value | Description |
|----------|------------|-----------------|-------------|
| Pod | `PodKill` | 0 | Kill pod |
| Pod | `PodFailure` | 1 | Pod failure |
| Pod | `ContainerKill` | 2 | Kill container |
| Stress | `MemoryStress` | 3 | Memory stress |
| Stress | `CPUStress` | 4 | CPU stress |
| HTTP | `HTTPRequestAbort` | 5 | Abort HTTP request |
| HTTP | `HTTPResponseAbort` | 6 | Abort HTTP response |
| HTTP | `HTTPRequestDelay` | 7 | Delay HTTP request |
| HTTP | `HTTPResponseDelay` | 8 | Delay HTTP response |
| HTTP | `HTTPResponseReplaceBody` | 9 | Replace HTTP response body |
| HTTP | `HTTPResponsePatchBody` | 10 | Patch HTTP response body |
| HTTP | `HTTPRequestReplacePath` | 11 | Replace HTTP request path |
| HTTP | `HTTPRequestReplaceMethod` | 12 | Replace HTTP request method |
| HTTP | `HTTPResponseReplaceCode` | 13 | Replace HTTP response code |
| DNS | `DNSError` | 14 | DNS error |
| DNS | `DNSRandom` | 15 | DNS random response |
| Time | `TimeSkew` | 16 | Time skew |
| Network | `NetworkDelay` | 17 | Network delay |
| Network | `NetworkLoss` | 18 | Network packet loss |
| Network | `NetworkDuplicate` | 19 | Network packet duplication |
| Network | `NetworkCorrupt` | 20 | Network packet corruption |
| Network | `NetworkPartition` | 21 | Network partition |
| Network | `NetworkBandwidth` | 22 | Network bandwidth limit |
| IO | `IODelay` | 23 | IO delay |
| IO | `IOError` | 24 | IO error |
| JVM | `JVMException` | 25 | JVM exception injection |
| JVM | `JVMLatency` | 26 | JVM method latency |
| JVM | `JVMReturn` | 27 | JVM return value modification |
| JVM | `JVMStress` | 28 | JVM stress |
| JVM | `JVMGCStress` | 29 | JVM garbage collection stress |
| JVM | `JVMMySQLLatency` | 30 | JVM MySQL latency |
| JVM | `JVMMySQLException` | 31 | JVM MySQL exception |
| JVM | `JVMRuntimeMutator` | 32 | JVM runtime mutator |

## Network Faults

Network faults simulate communication issues between services.

### NetworkDelay

Add latency to network packets between services.

**ChaosNode Example**:
```python
from rcabench.openapi.models import ChaosNode

network_delay = ChaosNode(
    name="NetworkDelay",
    children={
        "service": ChaosNode(name="ts-order-service"),
        "namespace": ChaosNode(name="ts"),
        "delay": ChaosNode(name="200ms"),
        "jitter": ChaosNode(name="50ms"),  # Optional
    }
)
```

**Use Cases**:
- Simulate slow network connections
- Test timeout handling
- Study latency propagation in distributed systems

### NetworkLoss

Drop network packets randomly.

**ChaosNode Example**:
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

**Use Cases**:
- Test retry mechanisms
- Study impact of unreliable networks
- Validate error handling

### NetworkPartition

Isolate services from each other (split-brain scenarios).

**ChaosNode Example**:
```python
network_partition = ChaosNode(
    name="NetworkPartition",
    children={
        "service": ChaosNode(name="ts-user-service"),
        "namespace": ChaosNode(name="ts"),
        "direction": ChaosNode(name="both"),
    }
)
```

**Use Cases**:
- Test split-brain scenarios
- Validate partition tolerance
- Study cascading failures

### NetworkDuplicate

Duplicate network packets.

**ChaosNode Example**:
```python
network_duplicate = ChaosNode(
    name="NetworkDuplicate",
    children={
        "service": ChaosNode(name="ts-order-service"),
        "namespace": ChaosNode(name="ts"),
        "duplicate": ChaosNode(value=50),  # 50% duplication
    }
)
```

**Use Cases**:
- Test idempotency
- Validate duplicate detection
- Study message ordering issues

### NetworkCorrupt

Corrupt network packet data.

**ChaosNode Example**:
```python
network_corrupt = ChaosNode(
    name="NetworkCorrupt",
    children={
        "service": ChaosNode(name="ts-payment-service"),
        "namespace": ChaosNode(name="ts"),
        "corrupt": ChaosNode(value=5),  # 5% corruption
    }
)
```

**Use Cases**:
- Test data validation
- Study checksum handling
- Validate error detection

## Pod Faults

Pod faults simulate container and pod-level failures.

### PodKill

Terminate a pod immediately.

**ChaosNode Example**:
```python
pod_kill = ChaosNode(
    name="PodKill",
    children={
        "service": ChaosNode(name="ts-order-service"),
        "namespace": ChaosNode(name="ts"),
    }
)
```

**Use Cases**:
- Test service restart behavior
- Validate high availability
- Study recovery time

### PodFailure

Make pod fail health checks without killing it.

**ChaosNode Example**:
```python
pod_failure = ChaosNode(
    name="PodFailure",
    children={
        "service": ChaosNode(name="ts-payment-service"),
        "namespace": ChaosNode(name="ts"),
    }
)
```

**Use Cases**:
- Test health check handling
- Study load balancer behavior
- Validate graceful degradation

### ContainerKill

Kill specific container in a pod.

**ChaosNode Example**:
```python
container_kill = ChaosNode(
    name="ContainerKill",
    children={
        "service": ChaosNode(name="ts-user-service"),
        "namespace": ChaosNode(name="ts"),
        "container": ChaosNode(name="main"),  # Optional
    }
)
```

**Use Cases**:
- Test sidecar container failures
- Validate multi-container pod behavior
- Study container restart policies

## Stress Faults

Stress faults simulate resource exhaustion.

### CPUStress

Consume CPU resources.

**ChaosNode Example**:
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

**Use Cases**:
- Test CPU throttling
- Study performance degradation
- Validate resource limits

### MemoryStress

Consume memory resources.

**ChaosNode Example**:
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

**Use Cases**:
- Test OOM handling
- Study memory leak detection
- Validate memory limits

## JVM Faults

JVM faults inject failures at the application level (Java services only).

### JVMException

Throw exceptions in specific methods.

**ChaosNode Example**:
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

**Use Cases**:
- Test exception handling
- Study error propagation
- Validate circuit breakers

### JVMLatency

Add delays to method execution.

**ChaosNode Example**:
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

**Use Cases**:
- Test timeout handling
- Study latency propagation
- Validate async processing

### JVMReturn

Modify method return values.

**ChaosNode Example**:
```python
jvm_return = ChaosNode(
    name="JVMReturn",
    children={
        "service": ChaosNode(name="ts-user-service"),
        "namespace": ChaosNode(name="ts"),
        "class": ChaosNode(name="com.example.UserService"),
        "method": ChaosNode(name="getUserBalance"),
        "value": ChaosNode(name="0"),
    }
)
```

**Use Cases**:
- Test edge case handling
- Study data validation
- Validate business logic

## HTTP Faults

HTTP faults inject failures at the HTTP protocol level.

### HTTPRequestAbort

Return error responses for HTTP requests.

**ChaosNode Example**:
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

**Use Cases**:
- Test error handling
- Study retry mechanisms
- Validate fallback behavior

### HTTPResponseDelay

Add latency to HTTP responses.

**ChaosNode Example**:
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

**Use Cases**:
- Test timeout handling
- Study latency impact
- Validate async patterns

## Combining Multiple Faults

You can inject multiple faults simultaneously using the 2D specs array:

```python
from rcabench.openapi.models import SubmitInjectionReq, ChaosNode, ContainerSpec

# Parallel faults (injected simultaneously)
specs = [[
    ChaosNode(name="NetworkDelay", children={
        "service": ChaosNode(name="ts-order-service"),
        "namespace": ChaosNode(name="ts"),
        "delay": ChaosNode(name="100ms"),
    }),
    ChaosNode(name="CPUStress", children={
        "service": ChaosNode(name="ts-payment-service"),
        "namespace": ChaosNode(name="ts"),
        "workers": ChaosNode(value=2),
        "load": ChaosNode(value=80),
    }),
]]

request = SubmitInjectionReq(
    project_name="my-project",
    benchmark=ContainerSpec(name="trainticket"),
    pedestal=ContainerSpec(name="loadgenerator"),
    specs=specs,
    interval=5,
    pre_duration=1,
)
```

## Best Practices

1. **Start with single faults**: Test one fault type at a time before combining
2. **Use realistic parameters**: Match production failure patterns
3. **Consider cascading effects**: Some faults may trigger others
4. **Monitor system impact**: Watch for unexpected side effects
5. **Set appropriate duration**: Long enough to collect data, short enough to avoid damage

## Next Steps

- [HandlerNode Format](../handler-node-format): Complete ChaosNode reference
- [Workflow Overview](../workflow-overview): End-to-end process
- [Quickstart](../../quickstart): Submit your first injection
