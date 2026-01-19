---
title: Fault Catalog
weight: 2
---

Complete catalog of available fault types and their parameters.

## Network Faults

### network_delay

Add network latency between services.

**Handler:** `network_delay`

**Parameters:**
- `delay` (required): Latency amount (e.g., "100ms", "1s")
- `jitter` (optional): Latency variation (e.g., "10ms")
- `correlation` (optional): Correlation coefficient (0-100)
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace
- `direction` (optional): "to", "from", or "both" (default: "to")

**Example:**

```python
{
    "handler": "network_delay",
    "params": {
        "delay": "100ms",
        "jitter": "10ms",
        "target_service": "ts-order-service",
        "target_namespace": "ts"
    }
}
```

### network_loss

Drop network packets.

**Handler:** `network_loss`

**Parameters:**
- `loss` (required): Packet loss percentage (0-100)
- `correlation` (optional): Correlation coefficient (0-100)
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace

**Example:**

```python
{
    "handler": "network_loss",
    "params": {
        "loss": "25",
        "target_service": "ts-payment-service",
        "target_namespace": "ts"
    }
}
```

### network_partition

Create network partition between services.

**Handler:** `network_partition`

**Parameters:**
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace
- `partition_services` (required): List of services to partition from

**Example:**

```python
{
    "handler": "network_partition",
    "params": {
        "target_service": "ts-order-service",
        "target_namespace": "ts",
        "partition_services": ["ts-payment-service", "ts-user-service"]
    }
}
```

## Pod Faults

### pod_failure

Kill pod (restarts automatically).

**Handler:** `pod_failure`

**Parameters:**
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace
- `mode` (optional): "one", "all", "fixed", "fixed-percent", "random-max-percent" (default: "one")

**Example:**

```python
{
    "handler": "pod_failure",
    "params": {
        "target_service": "ts-order-service",
        "target_namespace": "ts",
        "mode": "one"
    }
}
```

### pod_kill

Kill pod permanently (no restart).

**Handler:** `pod_kill`

**Parameters:**
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace

**Example:**

```python
{
    "handler": "pod_kill",
    "params": {
        "target_service": "ts-notification-service",
        "target_namespace": "ts"
    }
}
```

### container_kill

Kill specific container in pod.

**Handler:** `container_kill`

**Parameters:**
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace
- `container_name` (required): Container name to kill

**Example:**

```python
{
    "handler": "container_kill",
    "params": {
        "target_service": "ts-order-service",
        "target_namespace": "ts",
        "container_name": "order-service"
    }
}
```

## Resource Faults

### memory_pressure

Inject memory stress.

**Handler:** `memory_pressure`

**Parameters:**
- `size` (required): Memory to consume (e.g., "256MB", "1GB")
- `workers` (optional): Number of workers (default: 4)
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace

**Example:**

```python
{
    "handler": "memory_pressure",
    "params": {
        "size": "512MB",
        "workers": 4,
        "target_service": "ts-order-service",
        "target_namespace": "ts"
    }
}
```

### cpu_stress

Inject CPU stress.

**Handler:** `cpu_stress`

**Parameters:**
- `workers` (required): Number of CPU workers
- `load` (optional): CPU load percentage per worker (0-100)
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace

**Example:**

```python
{
    "handler": "cpu_stress",
    "params": {
        "workers": 4,
        "load": 80,
        "target_service": "ts-payment-service",
        "target_namespace": "ts"
    }
}
```

### disk_pressure

Fill disk space.

**Handler:** `disk_pressure`

**Parameters:**
- `size` (required): Disk space to fill (e.g., "1GB", "500MB")
- `path` (optional): Path to fill (default: "/tmp")
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace

**Example:**

```python
{
    "handler": "disk_pressure",
    "params": {
        "size": "1GB",
        "path": "/tmp",
        "target_service": "ts-order-service",
        "target_namespace": "ts"
    }
}
```

## JVM Faults

### jvm_exception

Throw exception in JVM method.

**Handler:** `jvm_exception`

**Parameters:**
- `class` (required): Java class name
- `method` (required): Method name
- `exception` (required): Exception to throw
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace

**Example:**

```python
{
    "handler": "jvm_exception",
    "params": {
        "class": "com.trainticket.service.OrderService",
        "method": "createOrder",
        "exception": "java.lang.RuntimeException('Injected fault')",
        "target_service": "ts-order-service",
        "target_namespace": "ts"
    }
}
```

### jvm_latency

Add latency to JVM method.

**Handler:** `jvm_latency`

**Parameters:**
- `class` (required): Java class name
- `method` (required): Method name
- `latency` (required): Latency in milliseconds
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace

**Example:**

