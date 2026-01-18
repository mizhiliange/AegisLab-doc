---
title: Workflow Overview
weight: 1
---

# Fault Injection Workflow Overview

Complete end-to-end workflow for creating datasets through fault injection.

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    1. Design Experiment                          │
│  - Choose fault types                                            │
│  - Select target services                                        │
│  - Define parameters                                             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    2. Submit via API                             │
│  - Create DtoSubmitInjectionReq                                  │
│  - Define handler_nodes                                          │
│  - Set duration and metadata                                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    3. Task Queuing                               │
│  - AegisLab validates request                                    │
│  - Task added to Redis queue                                     │
│  - Returns task_id                                               │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    4. Fault Execution                            │
│  - Consumer worker picks up task                                 │
│  - Generates Chaos Mesh CRDs                                     │
│  - Applies chaos to target pods                                  │
│  - Monitors execution                                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    5. Data Collection                            │
│  - OpenTelemetry captures traces                                 │
│  - Prometheus collects metrics                                   │
│  - Logs aggregated with context                                  │
│  - Ground truth metadata recorded                                │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    6. Data Processing                            │
│  - Convert to parquet format                                     │
│  - Add ground truth labels                                       │
│  - Validate completeness                                         │
│  - Store in dataset repository                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    7. Dataset Ready                              │
│  - Dataset available for download                                │
│  - Can be used for algorithm evaluation                          │
│  - Metadata includes fault details                               │
└─────────────────────────────────────────────────────────────────┘
```

## Phase 1: Design Experiment

### Choose Fault Types

Select appropriate fault types based on research goals:

```python
# Network faults for communication issues
network_faults = ["network_delay", "network_loss", "network_partition"]

# Pod faults for service failures
pod_faults = ["pod_kill", "pod_failure", "container_kill"]

# Stress faults for resource exhaustion
stress_faults = ["cpu_stress", "memory_stress"]

# JVM faults for application-level issues
jvm_faults = ["jvm_exception", "jvm_latency", "jvm_return"]

# HTTP faults for API-level problems
http_faults = ["http_abort", "http_delay", "http_replace"]
```

### Select Target Services

Identify critical services in the request path:

```python
# TrainTicket critical services
critical_services = [
    "ts-order-service",      # Order processing
    "ts-payment-service",    # Payment handling
    "ts-user-service",       # User authentication
    "ts-travel-service",     # Travel planning
    "ts-route-service"       # Route management
]
```

### Define Parameters

Configure fault-specific parameters:

```python
# Example: Network delay configuration
network_delay_config = {
    "handler": "network_delay",
    "params": {
        "delay": "100ms",           # Delay amount
        "jitter": "10ms",           # Random variation
        "target_service": "ts-order-service",
        "target_namespace": "ts"
    }
}
```

## Phase 2: Submit via API

### Create Injection Request

```python
from rcabench.openapi import FaultInjectionApi
from rcabench.openapi.models import DtoSubmitInjectionReq

# Create request
req = DtoSubmitInjectionReq(
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
    duration=60,  # Fault duration in seconds
    description="Network delay experiment"
)

# Submit
api = FaultInjectionApi(client)
response = api.submit_injection(req)
task_id = response.task_id
```

### Request Validation

AegisLab validates:
- Benchmark system exists
- Handler types are valid
- Required parameters present
- Target services exist
- Duration is reasonable

## Phase 3: Task Queuing

### Task Creation

```python
# AegisLab creates task record
task = {
    "id": "task-123",
    "status": "pending",
    "benchmark": "trainticket",
    "handler_nodes": [...],
    "duration": 60,
    "created_at": "2026-01-18T10:00:00Z"
}
```

### Queue Management

Tasks are added to Redis delayed queue:

```
Redis Queue:
┌─────────────────────────────────────┐
│ Priority Queue (sorted by timestamp)│
│                                     │
│ task-123 (pending)                  │
│ task-124 (pending)                  │
│ task-125 (running)                  │
└─────────────────────────────────────┘
```

## Phase 4: Fault Execution

### Consumer Processing

Consumer worker picks up task:

```go
// AegisLab consumer (simplified)
func ProcessFaultInjection(task *Task) error {
    // Generate Chaos Mesh CRD
    chaos := GenerateChaos(task.HandlerNodes)

    // Apply to Kubernetes
    err := ApplyChaos(chaos, task.Duration)

    // Monitor execution
    MonitorChaos(chaos.Name, task.Duration)

    // Cleanup
    DeleteChaos(chaos.Name)

    return nil
}
```

### Chaos Mesh Application

Generated CRD example:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-123
  namespace: ts
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - ts
    labelSelectors:
      app: ts-order-service
  delay:
    latency: "100ms"
    jitter: "10ms"
  duration: "60s"
```

## Phase 5: Data Collection

### Trace Collection

OpenTelemetry captures distributed traces:

