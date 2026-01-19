---
title: Observability Data
weight: 2
---

# Observability Data

Understanding the traces, metrics, and logs collected during fault injection.

## Data Types

AegisLab collects three types of observability data:

1. **Traces**: Distributed traces with spans showing request flow
2. **Metrics**: Time-series metrics (CPU, memory, latency, throughput)
3. **Logs**: Structured application logs

## Distributed Traces

### Trace Structure

Each trace represents a single request through the system:

```
Trace ID: abc123
├── Span 1: gateway (200ms)
│   ├── Span 2: order-service (150ms)
│   │   ├── Span 3: payment-service (80ms)
│   │   └── Span 4: user-service (50ms)
│   └── Span 5: notification-service (30ms)
```

### Span Attributes

Each span contains:
- `trace_id`: Unique trace identifier
- `span_id`: Unique span identifier
- `parent_span_id`: Parent span ID (null for root)
- `service_name`: Service that created the span
- `operation_name`: Operation being performed
- `start_time`: Span start timestamp (microseconds)
- `end_time`: Span end timestamp (microseconds)
- `status_code`: Status (OK, ERROR, UNSET)
- `attributes`: Key-value metadata

### Example Span Data

```json
{
  "trace_id": "abc123",
  "span_id": "span-456",
  "parent_span_id": "span-123",
  "service_name": "ts-order-service",
  "operation_name": "POST /api/v1/orders",
  "start_time": 1705584000000000,
  "end_time": 1705584000150000,
  "status_code": "ERROR",
  "attributes": {
    "http.method": "POST",
    "http.status_code": 500,
    "error.message": "Database connection timeout"
  }
}
```

### Trace Collection Window

Traces are collected during and after fault injection:

```
Timeline:
|-------|-------|-------|-------|
0s     30s     60s     90s    120s
       ↑       ↑       ↑
    Inject  Active  Collect
```

- **Pre-injection (0-30s)**: Baseline traces
- **Injection (30-90s)**: Fault active, traces show impact
- **Post-injection (90-120s)**: Recovery traces

## Metrics

### Metric Types

**System Metrics:**
- CPU usage (%)
- Memory usage (MB)
- Disk I/O (MB/s)
- Network I/O (MB/s)

**Application Metrics:**
- Request rate (req/s)
- Error rate (%)
- Latency (ms): p50, p95, p99
- Throughput (req/s)

**Service-Specific Metrics:**
- Database connections
- Queue depth
- Cache hit rate
- Thread pool usage

### Metric Format

```json
{
  "timestamp": 1705584000,
  "service_name": "ts-order-service",
  "metric_name": "http_request_duration_ms",
  "value": 150.5,
  "labels": {
    "method": "POST",
    "endpoint": "/api/v1/orders",
    "status": "500"
  }
}
```

### Time-Series Data

Metrics are collected at regular intervals (default: 15s):

```
CPU Usage (ts-order-service):
Time    Value
10:30   45%
10:31   48%
10:32   92%  ← Fault injected
10:33   95%
10:34   50%  ← Fault removed
10:35   46%
```

## Logs

### Log Levels

- **ERROR**: Application errors
- **WARNING**: Warning conditions
- **INFO**: Informational messages
- **DEBUG**: Debug information

### Log Structure

```json
{
  "timestamp": "2026-01-18T10:32:15.123Z",
  "service_name": "ts-order-service",
  "level": "ERROR",
  "message": "Failed to process order",
  "attributes": {
    "order_id": "12345",
    "error_code": "DB_TIMEOUT",
    "stack_trace": "..."
  }
}
```

### Log Correlation

Logs are correlated with traces using trace context:

```json
{
  "timestamp": "2026-01-18T10:32:15.123Z",
  "service_name": "ts-order-service",
  "level": "ERROR",
  "message": "Database timeout",
  "trace_id": "abc123",
  "span_id": "span-456"
}
```

## Data Quality Checks

### Completeness

Check if all expected data is collected:

```python
from rcabench.openapi import TaskApi

task = task_api.get_task(task_id="task-abc123")

# Check trace completeness
print(f"Traces collected: {task.traces_collected}")
print(f"Expected traces: ~{task.expected_traces}")

# Check metric completeness
print(f"Metrics collected: {task.metrics_collected}")
print(f"Metric series: {task.metric_series_count}")

# Check log completeness
print(f"Logs collected: {task.logs_collected}")
print(f"Error logs: {task.error_logs}")
```

### Consistency

Verify data consistency:

