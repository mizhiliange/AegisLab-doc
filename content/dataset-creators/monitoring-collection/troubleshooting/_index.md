---
title: Troubleshooting
weight: 3
---

# Troubleshooting Collection Issues

Common problems during fault injection and data collection.

## No Traces Collected

### Symptom

```
Status: completed
Traces collected: 0
Spans collected: 0
```

### Possible Causes

1. **OpenTelemetry not configured**
2. **Collector not receiving traces**
3. **Wrong time window**
4. **Services not instrumented**

### Solutions

**Check OTEL Collector:**

```bash
# Check collector status
kubectl get pods -n observability | grep otel-collector

# View collector logs
kubectl logs -n observability otel-collector-0

# Check if traces are being received
kubectl logs -n observability otel-collector-0 | grep "traces received"
```

**Verify Service Instrumentation:**

```bash
# Check if services have OTEL env vars
kubectl get deployment -n ts ts-order-service -o yaml | grep OTEL

# Should see:
# - name: OTEL_EXPORTER_OTLP_ENDPOINT
#   value: http://otel-collector:4317
```

**Check ClickHouse:**

```bash
# Verify traces in ClickHouse
clickhouse-client --query "SELECT count() FROM traces WHERE timestamp > now() - 300"

# If zero, traces aren't reaching ClickHouse
```

**Increase Collection Window:**

```python
request = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[...],
    duration=60,
    collection_window=180  # Collect for 3 minutes
)
```

## Fault Not Applied

### Symptom

```
Status: completed
Error rate: 0%
No visible impact on system
```

### Possible Causes

1. **Target service doesn't exist**
2. **Chaos Mesh not installed**
3. **Wrong namespace**
4. **Fault parameters incorrect**

### Solutions

**Verify Target Service:**

```bash
# Check if service exists
kubectl get pods -n ts | grep ts-order-service

# Check service is running
kubectl get pods -n ts ts-order-service-0 -o wide
```

**Check Chaos Mesh:**

```bash
# Verify Chaos Mesh is installed
kubectl get pods -n chaos-mesh

# Check if chaos experiment was created
kubectl get networkchaos -n ts

# View chaos experiment details
kubectl describe networkchaos -n ts <chaos-name>
```

**Verify Namespace:**

```python
# Ensure correct namespace
request = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[
        {
            "handler": "network_delay",
            "params": {
                "delay": "100ms",
                "target_service": "ts-order-service",
                "target_namespace": "ts"  # Correct namespace
            }
        }
    ]
)
```

## Incomplete Data Collection

### Symptom

```
Status: completed
Traces collected: 234 (expected ~15000)
Metrics collected: 1200 (expected ~8000)
```

### Possible Causes

1. **Collection window too short**
2. **Load generation stopped early**
3. **Storage issues**
4. **Network problems**

### Solutions

**Increase Collection Window:**

```python
request = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[...],
    duration=60,
    collection_window=120,  # Double the injection duration
    pre_collection_window=30  # Collect baseline data
)
```

**Check Load Generator:**

```bash
# Verify load generator is running
kubectl get pods -n loadgen

# Check load generator logs
kubectl logs -n loadgen loadgen-0

# Verify traffic is reaching services
kubectl logs -n ts ts-gateway-0 | grep "HTTP"
```

**Check Storage:**

```bash
# Check ClickHouse disk space
kubectl exec -n observability clickhouse-0 -- df -h

# Check if ClickHouse is accepting writes
kubectl exec -n observability clickhouse-0 -- clickhouse-client --query "SELECT 1"
```

## Task Stuck in Pending

### Symptom

```
Status: pending
Progress: 0%
Duration: 10 minutes
```

### Possible Causes

1. **High queue backlog**
2. **Insufficient cluster resources**
3. **Load generator unavailable**

### Solutions

**Check Queue Status:**

```bash
# Check Redis queue
redis-cli -h aegislab-redis LLEN fault_injection_queue

# If queue is long, wait or increase workers
```

**Check Cluster Resources:**

```bash
# Check node resources
kubectl top nodes

# Check pod resource requests
kubectl describe nodes | grep -A 5 "Allocated resources"
```

**Check Load Generator:**

```bash
# Verify load generator deployment
kubectl get deployment -n loadgen

# Scale up if needed
kubectl scale deployment -n loadgen loadgen --replicas=3
```

## Task Failed During Injection

### Symptom

```
Status: failed
Phase: injecting
Error: Failed to apply chaos: pod not found
```

### Solutions

**Check Error Details:**

