---
title: Data Formats
weight: 2
---

Understanding the input data formats is essential for developing RCA algorithms. All data is provided in standardized parquet files with consistent schemas.

## Input Data Structure

When your algorithm is invoked, you receive an `AlgorithmArgs` object:

```python
from pathlib import Path

@dataclass(kw_only=True, frozen=True, slots=True)
class AlgorithmArgs:
    dataset: str           # Dataset name (e.g., "trainticket-pandora-v1")
    datapack: str          # Datapack identifier (e.g., "0", "1", "2")
    input_folder: Path     # Path to folder containing parquet files
    output_folder: Path    # Path to folder for writing results
```

The `input_folder` contains standardized parquet files:

```
input_folder/
├── abnormal_traces.parquet           # Distributed trace data during fault
├── normal_traces.parquet             # Baseline trace data before fault
├── abnormal_logs.parquet             # Application logs during fault (optional)
├── normal_logs.parquet               # Baseline logs before fault (optional)
├── abnormal_metrics.parquet          # Time-series metrics during fault (optional)
├── normal_metrics.parquet            # Baseline metrics before fault (optional)
├── abnormal_metrics_sum.parquet      # Sum metrics during fault (optional)
├── abnormal_metrics_histogram.parquet # Histogram metrics during fault (optional)
├── metrics_sli.parquet               # SLI metrics (optional)
├── injection.json                    # Ground truth fault injection info
└── conclusion.parquet                # Span-level performance comparison
```

**Note**: The dataset provides both abnormal (during fault) and normal (baseline) data for comparison. Most algorithms focus on the abnormal data files.

## Trace Data Schema

The `abnormal_traces.parquet` file contains distributed traces with the following schema:

| Column | Type | Description |
|--------|------|-------------|
| `time` | Datetime(ns, UTC) | Span start timestamp |
| `trace_id` | String | Unique identifier for the trace |
| `span_id` | String | Unique identifier for the span |
| `parent_span_id` | String | Parent span ID (empty string for root spans) |
| `span_name` | String | Operation/endpoint name |
| `attr.span_kind` | String | INTERNAL, SERVER, CLIENT, PRODUCER, CONSUMER |
| `service_name` | String | Name of the service that created the span |
| `duration` | UInt64 | Duration in nanoseconds |
| `attr.status_code` | String | Status: Ok, Error, Unset |
| `attr.k8s.pod.name` | String | Kubernetes pod name |
| `attr.k8s.service.name` | String | Kubernetes service name |
| `attr.k8s.namespace.name` | String | Kubernetes namespace |
| `attr.http.request.content_length` | UInt64 | HTTP request size in bytes |
| `attr.http.response.content_length` | UInt64 | HTTP response size in bytes |
| `attr.http.request.method` | String | HTTP method (GET, POST, etc.) |
| `attr.http.response.status_code` | UInt16 | HTTP status code (200, 500, etc.) |

**Sample data** (from rcabench_filtered dataset):

```
time: 2025-08-07 18:55:13.006381566 UTC
trace_id: 884e7b3d6eb50948a1d7702c9169e0...
span_id: eb33ccdeefb81189
parent_span_id: (empty - root span)
span_name: HTTP GET http://ts-ui-dashboard...
service_name: loadgenerator
duration: 6844377 (nanoseconds)
attr.status_code: Ok
```

### Example: Reading Traces

```python
import polars as pl

# Read trace data from input folder
traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

# Filter error spans
errors = traces.filter(pl.col("attr.status_code") == "Error")

# Calculate service error rates
error_rates = (
    traces
    .group_by("service_name")
    .agg([
        pl.col("attr.status_code").eq("Error").sum().alias("error_count"),
        pl.count().alias("total_count")
    ])
    .with_columns(
        (pl.col("error_count") / pl.col("total_count")).alias("error_rate")
    )
    .sort("error_rate", descending=True)
)
```

## Metrics Data Schema

The `abnormal_metrics.parquet` file contains time-series metrics:

| Column | Type | Description |
|--------|------|-------------|
| `time` | Datetime(ns, UTC) | Metric timestamp |
| `metric` | String | Name of the metric (e.g., "container.cpu.usage") |
| `value` | Float64 | Metric value |
| `service_name` | String | Service that emitted the metric |
| `attr.k8s.node.name` | String | Kubernetes node name |
| `attr.k8s.namespace.name` | String | Kubernetes namespace |
| `attr.k8s.statefulset.name` | String | StatefulSet name (if applicable) |
| `attr.k8s.deployment.name` | String | Deployment name (if applicable) |
| `attr.k8s.replicaset.name` | String | ReplicaSet name |
| `attr.k8s.pod.name` | String | Pod name |
| `attr.k8s.container.name` | String | Container name |
| `attr.destination_workload` | String | Destination workload for network metrics |
| `attr.source_workload` | String | Source workload for network metrics |
| `attr.destination` | String | Destination for network metrics |
| `attr.source` | String | Source for network metrics |

