---
title: Data Formats
weight: 2
---

# Data Formats

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
├── trace.parquet          # Distributed trace data
├── log.parquet            # Application logs (optional)
├── metrics.parquet        # Time-series metrics (optional)
├── metrics_sli.parquet    # SLI metrics (optional)
├── injection.json         # Ground truth fault injection info
└── conclusion.json        # Expected root causes (labels)
```

## Trace Data Schema

The `trace.parquet` file contains distributed traces with the following schema:

| Column | Type | Description |
|--------|------|-------------|
| `trace_id` | String | Unique identifier for the trace |
| `span_id` | String | Unique identifier for the span |
| `parent_span_id` | String | Parent span ID (null for root spans) |
| `service_name` | String | Name of the service that created the span |
| `operation_name` | String | Operation/endpoint name |
| `start_time` | Int64 | Start timestamp (nanoseconds) |
| `end_time` | Int64 | End timestamp (nanoseconds) |
| `duration` | Int64 | Duration in nanoseconds |
| `status_code` | String | Status: OK, ERROR, UNSET |
| `status_message` | String | Error message if status is ERROR |
| `span_kind` | String | INTERNAL, SERVER, CLIENT, PRODUCER, CONSUMER |
| `attributes` | String | JSON-encoded span attributes |

### Example: Reading Traces

```python
import polars as pl

# Read trace data from input folder
traces = pl.read_parquet(args.input_folder / "trace.parquet")

# Filter error spans
errors = traces.filter(pl.col("status_code") == "ERROR")

# Calculate service error rates
error_rates = (
    traces
    .group_by("service_name")
    .agg([
        pl.col("status_code").eq("ERROR").sum().alias("error_count"),
        pl.count().alias("total_count")
    ])
    .with_columns(
        (pl.col("error_count") / pl.col("total_count")).alias("error_rate")
    )
    .sort("error_rate", descending=True)
)
```

## Metrics Data Schema

The `metrics.parquet` file contains time-series metrics:

| Column | Type | Description |
|--------|------|-------------|
| `timestamp` | Int64 | Metric timestamp (nanoseconds) |
| `service_name` | String | Service that emitted the metric |
| `metric_name` | String | Name of the metric |
| `metric_type` | String | GAUGE, COUNTER, HISTOGRAM |
| `value` | Float64 | Metric value |
| `labels` | String | JSON-encoded labels/tags |

### Example: Reading Metrics

```python
# Read metrics from input folder
metrics_path = args.input_folder / "metrics.parquet"
if metrics_path.exists():
    metrics = pl.read_parquet(metrics_path)

    # Get CPU usage by service
    cpu_usage = (
        metrics
        .filter(pl.col("metric_name") == "system.cpu.utilization")
        .group_by("service_name")
        .agg(pl.mean("value").alias("avg_cpu"))
        .sort("avg_cpu", descending=True)
    )
```

## Logs Data Schema

The `log.parquet` file contains structured logs:

| Column | Type | Description |
|--------|------|-------------|
| `timestamp` | Int64 | Log timestamp (nanoseconds) |
| `trace_id` | String | Associated trace ID (if available) |
| `span_id` | String | Associated span ID (if available) |
| `service_name` | String | Service that emitted the log |
| `severity` | String | DEBUG, INFO, WARN, ERROR, FATAL |
| `message` | String | Log message |
| `attributes` | String | JSON-encoded log attributes |

### Example: Reading Logs

```python
# Read logs from input folder
log_path = args.input_folder / "log.parquet"
if log_path.exists():
    logs = pl.read_parquet(log_path)

    # Find error logs
    error_logs = (
        logs
        .filter(pl.col("severity").is_in(["ERROR", "FATAL"]))
        .sort("timestamp")
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

The `injection.json` file contains fault injection metadata:

```json
{
  "fault_type": "network_delay",
  "target_service": "ts-order-service",
  "target_operation": "/api/orders",
  "start_time": 1234567890000000000,
  "end_time": 1234567950000000000,
  "parameters": {
    "delay": "100ms",
    "jitter": "10ms"
  }
}
```

### Example: Reading Ground Truth

```python
import json

# Read injection metadata
with open(args.input_folder / "injection.json") as f:
    injection = json.load(f)

# Get target service
target_service = injection["target_service"]
print(f"Ground truth root cause: {target_service}")

# Get fault time window
fault_start = injection["start_time"]
fault_end = injection["end_time"]
```

## Working with JSON Attributes

Many columns store JSON-encoded data. Use Polars' JSON parsing:

```python
# Parse span attributes
traces_with_attrs = traces.with_columns(
    pl.col("attributes").str.json_decode().alias("attrs_parsed")
)

# Extract specific attribute
http_status = traces_with_attrs.with_columns(
    pl.col("attrs_parsed").struct.field("http.status_code").alias("http_status")
)
```

## Lazy Evaluation for Large Datasets

For large datasets, use lazy evaluation to avoid loading everything into memory:

```python
# Scan instead of read
traces = pl.scan_parquet(args.input_folder / "trace.parquet")

# Build query without executing
query = (
    traces
    .filter(pl.col("status_code") == "ERROR")
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
metrics_path = args.input_folder / "metrics.parquet"
if metrics_path.exists():
    metrics = pl.read_parquet(metrics_path)
else:
    metrics = None

# Handle null values in traces
traces = traces.with_columns(
    pl.col("parent_span_id").fill_null("ROOT")
)
```

## Time Range Filtering

Filter data to the fault injection window:

```python
import json

# Get fault time range from injection metadata
with open(args.input_folder / "injection.json") as f:
    injection = json.load(f)

fault_start = injection["start_time"]
fault_end = injection["end_time"]

# Filter traces to fault window
traces = pl.read_parquet(args.input_folder / "trace.parquet")
fault_traces = traces.filter(
    (pl.col("start_time") >= fault_start) &
    (pl.col("start_time") <= fault_end)
)
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
