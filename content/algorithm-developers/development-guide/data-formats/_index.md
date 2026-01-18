---
title: Data Formats
weight: 2
---

# Data Formats

Understanding the input data formats is essential for developing RCA algorithms. All data is provided in standardized parquet files with consistent schemas.

## Input Data Structure

When your algorithm is invoked, you receive an `AlgorithmArgs` object with paths to data files:

```python
@dataclass
class AlgorithmArgs:
    trace_path: str          # Path to traces.parquet
    metric_path: str         # Path to metrics.parquet (optional)
    log_path: str           # Path to logs.parquet (optional)
    ground_truth_path: str  # Path to ground_truth.parquet
    output_path: str        # Where to write results
    # Additional metadata fields
```

## Trace Data Schema

The `traces.parquet` file contains distributed traces with the following schema:

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

# Read trace data
traces = pl.read_parquet(args.trace_path)

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
# Read metrics
metrics = pl.read_parquet(args.metric_path)

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

The `logs.parquet` file contains structured logs:

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
# Read logs
logs = pl.read_parquet(args.log_path)

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

The `ground_truth.parquet` file contains fault injection metadata:

| Column | Type | Description |
|--------|------|-------------|
| `fault_type` | String | Type of fault injected |
| `target_service` | String | Service where fault was injected |
| `target_operation` | String | Specific operation affected (optional) |
| `fault_start_time` | Int64 | When fault injection started |
| `fault_end_time` | Int64 | When fault injection ended |
| `parameters` | String | JSON-encoded fault parameters |

### Example: Reading Ground Truth

```python
# Read ground truth
ground_truth = pl.read_parquet(args.ground_truth_path)

# Get target service
target_service = ground_truth.select("target_service").item()
print(f"Ground truth root cause: {target_service}")
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
traces = pl.scan_parquet(args.trace_path)

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

Not all fields are guaranteed to be present:

```python
# Check if metrics file exists
if args.metric_path and os.path.exists(args.metric_path):
    metrics = pl.read_parquet(args.metric_path)
else:
    metrics = None

# Handle null values
traces = traces.with_columns(
    pl.col("parent_span_id").fill_null("ROOT")
)
```

## Time Range Filtering

Filter data to the fault injection window:

```python
# Get fault time range from ground truth
gt = pl.read_parquet(args.ground_truth_path)
fault_start = gt.select("fault_start_time").item()
fault_end = gt.select("fault_end_time").item()

# Filter traces to fault window
fault_traces = traces.filter(
    (pl.col("start_time") >= fault_start) &
    (pl.col("start_time") <= fault_end)
)
```

## Output Format

Your algorithm must return an `AlgorithmAnswer` with ranked predictions:

```python
@dataclass
class AlgorithmAnswer:
    ranked_services: List[str]  # Ordered list of suspected root causes
    scores: Optional[List[float]] = None  # Optional confidence scores
    metadata: Optional[Dict] = None  # Optional additional info
```

Example:

```python
return AlgorithmAnswer(
    ranked_services=["ts-order-service", "ts-payment-service", "ts-user-service"],
    scores=[0.95, 0.72, 0.43],
    metadata={"algorithm_version": "1.0", "processing_time_ms": 123}
)
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
