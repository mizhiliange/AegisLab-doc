---
title: HandlerNode Format
weight: 3
---

Complete reference for the HandlerNode structure used in fault injection requests.

## Overview

HandlerNode is the core data structure for specifying fault injections in AegisLab. Each HandlerNode defines:
- **handler**: The type of fault to inject
- **params**: Fault-specific parameters

## Basic Structure

```python
{
    "handler": "fault_type",  # Required: type of fault
    "params": {               # Required: fault parameters
        "target_service": "service-name",
        "target_namespace": "namespace",
        # Additional fault-specific parameters
    }
}
```

## Common Parameters

These parameters are common across most fault types:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `target_service` | string | Yes | Name of the target service |
| `target_namespace` | string | No | Kubernetes namespace (default: "ts") |
| `mode` | string | No | Selection mode: "one", "all", "fixed", "percent" |
| `value` | string | No | Value for mode (e.g., "2" for fixed, "50" for percent) |

### Selection Modes

Control how many pods are affected:

```python
# Affect one random pod (default)
{"mode": "one"}

# Affect all pods
{"mode": "all"}

# Affect fixed number of pods
{"mode": "fixed", "value": "2"}

# Affect percentage of pods
{"mode": "percent", "value": "50"}
```

## Network Fault Parameters

### Network Delay

```python
{
    "handler": "network_delay",
    "params": {
        "target_service": "ts-order-service",
        "target_namespace": "ts",
        "delay": "100ms",           # Required: delay amount
        "jitter": "10ms",           # Optional: random variation
        "correlation": "50",        # Optional: correlation (0-100)
        "direction": "to"           # Optional: "to", "from", "both"
    }
}
```

**Parameters**:
- `delay`: Delay duration (e.g., "50ms", "1s", "500ms")
- `jitter`: Random variation in delay
- `correlation`: Correlation between consecutive delays (0-100)
- `direction`: Traffic direction to affect

### Network Loss

```python
{
    "handler": "network_loss",
    "params": {
        "target_service": "ts-payment-service",
        "loss": "10",               # Required: loss percentage (0-100)
        "correlation": "25"         # Optional: correlation (0-100)
    }
}
```

### Network Partition

```python
{
    "handler": "network_partition",
    "params": {
        "target_service": "ts-user-service",
        "direction": "both",        # Optional: "to", "from", "both"
        "external_targets": [       # Optional: external IPs/domains
            "8.8.8.8",
            "example.com"
        ]
    }
}
```

### Network Duplicate

```python
{
    "handler": "network_duplicate",
    "params": {
        "target_service": "ts-order-service",
        "duplicate": "50",          # Required: duplication percentage
        "correlation": "25"         # Optional: correlation
    }
}
```

### Network Corrupt

```python
{
    "handler": "network_corrupt",
    "params": {
        "target_service": "ts-payment-service",
        "corrupt": "5",             # Required: corruption percentage
        "correlation": "10"         # Optional: correlation
    }
}
```

## Pod Fault Parameters

### Pod Kill

```python
{
    "handler": "pod_kill",
    "params": {
        "target_service": "ts-order-service",
        "grace_period": 5           # Optional: termination grace period (seconds)
    }
}
```

### Pod Failure

```python
{
    "handler": "pod_failure",
    "params": {
        "target_service": "ts-payment-service"
    }
}
```

### Container Kill

```python
{
    "handler": "container_kill",
    "params": {
        "target_service": "ts-user-service",
        "container_name": "main"    # Optional: specific container name
    }
}
```

## Stress Fault Parameters

### CPU Stress

```python
{
    "handler": "cpu_stress",
    "params": {
        "target_service": "ts-order-service",
        "workers": 2,               # Required: number of CPU workers
        "load": 80                  # Optional: CPU load per worker (0-100)
    }
}
```

**Parameters**:
- `workers`: Number of CPU stress workers (typically 1-4)
- `load`: CPU load percentage per worker (default: 100)

### Memory Stress

```python
{
    "handler": "memory_stress",
    "params": {
        "target_service": "ts-payment-service",
        "size": "512MB"             # Required: memory to consume
    }
}
```

**Parameters**:
- `size`: Memory size (e.g., "256MB", "1GB", "512MB")

## JVM Fault Parameters

### JVM Exception

```python
{
    "handler": "jvm_exception",
    "params": {
        "target_service": "ts-order-service",
        "class": "com.example.OrderService",        # Required: class name
        "method": "createOrder",                    # Required: method name
        "exception": "java.lang.RuntimeException"   # Required: exception class
    }
}
```

**Parameters**:
- `class`: Fully qualified class name
- `method`: Method name to inject exception
- `exception`: Exception class to throw

### JVM Latency

```python
{
    "handler": "jvm_latency",
    "params": {
        "target_service": "ts-payment-service",
        "class": "com.example.PaymentService",
        "method": "processPayment",
        "latency": 5000             # Required: delay in milliseconds
    }
}
```

**Parameters**:
- `latency`: Delay in milliseconds (e.g., 1000 = 1 second)

### JVM Return Value

```python
{
    "handler": "jvm_return",
    "params": {
        "target_service": "ts-user-service",
        "class": "com.example.UserService",
        "method": "getUserBalance",
        "value": "0"                # Required: return value to inject
    }
}
```

