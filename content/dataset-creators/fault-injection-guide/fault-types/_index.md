---
title: Fault Types
weight: 2
---

Complete reference for all fault types supported by AegisLab. Each fault type includes parameters, examples, and use cases.

## Network Faults

Network faults simulate communication issues between services.

### Network Delay

Add latency to network packets between services.

**Handler**: `network_delay`

**Parameters**:
- `delay` (required): Delay duration (e.g., "100ms", "1s")
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace (default: "ts")
- `jitter` (optional): Random variation in delay (e.g., "10ms")
- `correlation` (optional): Correlation between delays (0-100)

**Example**:
```python
{
    "handler": "network_delay",
    "params": {
        "delay": "200ms",
        "target_service": "ts-order-service",
        "target_namespace": "ts",
        "jitter": "50ms"
    }
}
```

**Use Cases**:
- Simulate slow network connections
- Test timeout handling
- Study latency propagation in distributed systems

### Network Loss

Drop network packets randomly.

**Handler**: `network_loss`

**Parameters**:
- `loss` (required): Packet loss percentage (0-100)
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `correlation` (optional): Correlation between losses (0-100)

**Example**:
```python
{
    "handler": "network_loss",
    "params": {
        "loss": "10",
        "target_service": "ts-payment-service",
        "correlation": "25"
    }
}
```

**Use Cases**:
- Test retry mechanisms
- Study impact of unreliable networks
- Validate error handling

### Network Partition

Isolate services from each other (split-brain scenarios).

**Handler**: `network_partition`

**Parameters**:
- `target_service` (required): Service to isolate
- `target_namespace` (optional): Kubernetes namespace
- `direction` (optional): "to", "from", or "both" (default: "both")
- `external_targets` (optional): List of external IPs/domains to block

**Example**:
```python
{
    "handler": "network_partition",
    "params": {
        "target_service": "ts-user-service",
        "direction": "both"
    }
}
```

**Use Cases**:
- Test split-brain scenarios
- Validate partition tolerance
- Study cascading failures

### Network Duplicate

Duplicate network packets.

**Handler**: `network_duplicate`

**Parameters**:
- `duplicate` (required): Duplication percentage (0-100)
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `correlation` (optional): Correlation between duplications

**Example**:
```python
{
    "handler": "network_duplicate",
    "params": {
        "duplicate": "50",
        "target_service": "ts-order-service"
    }
}
```

**Use Cases**:
- Test idempotency
- Validate duplicate detection
- Study message ordering issues

### Network Corrupt

Corrupt network packet data.

**Handler**: `network_corrupt`

**Parameters**:
- `corrupt` (required): Corruption percentage (0-100)
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `correlation` (optional): Correlation between corruptions

**Example**:
```python
{
    "handler": "network_corrupt",
    "params": {
        "corrupt": "5",
        "target_service": "ts-payment-service"
    }
}
```

**Use Cases**:
- Test data validation
- Study checksum handling
- Validate error detection

## Pod Faults

Pod faults simulate container and pod-level failures.

### Pod Kill

Terminate a pod immediately.

**Handler**: `pod_kill`

**Parameters**:
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `grace_period` (optional): Termination grace period in seconds (default: 0)

**Example**:
```python
{
    "handler": "pod_kill",
    "params": {
        "target_service": "ts-order-service",
        "grace_period": 5
    }
}
```

**Use Cases**:
- Test service restart behavior
- Validate high availability
- Study recovery time

### Pod Failure

Make pod fail health checks without killing it.

**Handler**: `pod_failure`

**Parameters**:
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace

**Example**:
```python
{
    "handler": "pod_failure",
    "params": {
        "target_service": "ts-payment-service"
    }
}
```

**Use Cases**:
- Test health check handling
- Study load balancer behavior
- Validate graceful degradation

### Container Kill

Kill specific container in a pod.

**Handler**: `container_kill`

**Parameters**:
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `container_name` (optional): Specific container to kill

**Example**:
```python
{
    "handler": "container_kill",
    "params": {
        "target_service": "ts-user-service",
        "container_name": "main"
    }
}
```

**Use Cases**:
- Test sidecar container failures
- Validate multi-container pod behavior
- Study container restart policies

## Stress Faults

Stress faults simulate resource exhaustion.

### CPU Stress

Consume CPU resources.

**Handler**: `cpu_stress`

**Parameters**:
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `workers` (required): Number of CPU workers
- `load` (optional): CPU load percentage per worker (0-100)

**Example**:
```python
{
    "handler": "cpu_stress",
    "params": {
        "target_service": "ts-order-service",
        "workers": 2,
        "load": 80
    }
}
```

**Use Cases**:
- Test CPU throttling
- Study performance degradation
- Validate resource limits

### Memory Stress

Consume memory resources.

**Handler**: `memory_stress`

