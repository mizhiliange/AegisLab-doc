---
title: Dataset Structure
weight: 2
---

# Dataset Structure

Understanding the organization and format of collected datasets.

## Directory Structure

Each dataset follows a standardized structure:

```
dataset-name/
├── metadata.json
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
├── 2/
│   └── ...
└── N/
    └── ...
```

## Metadata File

### metadata.json

Contains dataset-level information:

```json
{
  "dataset_name": "trainticket-custom-001",
  "benchmark": "trainticket",
  "datapack_count": 100,
  "created_at": "2026-01-18T10:30:00Z",
  "updated_at": "2026-01-18T15:45:00Z",
  "description": "Custom fault scenarios for TrainTicket",
  "version": "1.0.0",
  "generation_method": "manual",
  "fault_types": [
    "network_delay",
    "pod_failure",
    "memory_pressure",
    "cpu_stress"
  ],
  "statistics": {
    "total_traces": 1523400,
    "total_spans": 1523400,
    "total_metrics": 845600,
    "total_logs": 1234500,
    "avg_traces_per_datapack": 15234,
    "avg_spans_per_datapack": 15234
  },
  "schema_version": "2.0"
}
```

## Datapack Structure

Each datapack (numbered directory) represents one fault injection scenario.

### Naming Convention

- Datapacks are numbered sequentially: `0`, `1`, `2`, ..., `N`
- Each datapack is independent and self-contained
- Datapack IDs are used for evaluation and retrieval

### Required Files

Each datapack must contain four parquet files:

1. **traces.parquet**: Distributed traces
2. **metrics.parquet**: Time-series metrics
3. **logs.parquet**: Application logs
4. **ground_truth.parquet**: Fault injection metadata

## File Schemas

### traces.parquet

Distributed tracing data following OpenTelemetry format:

| Column | Type | Description |
|--------|------|-------------|
| trace_id | String | Unique trace identifier |
| span_id | String | Unique span identifier |
| parent_span_id | String | Parent span ID (null for root) |
| service_name | String | Service that created the span |
| operation_name | String | Operation being performed |
| start_time | Int64 | Start timestamp (microseconds) |
| end_time | Int64 | End timestamp (microseconds) |
| status_code | String | Status (OK, ERROR, UNSET) |
| attributes | Map | Key-value metadata |

**Example:**

```python
import polars as pl

traces = pl.read_parquet("dataset/0/traces.parquet")
print(traces.head())

# Output:
# ┌──────────┬──────────┬────────────────┬─────────────────┬───────────────┬────────────────┬──────────────┬─────────────┬────────────┐
# │ trace_id │ span_id  │ parent_span_id │ service_name    │ operation_name│ start_time     │ end_time     │ status_code │ attributes │
# ├──────────┼──────────┼────────────────┼─────────────────┼───────────────┼────────────────┼──────────────┼─────────────┼────────────┤
# │ abc123   │ span-1   │ null           │ ts-gateway      │ GET /api/v1   │ 1705584000000  │ 1705584000200│ OK          │ {...}      │
# │ abc123   │ span-2   │ span-1         │ ts-order-service│ POST /orders  │ 1705584000050  │ 1705584000180│ ERROR       │ {...}      │
# └──────────┴──────────┴────────────────┴─────────────────┴───────────────┴────────────────┴──────────────┴─────────────┴────────────┘
```

### metrics.parquet

Time-series metrics data:

| Column | Type | Description |
|--------|------|-------------|
| timestamp | Int64 | Metric timestamp (seconds) |
| service_name | String | Service name |
| metric_name | String | Metric identifier |
| value | Float64 | Metric value |
| labels | Map | Additional labels |

**Example:**

```python
metrics = pl.read_parquet("dataset/0/metrics.parquet")
print(metrics.head())

# Output:
# ┌────────────┬─────────────────┬──────────────────────────┬────────┬────────────┐
# │ timestamp  │ service_name    │ metric_name              │ value  │ labels     │
# ├────────────┼─────────────────┼──────────────────────────┼────────┼────────────┤
# │ 1705584000 │ ts-order-service│ cpu_usage_percent        │ 45.2   │ {...}      │
# │ 1705584000 │ ts-order-service│ memory_usage_mb          │ 512.8  │ {...}      │
# │ 1705584000 │ ts-order-service│ http_request_duration_ms │ 150.5  │ {...}      │
# └────────────┴─────────────────┴──────────────────────────┴────────┴────────────┘
```

### logs.parquet

Structured application logs:

| Column | Type | Description |
|--------|------|-------------|
| timestamp | String | Log timestamp (ISO 8601) |
| service_name | String | Service name |
| level | String | Log level (ERROR, WARNING, INFO, DEBUG) |
| message | String | Log message |
| attributes | Map | Additional context |