**Sample data** (from rcabench_filtered dataset):

```
time: 2025-08-07 18:55:13.019864982 UTC
metric: container.cpu.usage
value: 0.035307
service_name: ts-food-service
attr.k8s.pod.name: ts-food-service-7d9f8b5c4d-xyz
```

### Example: Reading Metrics

```python
# Read metrics from input folder
metrics_path = args.input_folder / "abnormal_metrics.parquet"
if metrics_path.exists():
    metrics = pl.read_parquet(metrics_path)

    # Get CPU usage by service
    cpu_usage = (
        metrics
        .filter(pl.col("metric") == "container.cpu.usage")
        .group_by("service_name")
        .agg(pl.mean("value").alias("avg_cpu"))
        .sort("avg_cpu", descending=True)
    )
```

## Logs Data Schema

The `abnormal_logs.parquet` file contains structured logs:

| Column | Type | Description |
|--------|------|-------------|
| `time` | Datetime(ns, UTC) | Log timestamp |
| `trace_id` | String | Associated trace ID (if available) |
| `span_id` | String | Associated span ID (if available) |
| `level` | String | DEBUG, INFO, WARN, ERROR, FATAL |
| `service_name` | String | Service that emitted the log |
| `message` | String | Log message |
| `attr.k8s.pod.name` | String | Kubernetes pod name |
| `attr.k8s.service.name` | String | Kubernetes service name |
| `attr.k8s.namespace.name` | String | Kubernetes namespace |
| `attr.template_id` | UInt16 | Log template identifier |
| `attr.log_template` | String | Log template pattern |

**Sample data** (from rcabench_filtered dataset):

```
time: 2025-08-07 18:56:27.893 UTC
trace_id: ebb8c911b293f1209764...
span_id: f23507c4af6f3980
level: INFO
service_name: ts-contacts-service
message: findContactsByAccountId Query ...
attr.log_template: findContactsByAccountId Query ...
```

### Example: Reading Logs

```python
# Read logs from input folder
log_path = args.input_folder / "abnormal_logs.parquet"
if log_path.exists():
    logs = pl.read_parquet(log_path)

    # Find error logs
    error_logs = (
        logs
        .filter(pl.col("level").is_in(["ERROR", "FATAL"]))
        .sort("time")
    )

    # Count errors by service
    error_counts = (
        error_logs
        .group_by("service_name")
        .agg(pl.count().alias("error_count"))
        .sort("error_count", descending=True)
    )
```

## Ground Truth Schema

### injection.json

The `injection.json` file contains fault injection metadata:

```json
{
  "benchmark": "clickhouse",
  "created_at": "2025-08-07T18:55:12.029Z",
  "description": "Fault for task 5cea6db4-96bb-4105-b2fb-9db4bb21056d",
  "start_time": "2025-08-07T18:55:13Z",
  "end_time": "2025-08-07T18:59:12Z",
  "fault_type": 28,
  "injection_name": "ts0-ts-auth-service-stress-jv8m9r",
  "ground_truth": {
    "service": ["ts-auth-service"],
    "pod": ["ts-auth-service-5dd97d5ccd-rvsh8"],
    "container": ["ts-auth-service"],
    "function": ["auth.entity.User.getPassword"],
    "metric": ["memory"],
    "span": null
  },
  "task_id": "5cea6db4-96bb-4105-b2fb-9db4bb21056d"
}
```

**Key fields:**
- `ground_truth.service`: List of root cause services (primary label for evaluation)
- `ground_truth.pod`: Specific pod(s) affected
- `ground_truth.function`: Method/function where fault was injected
- `ground_truth.metric`: Type of fault (memory, cpu, network, etc.)
- `start_time` / `end_time`: Fault injection time window
- `fault_type`: Numeric fault type identifier

### conclusion.parquet

The `conclusion.parquet` file contains span-level performance comparison between normal and abnormal periods:

| Column | Type | Description |
|--------|------|-------------|
| `SpanName` | String | Span/operation name |
| `Issues` | String | JSON-encoded issues detected |
| `AbnormalAvgDuration` | Float64 | Average duration during fault (seconds) |
| `NormalAvgDuration` | Float64 | Average duration before fault (seconds) |
| `AbnormalSuccRate` | Float64 | Success rate during fault |
| `NormalSuccRate` | Float64 | Success rate before fault |
| `AbnormalP90` | Float64 | 90th percentile latency during fault |
| `NormalP90` | Float64 | 90th percentile latency before fault |
| `AbnormalP95` | Float64 | 95th percentile latency during fault |
| `NormalP95` | Float64 | 95th percentile latency before fault |
| `AbnormalP99` | Float64 | 99th percentile latency during fault |
| `NormalP99` | Float64 | 99th percentile latency before fault |

**Sample data:**