```
Trace Flow:
ts-gateway → ts-order-service → ts-payment-service → database
    │              │                    │
    │              │ (DELAY INJECTED)   │
    │              │                    │
    └──────────────┴────────────────────┴─→ OTEL Collector
                                              │
                                              ▼
                                         ClickHouse
```

### Metrics Collection

Prometheus scrapes metrics:

```
Metrics:
- http_request_duration_seconds
- http_requests_total
- system_cpu_utilization
- system_memory_usage
- jvm_memory_used_bytes
```

### Log Collection

Logs with trace context:

```json
{
  "timestamp": "2026-01-18T10:01:23Z",
  "trace_id": "abc123",
  "span_id": "def456",
  "service": "ts-order-service",
  "level": "ERROR",
  "message": "Connection timeout"
}
```

## Phase 6: Data Processing

### Convert to Parquet

Raw data converted to standardized format:

```python
# Traces
traces_df = pl.DataFrame({
    "trace_id": [...],
    "span_id": [...],
    "service_name": [...],
    "status_code": [...],
    # ... other columns
})
traces_df.write_parquet("traces.parquet")

# Metrics
metrics_df = pl.DataFrame({
    "timestamp": [...],
    "service_name": [...],
    "metric_name": [...],
    "value": [...]
})
metrics_df.write_parquet("metrics.parquet")

# Ground truth
gt_df = pl.DataFrame({
    "fault_type": ["network_delay"],
    "target_service": ["ts-order-service"],
    "fault_start_time": [1705572000000000000],
    "fault_end_time": [1705572060000000000]
})
gt_df.write_parquet("ground_truth.parquet")
```

### Validation

Check data completeness:

```python
def validate_dataset(datapack_path):
    """Validate dataset completeness."""
    checks = {
        "traces_exist": os.path.exists(f"{datapack_path}/traces.parquet"),
        "gt_exist": os.path.exists(f"{datapack_path}/ground_truth.parquet"),
        "traces_not_empty": len(pl.read_parquet(f"{datapack_path}/traces.parquet")) > 0,
        "has_errors": len(pl.read_parquet(f"{datapack_path}/traces.parquet")
                          .filter(pl.col("status_code") == "ERROR")) > 0
    }
    return all(checks.values())
```

## Phase 7: Dataset Ready

### Dataset Structure

```
trainticket-experiment-v1/
├── 0/
│   ├── traces.parquet
│   ├── metrics.parquet
│   ├── logs.parquet
│   └── ground_truth.parquet
├── 1/
│   ├── traces.parquet
│   ├── metrics.parquet
│   ├── logs.parquet
│   └── ground_truth.parquet
└── metadata.json
```

### Metadata

```json
{
  "dataset_id": "trainticket-experiment-v1",
  "benchmark": "trainticket",
  "datapack_count": 100,
  "created_at": "2026-01-18T10:00:00Z",
  "fault_types": ["network_delay", "pod_kill", "cpu_stress"],
  "target_services": ["ts-order-service", "ts-payment-service"],
  "description": "Network and resource faults on critical services"
}
```

## Monitoring Progress

### Track Task Status

```python
from rcabench.openapi import TaskApi

task_api = TaskApi(client)

# Poll for status
while True:
    task = task_api.get_task(task_id=task_id)
    print(f"Status: {task.status}, Progress: {task.progress}%")

    if task.status in ["completed", "failed"]:
        break

    time.sleep(5)
```

### Stream Events

```python
# Real-time event stream
for event in task_api.stream_trace_events(task_id=task_id):
    print(f"[{event.timestamp}] {event.level}: {event.message}")
```

Example events:

```
[10:00:00] INFO: Task started
[10:00:01] INFO: Generating Chaos Mesh CRD
[10:00:02] INFO: Applying chaos to ts-order-service
[10:00:03] INFO: Chaos applied successfully
[10:00:05] INFO: Collecting traces...
[10:01:03] INFO: Chaos duration completed
[10:01:04] INFO: Cleaning up chaos resources
[10:01:05] INFO: Processing collected data
[10:01:10] INFO: Dataset created: datapack-123
[10:01:11] INFO: Task completed
```

## Error Handling

### Common Failures

**Chaos application failed**:
```
Error: Failed to apply chaos: target service not found
Solution: Verify service exists in target namespace
```

**No traces collected**:
```
Error: No traces found during fault window
Solution: Ensure OpenTelemetry is enabled and load is running
```

**Data validation failed**:
```
Error: Dataset validation failed: no error spans found
Solution: Check if fault actually caused errors
```

## Best Practices

1. **Start with single faults**: Test one fault type before combining
2. **Verify load is running**: Ensure traffic during injection
3. **Monitor system health**: Watch for cascading failures
4. **Validate data quality**: Check traces captured during fault window
5. **Use appropriate duration**: Long enough for data collection (60-120s typical)

## Next Steps

- [Fault Types](fault-types): Complete fault type reference
- [HandlerNode Format](handler-node-format): Detailed parameter reference
- [Using AegisLab](../using-aegislab): Submit injections programmatically
- [Monitoring & Collection](../monitoring-collection): Track data collection