```python
import polars as pl

# Load collected data
traces = pl.read_parquet("dataset/0/trace.parquet")
metrics = pl.read_parquet("dataset/0/metrics.parquet")
logs = pl.read_parquet("dataset/0/log.parquet")

# Check time ranges
trace_start = traces["start_time"].min()
trace_end = traces["end_time"].max()
metric_start = metrics["timestamp"].min()
metric_end = metrics["timestamp"].max()

print(f"Trace window: {trace_start} - {trace_end}")
print(f"Metric window: {metric_start} - {metric_end}")

# Check for gaps
time_gaps = traces.group_by_dynamic("start_time", every="1s").agg(
    pl.count().alias("count")
).filter(pl.col("count") == 0)

if len(time_gaps) > 0:
    print(f"WARNING: {len(time_gaps)} time gaps in traces")
```

### Accuracy

Validate fault injection impact:

```python
# Load ground truth
gt = pl.read_parquet("dataset/0/ground_truth.parquet")
fault_service = gt["root_cause_service"][0]
injection_time = gt["injection_time"][0]

# Check if fault service shows errors
fault_spans = traces.filter(
    (pl.col("service_name") == fault_service) &
    (pl.col("start_time") >= injection_time)
)

error_spans = fault_spans.filter(pl.col("status_code") == "ERROR")
error_rate = len(error_spans) / len(fault_spans)

print(f"Error rate in {fault_service}: {error_rate:.2%}")

if error_rate < 0.1:
    print("WARNING: Low error rate, fault may not have been effective")
```

## Data Storage

### Parquet Format

All data is stored in Apache Parquet format:

**Benefits:**
- Columnar storage (efficient queries)
- Compression (smaller file sizes)
- Schema enforcement (data validation)
- Fast read performance

**File Structure:**
```
dataset-name/
└── 0/
    ├── trace.parquet      (15 MB)
    ├── metrics.parquet     (8 MB)
    ├── log.parquet        (12 MB)
    └── ground_truth.parquet (1 KB)
```

### Schema Validation

Validate parquet schema:

```python
import polars as pl

# Check traces schema
traces = pl.read_parquet("dataset/0/trace.parquet")
expected_columns = [
    "trace_id", "span_id", "parent_span_id",
    "service_name", "operation_name",
    "start_time", "end_time", "status_code"
]

missing = [col for col in expected_columns if col not in traces.columns]
if missing:
    print(f"ERROR: Missing columns: {missing}")
```

## Accessing Data

### Via Python SDK

```python
from rcabench.openapi import DatasetApi

dataset_api = DatasetApi(client)

# Download dataset
dataset_api.download_dataset(
    dataset_id="trainticket-custom-001",
    output_path="./data/"
)

# Load data
import polars as pl
traces = pl.read_parquet("./data/trainticket-custom-001/0/trace.parquet")
```

### Via JuiceFS

```bash
# Mount JuiceFS (use your JUICEFS_REDIS_URL from .env)
sudo juicefs mount ${JUICEFS_REDIS_URL} /mnt/jfs -d
# Default: redis://10.10.10.119:6379/1

# Access data
ls /mnt/jfs/rcabench-platform-v2/trainticket-custom-001/0/
```

## Data Analysis Examples

### Error Analysis

```python
# Find services with errors
error_services = traces.filter(
    pl.col("status_code") == "ERROR"
).group_by("service_name").agg(
    pl.count().alias("error_count")
).sort("error_count", descending=True)

print(error_services)
```

### Latency Analysis

```python
# Calculate latency per service
latencies = traces.with_columns(
    (pl.col("end_time") - pl.col("start_time")).alias("latency")
).group_by("service_name").agg([
    pl.col("latency").mean().alias("avg_latency"),
    pl.col("latency").quantile(0.95).alias("p95_latency"),
    pl.col("latency").max().alias("max_latency")
])

print(latencies)
```

### Dependency Analysis

```python
# Build service dependency graph
dependencies = traces.filter(
    pl.col("parent_span_id").is_not_null()
).join(
    traces.select(["span_id", "service_name"]).rename({"service_name": "parent_service"}),
    left_on="parent_span_id",
    right_on="span_id"
).select(["parent_service", "service_name"]).unique()

print(dependencies)
```

## Best Practices

1. **Verify completeness**: Check all data types are collected
2. **Validate consistency**: Ensure time ranges align
3. **Check accuracy**: Verify fault impact is visible
4. **Monitor quality**: Track collection metrics
5. **Store efficiently**: Use parquet format
6. **Document metadata**: Include ground truth and fault parameters

## Next Steps

- [Troubleshooting](../troubleshooting): Debug collection issues
- [Dataset Management](../../dataset-management): Retrieve and manage datasets
- [Data Formats](../../algorithm-developers/development-guide/data-formats): Detailed schema documentation