```python
task = task_api.get_task(task_id="task-abc123")
print(f"Error: {task.error_message}")
print(f"Phase: {task.current_phase}")

# Get detailed logs
logs = task_api.get_logs(task_id="task-abc123")
print(logs.stderr)
```

**Verify Chaos Mesh CRD:**

```bash
# Check if CRD exists
kubectl get crd | grep chaos-mesh

# Check chaos experiment status
kubectl get networkchaos -n ts -o yaml
```

**Check Permissions:**

```bash
# Verify service account has permissions
kubectl auth can-i create networkchaos --as=system:serviceaccount:aegislab:aegislab-worker -n ts
```

## Low Error Rate

### Symptom

```
Status: completed
Error rate: 2% (expected >50%)
Fault seems ineffective
```

### Possible Causes

1. **Fault parameters too weak**
2. **Wrong target service**
3. **Retry logic masking errors**
4. **Circuit breaker activated**

### Solutions

**Increase Fault Severity:**

```python
# Increase delay
handler_nodes=[
    {
        "handler": "network_delay",
        "params": {
            "delay": "500ms",  # Increase from 100ms
            "jitter": "100ms"
        }
    }
]

# Or use pod failure instead
handler_nodes=[
    {
        "handler": "pod_failure",
        "params": {
            "target_service": "ts-order-service"
        }
    }
]
```

**Verify Target Service:**

```python
# Check if target service is critical path
# Use dependency analysis to find bottleneck services
```

**Disable Retry Logic:**

```bash
# Temporarily disable retries in target service
kubectl set env deployment/ts-order-service -n ts MAX_RETRIES=0
```

## Data Quality Issues

### Symptom

```
Status: completed
Data collected but quality is poor:
- Missing timestamps
- Duplicate spans
- Invalid status codes
```

### Solutions

**Validate Data Schema:**

```python
import polars as pl

traces = pl.read_parquet("dataset/0/traces.parquet")

# Check for nulls
null_counts = traces.null_count()
print(null_counts)

# Check for duplicates
duplicates = traces.group_by("span_id").agg(
    pl.count().alias("count")
).filter(pl.col("count") > 1)

if len(duplicates) > 0:
    print(f"WARNING: {len(duplicates)} duplicate span IDs")
```

**Fix Data Issues:**

```python
# Remove duplicates
traces = traces.unique(subset=["span_id"])

# Fill missing timestamps
traces = traces.with_columns(
    pl.col("start_time").fill_null(strategy="forward")
)

# Validate status codes
valid_codes = ["OK", "ERROR", "UNSET"]
traces = traces.filter(pl.col("status_code").is_in(valid_codes))
```

## Connection Timeout

### Symptom

```
Error: Connection timeout while streaming events
```

### Solutions

**Increase Timeout:**

```python
config = Configuration(
    host="http://10.10.10.220:8080",
    timeout=60  # Increase from default 30s
)
```

**Use Polling Instead:**

```python
# Instead of streaming
# for event in task_api.stream_trace_events(task_id):
#     print(event)

# Use polling
while True:
    task = task_api.get_task(task_id="task-abc123")
    print(f"Status: {task.status}")

    if task.status in ["completed", "failed"]:
        break

    time.sleep(10)
```

## Getting Help

If your issue isn't resolved:

1. **Check logs**: Review task logs and Kubernetes pod logs
2. **Verify configuration**: Double-check all parameters
3. **Test manually**: Try applying chaos manually with kubectl
4. **Contact support**: Open an issue with full error details

## Reporting Issues

When reporting issues, include:

1. **Task ID**: For tracking
2. **Error message**: Full error text
3. **Configuration**: Injection request parameters
4. **Logs**: Task logs and relevant pod logs
5. **Environment**: Cluster version, Chaos Mesh version

Example issue report:

```
**Task ID:** task-abc123

**Error:** No traces collected

**Configuration:**
```python
DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[{"handler": "network_delay", ...}],
    duration=60
)
```

**Logs:**
```
[10:30:15] INFO: Task started
[10:30:25] INFO: Fault injection active
[10:31:25] INFO: Collecting traces
[10:31:26] ERROR: No traces found in ClickHouse
```

**Environment:**
- Kubernetes: v1.28
- Chaos Mesh: v2.6
- OTEL Collector: v0.88
```

## See Also

- [Track Progress](../track-progress): Monitoring guide
- [Observability Data](../observability-data): Understanding collected data
- [Fault Injection Guide](../../fault-injection-guide): Fault types and parameters