**Example:**

```python
logs = pl.read_parquet("dataset/0/logs.parquet")
print(logs.head())

# Output:
# ┌─────────────────────────┬─────────────────┬─────────┬──────────────────────────┬────────────┐
# │ timestamp               │ service_name    │ level   │ message                  │ attributes │
# ├─────────────────────────┼─────────────────┼─────────┼──────────────────────────┼────────────┤
# │ 2026-01-18T10:32:15.123Z│ ts-order-service│ ERROR   │ Database connection fail │ {...}      │
# │ 2026-01-18T10:32:15.456Z│ ts-order-service│ WARNING │ Retry attempt 1          │ {...}      │
# └─────────────────────────┴─────────────────┴─────────┴──────────────────────────┴────────────┘
```

### ground_truth.parquet

Fault injection metadata and ground truth:

| Column | Type | Description |
|--------|------|-------------|
| root_cause_service | String | Service where fault was injected |
| fault_type | String | Type of fault (network_delay, pod_failure, etc.) |
| fault_params | Map | Fault parameters |
| injection_time | Int64 | Fault injection timestamp (microseconds) |
| duration | Int64 | Fault duration (seconds) |
| description | String | Fault description |

**Example:**

```python
gt = pl.read_parquet("dataset/0/ground_truth.parquet")
print(gt)

# Output:
# ┌────────────────────┬──────────────┬────────────────┬────────────────┬──────────┬─────────────────────┐
# │ root_cause_service │ fault_type   │ fault_params   │ injection_time │ duration │ description         │
# ├────────────────────┼──────────────┼────────────────┼────────────────┼──────────┼─────────────────────┤
# │ ts-order-service   │ network_delay│ {delay: 100ms} │ 1705584030000  │ 60       │ Network delay 100ms │
# └────────────────────┴──────────────┴────────────────┴────────────────┴──────────┴─────────────────────┘
```

## File Sizes

Typical file sizes per datapack:

| File | Typical Size | Range |
|------|--------------|-------|
| traces.parquet | 150 KB | 50 KB - 500 KB |
| metrics.parquet | 80 KB | 30 KB - 200 KB |
| logs.parquet | 120 KB | 40 KB - 300 KB |
| ground_truth.parquet | 1 KB | 0.5 KB - 2 KB |
| **Total per datapack** | **~350 KB** | **120 KB - 1 MB** |

For a 100-datapack dataset: ~35 MB total

## Schema Versioning

Datasets use semantic versioning for schema changes:

### Version 1.0 (Legacy)

- Basic trace, metric, log schemas
- Limited attribute support
- No standardized ground truth format

### Version 2.0 (Current)

- OpenTelemetry-compliant trace schema
- Standardized metric labels
- Structured log attributes
- Comprehensive ground truth metadata
- Parquet compression enabled

### Version 3.0 (Planned)

- Enhanced trace context propagation
- Additional metric types
- Log correlation with traces
- Multi-fault ground truth support

## Accessing Schema Information

### Check Schema Version

```python
import json

with open("dataset/metadata.json") as f:
    metadata = json.load(f)

print(f"Schema version: {metadata['schema_version']}")
```

### Inspect Parquet Schema

```python
import polars as pl

traces = pl.read_parquet("dataset/0/traces.parquet")
print(traces.schema)

# Output:
# {'trace_id': String, 'span_id': String, 'parent_span_id': String, ...}
```

### Validate Schema

```python
def validate_schema(datapack_path):
    """Validate datapack schema."""

    required_files = ["traces.parquet", "metrics.parquet", "logs.parquet", "ground_truth.parquet"]

    for file in required_files:
        file_path = f"{datapack_path}/{file}"

        if not os.path.exists(file_path):
            print(f"ERROR: Missing {file}")
            return False

        # Check schema
        df = pl.read_parquet(file_path)

        if file == "traces.parquet":
            required_cols = ["trace_id", "span_id", "service_name", "start_time", "end_time", "status_code"]
            missing = [col for col in required_cols if col not in df.columns]
            if missing:
                print(f"ERROR: Missing columns in traces: {missing}")
                return False

    return True

# Validate
if validate_schema("dataset/0"):
    print("Schema valid")
```

## Best Practices

1. **Preserve structure**: Don't modify the directory structure
2. **Maintain schema**: Ensure all required columns are present
3. **Use parquet**: Don't convert to other formats (CSV, JSON)
4. **Keep metadata**: Always include metadata.json
5. **Validate data**: Check schema before using datasets
6. **Document changes**: Update metadata when modifying datasets

## See Also

- [Retrieve Datasets](../retrieve-datasets): Downloading datasets
- [Quality Checks](../quality-checks): Validating dataset quality
- [Data Formats](../../algorithm-developers/development-guide/data-formats): Detailed schema documentation