**Parameters**:
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `size` (required): Memory to consume (e.g., "256MB", "1GB")

**Example**:
```python
{
    "handler": "memory_stress",
    "params": {
        "target_service": "ts-payment-service",
        "size": "512MB"
    }
}
```

**Use Cases**:
- Test OOM handling
- Study memory leak detection
- Validate memory limits

## JVM Faults

JVM faults inject failures at the application level (Java services only).

### JVM Exception

Throw exceptions in specific methods.

**Handler**: `jvm_exception`

**Parameters**:
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `class` (required): Fully qualified class name
- `method` (required): Method name
- `exception` (required): Exception class to throw

**Example**:
```python
{
    "handler": "jvm_exception",
    "params": {
        "target_service": "ts-order-service",
        "class": "com.example.OrderService",
        "method": "createOrder",
        "exception": "java.lang.RuntimeException"
    }
}
```

**Use Cases**:
- Test exception handling
- Study error propagation
- Validate circuit breakers

### JVM Latency

Add delays to method execution.

**Handler**: `jvm_latency`

**Parameters**:
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `class` (required): Fully qualified class name
- `method` (required): Method name
- `latency` (required): Delay duration (milliseconds)

**Example**:
```python
{
    "handler": "jvm_latency",
    "params": {
        "target_service": "ts-payment-service",
        "class": "com.example.PaymentService",
        "method": "processPayment",
        "latency": 5000
    }
}
```

**Use Cases**:
- Test timeout handling
- Study latency propagation
- Validate async processing

### JVM Return Value

Modify method return values.

**Handler**: `jvm_return`

**Parameters**:
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `class` (required): Fully qualified class name
- `method` (required): Method name
- `value` (required): Return value to inject

**Example**:
```python
{
    "handler": "jvm_return",
    "params": {
        "target_service": "ts-user-service",
        "class": "com.example.UserService",
        "method": "getUserBalance",
        "value": "0"
    }
}
```

**Use Cases**:
- Test edge case handling
- Study data validation
- Validate business logic

## HTTP Faults

HTTP faults inject failures at the HTTP protocol level.

### HTTP Abort

Return error responses for HTTP requests.

**Handler**: `http_abort`

**Parameters**:
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `status_code` (required): HTTP status code to return (e.g., 500, 503)
- `path` (optional): URL path pattern to match
- `percentage` (optional): Percentage of requests to abort (0-100)

**Example**:
```python
{
    "handler": "http_abort",
    "params": {
        "target_service": "ts-order-service",
        "status_code": 503,
        "path": "/api/orders",
        "percentage": 50
    }
}
```

**Use Cases**:
- Test error handling
- Study retry mechanisms
- Validate fallback behavior

### HTTP Delay

Add latency to HTTP requests.

**Handler**: `http_delay`

**Parameters**:
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `delay` (required): Delay duration (e.g., "1s", "500ms")
- `path` (optional): URL path pattern to match
- `percentage` (optional): Percentage of requests to delay

**Example**:
```python
{
    "handler": "http_delay",
    "params": {
        "target_service": "ts-payment-service",
        "delay": "2s",
        "path": "/api/payment",
        "percentage": 100
    }
}
```

**Use Cases**:
- Test timeout handling
- Study latency impact
- Validate async patterns

### HTTP Replace

Modify HTTP request or response content.

**Handler**: `http_replace`

**Parameters**:
- `target_service` (required): Target service name
- `target_namespace` (optional): Kubernetes namespace
- `path` (optional): URL path pattern to match
- `replace_body` (optional): New response body
- `replace_headers` (optional): Headers to modify

**Example**:
```python
{
    "handler": "http_replace",
    "params": {
        "target_service": "ts-user-service",
        "path": "/api/user",
        "replace_body": '{"error": "Service unavailable"}'
    }
}
```

**Use Cases**:
- Test data validation
- Study error message handling
- Validate content parsing

## Combining Multiple Faults

You can inject multiple faults simultaneously by providing multiple handler nodes:

```python
injection_req = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[
        {
            "handler": "network_delay",
            "params": {
                "delay": "100ms",
                "target_service": "ts-order-service"
            }
        },
        {
            "handler": "cpu_stress",
            "params": {
                "target_service": "ts-payment-service",
                "workers": 2,
                "load": 80
            }
        }
    ],
    duration=60
)
```

## Best Practices

1. **Start with single faults**: Test one fault type at a time before combining
2. **Use realistic parameters**: Match production failure patterns
3. **Consider cascading effects**: Some faults may trigger others
4. **Monitor system impact**: Watch for unexpected side effects
5. **Set appropriate duration**: Long enough to collect data, short enough to avoid damage

## Next Steps

- [HandlerNode Format](handler-node-format): Complete parameter reference
- [Using AegisLab](../using-aegislab): Submit fault injections via API
- [Best Practices](best-practices): Experiment design guidelines