```python
{
    "handler": "jvm_latency",
    "params": {
        "class": "com.trainticket.service.PaymentService",
        "method": "processPayment",
        "latency": 1000,
        "target_service": "ts-payment-service",
        "target_namespace": "ts"
    }
}
```

### jvm_return

Modify method return value.

**Handler:** `jvm_return`

**Parameters:**
- `class` (required): Java class name
- `method` (required): Method name
- `value` (required): Return value to inject
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace

**Example:**

```python
{
    "handler": "jvm_return",
    "params": {
        "class": "com.trainticket.service.UserService",
        "method": "getUser",
        "value": "null",
        "target_service": "ts-user-service",
        "target_namespace": "ts"
    }
}
```

## Database Faults

### database_latency

Add latency to database connections.

**Handler:** `database_latency`

**Parameters:**
- `delay` (required): Latency amount (e.g., "100ms")
- `target_service` (required): Service with database connection
- `target_namespace` (required): Kubernetes namespace
- `database_type` (optional): "postgres", "mysql", "mongodb" (default: auto-detect)

**Example:**

```python
{
    "handler": "database_latency",
    "params": {
        "delay": "200ms",
        "target_service": "ts-order-service",
        "target_namespace": "ts",
        "database_type": "mysql"
    }
}
```

### database_error

Inject database errors.

**Handler:** `database_error`

**Parameters:**
- `error_rate` (required): Error rate percentage (0-100)
- `target_service` (required): Service with database connection
- `target_namespace` (required): Kubernetes namespace

**Example:**

```python
{
    "handler": "database_error",
    "params": {
        "error_rate": 50,
        "target_service": "ts-user-service",
        "target_namespace": "ts"
    }
}
```

## HTTP Faults

### http_abort

Abort HTTP requests.

**Handler:** `http_abort`

**Parameters:**
- `status_code` (required): HTTP status code to return
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace
- `path` (optional): URL path pattern (default: all paths)

**Example:**

```python
{
    "handler": "http_abort",
    "params": {
        "status_code": 500,
        "target_service": "ts-order-service",
        "target_namespace": "ts",
        "path": "/api/v1/orders"
    }
}
```

### http_delay

Add HTTP request latency.

**Handler:** `http_delay`

**Parameters:**
- `delay` (required): Latency amount (e.g., "500ms")
- `target_service` (required): Target service name
- `target_namespace` (required): Kubernetes namespace
- `path` (optional): URL path pattern

**Example:**

```python
{
    "handler": "http_delay",
    "params": {
        "delay": "1s",
        "target_service": "ts-payment-service",
        "target_namespace": "ts",
        "path": "/api/v1/payments"
    }
}
```

## Composite Faults

### cascade_failure

Trigger cascading failures across services.

**Handler:** `cascade_failure`

**Parameters:**
- `initial_service` (required): Service to start cascade
- `target_namespace` (required): Kubernetes namespace
- `propagation_delay` (optional): Delay between cascades (default: "10s")

**Example:**

```python
{
    "handler": "cascade_failure",
    "params": {
        "initial_service": "ts-order-service",
        "target_namespace": "ts",
        "propagation_delay": "15s"
    }
}
```

## Parameter Guidelines

### Delay Values

Format: `<number><unit>`

Units:
- `ms`: milliseconds
- `s`: seconds
- `m`: minutes

Examples: `100ms`, `1s`, `2m`

### Memory/Disk Sizes

Format: `<number><unit>`

Units:
- `KB`: kilobytes
- `MB`: megabytes
- `GB`: gigabytes

Examples: `256MB`, `1GB`, `500KB`

### Percentages

Format: Integer 0-100

Examples: `25`, `50`, `100`

### Service Names

Must match Kubernetes deployment/pod labels.

Check with:
```bash
kubectl get pods -n ts --show-labels
```

## Fault Combinations

Multiple faults can be combined:

```python
{
    "handler_nodes": [
        {
            "handler": "network_delay",
            "params": {
                "delay": "100ms",
                "target_service": "ts-order-service",
                "target_namespace": "ts"
            }
        },
        {
            "handler": "memory_pressure",
            "params": {
                "size": "512MB",
                "target_service": "ts-payment-service",
                "target_namespace": "ts"
            }
        }
    ]
}
```

## Best Practices

1. **Start mild**: Begin with low severity (small delays, low error rates)
2. **Single fault first**: Test one fault type before combining
3. **Verify target**: Ensure service name and namespace are correct
4. **Monitor impact**: Watch metrics during injection
5. **Set duration**: Always specify fault duration
6. **Document parameters**: Keep track of effective parameter values

## See Also

- [Fault Injection Guide](../../fault-injection-guide): Basic concepts
- [Handler Node Format](../../fault-injection-guide/handler-node-format): Request format
- [SDK Examples](../sdk-examples): Code examples