**Parameters**:
- `value`: Return value as string (will be converted to method's return type)

## HTTP Fault Parameters

### HTTP Abort

```python
{
    "handler": "http_abort",
    "params": {
        "target_service": "ts-order-service",
        "status_code": 503,         # Required: HTTP status code
        "path": "/api/orders",      # Optional: URL path pattern
        "percentage": 50            # Optional: percentage of requests (0-100)
    }
}
```

**Parameters**:
- `status_code`: HTTP status code to return (e.g., 500, 503, 404)
- `path`: URL path pattern to match (supports wildcards)
- `percentage`: Percentage of requests to abort (default: 100)

### HTTP Delay

```python
{
    "handler": "http_delay",
    "params": {
        "target_service": "ts-payment-service",
        "delay": "2s",              # Required: delay duration
        "path": "/api/payment",     # Optional: URL path pattern
        "percentage": 100           # Optional: percentage of requests
    }
}
```

### HTTP Replace

```python
{
    "handler": "http_replace",
    "params": {
        "target_service": "ts-user-service",
        "path": "/api/user",
        "replace_body": '{"error": "Service unavailable"}',  # Optional
        "replace_headers": {                                  # Optional
            "X-Custom-Header": "value"
        }
    }
}
```

## Advanced Usage

### Label Selectors

Target specific pods using label selectors:

```python
{
    "handler": "network_delay",
    "params": {
        "target_service": "ts-order-service",
        "label_selectors": {
            "version": "v2",
            "environment": "production"
        },
        "delay": "100ms"
    }
}
```

### Annotation Selectors

Target pods by annotations:

```python
{
    "handler": "pod_kill",
    "params": {
        "target_service": "ts-payment-service",
        "annotation_selectors": {
            "chaos.mesh.org/inject": "true"
        }
    }
}
```

### External Targets

For network faults affecting external communication:

```python
{
    "handler": "network_partition",
    "params": {
        "target_service": "ts-order-service",
        "external_targets": [
            "database.example.com",
            "10.0.0.5"
        ]
    }
}
```

### Port Filtering

Target specific ports:

```python
{
    "handler": "network_delay",
    "params": {
        "target_service": "ts-order-service",
        "port": 8080,               # Target specific port
        "delay": "100ms"
    }
}
```

## Validation Rules

### Required Fields

All HandlerNodes must have:
- `handler`: Valid fault type
- `params`: Object with required parameters for the fault type
- `params.target_service`: Target service name (for most fault types)

### Parameter Constraints

**Percentages**: Must be 0-100
```python
"loss": "10"        # Valid
"loss": "150"       # Invalid: exceeds 100
```

**Durations**: Must use valid time units
```python
"delay": "100ms"    # Valid
"delay": "1s"       # Valid
"delay": "100"      # Invalid: missing unit
```

**Memory Sizes**: Must use valid size units
```python
"size": "512MB"     # Valid
"size": "1GB"       # Valid
"size": "512"       # Invalid: missing unit
```

## Complete Examples

### Single Fault

```python
from rcabench.openapi.models import DtoSubmitInjectionReq

req = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[
        {
            "handler": "network_delay",
            "params": {
                "delay": "100ms",
                "target_service": "ts-order-service",
                "target_namespace": "ts",
                "jitter": "10ms"
            }
        }
    ],
    duration=60
)
```

### Multiple Faults

```python
req = DtoSubmitInjectionReq(
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
        },
        {
            "handler": "pod_kill",
            "params": {
                "target_service": "ts-user-service",
                "mode": "fixed",
                "value": "1"
            }
        }
    ],
    duration=120
)
```

### Complex Scenario

```python
req = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[
        {
            "handler": "network_delay",
            "params": {
                "delay": "200ms",
                "jitter": "50ms",
                "target_service": "ts-order-service",
                "direction": "to",
                "mode": "percent",
                "value": "50"
            }
        },
        {
            "handler": "http_abort",
            "params": {
                "target_service": "ts-payment-service",
                "status_code": 503,
                "path": "/api/payment/*",
                "percentage": 30
            }
        },
        {
            "handler": "memory_stress",
            "params": {
                "target_service": "ts-user-service",
                "size": "1GB",
                "mode": "one"
            }
        }
    ],
    duration=180,
    description="Complex multi-fault scenario"
)
```

## Error Messages

Common validation errors:

**Missing required parameter**:
```
Error: Missing required parameter 'delay' for handler 'network_delay'
```

**Invalid parameter value**:
```
Error: Invalid value '150' for parameter 'loss': must be 0-100
```

**Unknown handler**:
```
Error: Unknown handler type 'invalid_handler'
```

**Service not found**:
```
Error: Target service 'ts-invalid-service' not found in namespace 'ts'
```

## Best Practices

1. **Start simple**: Test single faults before combining
2. **Use realistic values**: Match production failure patterns
3. **Specify namespace**: Always include `target_namespace` for clarity
4. **Add descriptions**: Use the `description` field for documentation
5. **Validate locally**: Test HandlerNode structure before submission
6. **Monitor impact**: Start with low percentages and short durations

## Next Steps

- [Fault Types](fault-types): Detailed fault type reference
- [Using AegisLab](../using-aegislab): Submit injections via SDK
- [Workflow Overview](workflow-overview): End-to-end process