```
SpanName: HTTP POST http://ts-ui-dashboard...
AbnormalAvgDuration: 0.461801
NormalAvgDuration: 0.698477
AbnormalSuccRate: 1.0
NormalSuccRate: 1.0
```

### Example: Reading Ground Truth

```python
import json
import polars as pl

# Read injection metadata
with open(args.input_folder / "injection.json") as f:
    injection = json.load(f)

# Get target service (ground truth root cause)
target_services = injection["ground_truth"]["service"]
print(f"Ground truth root cause: {target_services}")

# Get fault time window
fault_start = injection["start_time"]  # ISO 8601 string
fault_end = injection["end_time"]

# Read conclusion for span-level analysis
conclusion = pl.read_parquet(args.input_folder / "conclusion.parquet")

# Find spans with significant performance degradation
degraded_spans = conclusion.filter(
    pl.col("AbnormalAvgDuration") > pl.col("NormalAvgDuration") * 1.5
).sort("AbnormalAvgDuration", descending=True)
```

## Working with Attributes

Span and metric attributes are directly available as columns with `attr.` prefix:

```python
# Access HTTP status code directly
traces_with_http = traces.filter(
    pl.col("attr.http.response.status_code") >= 400
)

# Group by pod
pod_stats = traces.group_by("attr.k8s.pod.name").agg([
    pl.count().alias("span_count"),
    pl.mean("duration").alias("avg_duration")
])

# Filter by namespace
ns_traces = traces.filter(
    pl.col("attr.k8s.namespace.name") == "ts0"
)
```

## Lazy Evaluation for Large Datasets

For large datasets, use lazy evaluation to avoid loading everything into memory:

```python
# Scan instead of read
traces = pl.scan_parquet(args.input_folder / "abnormal_traces.parquet")

# Build query without executing
query = (
    traces
    .filter(pl.col("attr.status_code") == "Error")
    .group_by("service_name")
    .agg(pl.count().alias("error_count"))
    .sort("error_count", descending=True)
)

# Execute only when needed
result = query.collect()
```

## Handling Missing Data

Not all files are guaranteed to be present:

```python
# Check if metrics file exists
metrics_path = args.input_folder / "abnormal_metrics.parquet"
if metrics_path.exists():
    metrics = pl.read_parquet(metrics_path)
else:
    metrics = None

# Check if logs file exists
logs_path = args.input_folder / "abnormal_logs.parquet"
if logs_path.exists():
    logs = pl.read_parquet(logs_path)
else:
    logs = None

# Handle empty parent_span_id (root spans)
traces = traces.with_columns(
    pl.when(pl.col("parent_span_id") == "")
    .then(pl.lit("ROOT"))
    .otherwise(pl.col("parent_span_id"))
    .alias("parent_span_id")
)
```

## Time Range Filtering

Filter data to the fault injection window:

```python
import json
from datetime import datetime

# Get fault time range from injection metadata
with open(args.input_folder / "injection.json") as f:
    injection = json.load(f)

# Parse ISO 8601 timestamps
fault_start = datetime.fromisoformat(injection["start_time"].replace("Z", "+00:00"))
fault_end = datetime.fromisoformat(injection["end_time"].replace("Z", "+00:00"))

# Read traces
traces = pl.read_parquet(args.input_folder / "abnormal_traces.parquet")

# Filter traces to fault window (time column is already datetime)
fault_traces = traces.filter(
    (pl.col("time") >= fault_start) &
    (pl.col("time") <= fault_end)
)

# Alternative: The abnormal_traces.parquet already contains only fault window data
# So you typically don't need to filter by time
```

## Output Format

Your algorithm must return a list of `AlgorithmAnswer` objects:

```python
@dataclass(kw_only=True, frozen=True, slots=True)
class AlgorithmAnswer:
    level: str    # Level of the root cause (e.g., "service", "pod", "container")
    name: str     # Name of the suspected root cause
    rank: int     # Rank of this answer (1 = most likely root cause)
```

Example:

```python
def __call__(self, args: AlgorithmArgs) -> list[AlgorithmAnswer]:
    # Your algorithm logic

    return [
        AlgorithmAnswer(level="service", name="ts-order-service", rank=1),
        AlgorithmAnswer(level="service", name="ts-payment-service", rank=2),
        AlgorithmAnswer(level="service", name="ts-user-service", rank=3),
    ]
```

## Best Practices

1. **Use lazy evaluation** for large datasets to minimize memory usage
2. **Handle missing data** gracefully with null checks and defaults
3. **Filter to fault window** to focus on relevant time periods
4. **Parse JSON attributes** only when needed to avoid overhead
5. **Validate input data** before processing to catch schema issues early

## Next Steps

- [Algorithm Interface](algorithm-interface): Learn about the Algorithm base class
- [Containerization](containerization): Package your algorithm for deployment
- [Local Evaluation](../local-evaluation): Test your algorithm on real datasets
