---
title: Quality Checks
weight: 3
---

# Quality Checks

Validate dataset quality before using for algorithm evaluation.

## Overview

Quality checks ensure datasets are:
- Complete (all required files present)
- Consistent (data aligns across files)
- Accurate (fault injection had visible impact)
- Valid (schema matches specification)

## Automated Validation

### Using the SDK

```python
from rcabench.openapi import DatasetApi

dataset_api = DatasetApi(client)

# Validate dataset
validation_result = dataset_api.validate_dataset(
    dataset_id="trainticket-custom-001"
)

if validation_result.is_valid:
    print("Dataset is valid")
else:
    print(f"Validation errors: {validation_result.errors}")
```

### Using the CLI

```bash
cd rcabench-platform
./main.py validate-dataset trainticket-custom-001
```

## Manual Validation

### Check Completeness

Verify all datapacks have required files:

```python
import os
import polars as pl

def check_completeness(dataset_path, expected_count):
    """Check if all datapacks are complete."""

    missing_files = []
    required_files = ["trace.parquet", "metrics.parquet", "log.parquet", "ground_truth.parquet"]

    for i in range(expected_count):
        datapack_path = f"{dataset_path}/{i}"

        if not os.path.exists(datapack_path):
            missing_files.append(f"Datapack {i} missing entirely")
            continue

        for file in required_files:
            file_path = f"{datapack_path}/{file}"
            if not os.path.exists(file_path):
                missing_files.append(f"{file_path}")

    return missing_files

# Check
missing = check_completeness("data/trainticket-custom-001", 100)
if missing:
    print(f"Missing files: {len(missing)}")
    for f in missing[:10]:  # Show first 10
        print(f"  - {f}")
else:
    print("All files present")
```

### Check Schema

Validate parquet schemas:

```python
def check_schema(datapack_path):
    """Validate schema for all parquet files."""

    errors = []

    # Check traces schema
    traces = pl.read_parquet(f"{datapack_path}/trace.parquet")
    required_trace_cols = ["trace_id", "span_id", "service_name", "start_time", "end_time", "status_code"]
    missing = [col for col in required_trace_cols if col not in traces.columns]
    if missing:
        errors.append(f"Traces missing columns: {missing}")

    # Check metrics schema
    metrics = pl.read_parquet(f"{datapack_path}/metrics.parquet")
    required_metric_cols = ["timestamp", "service_name", "metric_name", "value"]
    missing = [col for col in required_metric_cols if col not in metrics.columns]
    if missing:
        errors.append(f"Metrics missing columns: {missing}")

    # Check logs schema
    logs = pl.read_parquet(f"{datapack_path}/log.parquet")
    required_log_cols = ["timestamp", "service_name", "level", "message"]
    missing = [col for col in required_log_cols if col not in logs.columns]
    if missing:
        errors.append(f"Logs missing columns: {missing}")

    # Check ground truth schema
    gt = pl.read_parquet(f"{datapack_path}/ground_truth.parquet")
    required_gt_cols = ["root_cause_service", "fault_type", "injection_time"]
    missing = [col for col in required_gt_cols if col not in gt.columns]
    if missing:
        errors.append(f"Ground truth missing columns: {missing}")

    return errors

# Validate
errors = check_schema("data/trainticket-custom-001/0")
if errors:
    print("Schema errors:")
    for e in errors:
        print(f"  - {e}")
else:
    print("Schema valid")
```

### Check Data Quality

Validate data quality metrics:

```python
def check_data_quality(datapack_path):
    """Check data quality metrics."""

    issues = []

    # Load data
    traces = pl.read_parquet(f"{datapack_path}/trace.parquet")
    gt = pl.read_parquet(f"{datapack_path}/ground_truth.parquet")

    # Check trace count
    if len(traces) < 100:
        issues.append(f"Low trace count: {len(traces)} (expected >100)")

    # Check for nulls
    null_counts = traces.null_count()
    for col in null_counts.columns:
        count = null_counts[col][0]
        if count > 0 and col not in ["parent_span_id"]:  # parent_span_id can be null for root spans
            issues.append(f"Nulls in {col}: {count}")

    # Check for errors
    error_spans = traces.filter(pl.col("status_code") == "ERROR")
    if len(error_spans) == 0:
        issues.append("No error spans found")

    # Check fault impact
    fault_service = gt["root_cause_service"][0]
    injection_time = gt["injection_time"][0]

    fault_spans = traces.filter(
        (pl.col("service_name") == fault_service) &
        (pl.col("start_time") >= injection_time)
    )

    if len(fault_spans) > 0:
        error_rate = len(fault_spans.filter(pl.col("status_code") == "ERROR")) / len(fault_spans)
        if error_rate < 0.05:
            issues.append(f"Low error rate in fault service: {error_rate:.2%}")
    else:
        issues.append(f"No spans from fault service after injection")

    # Check timestamp validity
    invalid_times = traces.filter(pl.col("end_time") < pl.col("start_time"))
    if len(invalid_times) > 0:
        issues.append(f"Invalid timestamps: {len(invalid_times)} spans")

    return issues

# Check quality
issues = check_data_quality("data/trainticket-custom-001/0")
if issues:
    print("Quality issues:")
    for i in issues:
        print(f"  - {i}")
else:
    print("Data quality good")
```

## Batch Validation

Validate multiple datapacks:

```python
def validate_dataset(dataset_path, datapack_count):
    """Validate entire dataset."""

    results = {
        "total": datapack_count,
        "valid": 0,
        "invalid": 0,
        "errors": []
    }

    for i in range(datapack_count):
        datapack_path = f"{dataset_path}/{i}"

        # Check completeness
        missing = check_completeness(dataset_path, 1)
        if missing:
            results["invalid"] += 1
            results["errors"].append(f"Datapack {i}: Missing files")
            continue

        # Check schema
        schema_errors = check_schema(datapack_path)
        if schema_errors:
            results["invalid"] += 1
            results["errors"].append(f"Datapack {i}: Schema errors")
            continue

        # Check quality
        quality_issues = check_data_quality(datapack_path)
        if quality_issues:
            results["invalid"] += 1
            results["errors"].append(f"Datapack {i}: Quality issues")
            continue

        results["valid"] += 1

    return results

# Validate
results = validate_dataset("data/trainticket-custom-001", 100)
print(f"Valid: {results['valid']}/{results['total']}")
print(f"Invalid: {results['invalid']}/{results['total']}")

if results["errors"]:
    print("\nErrors (first 10):")
    for e in results["errors"][:10]:
        print(f"  - {e}")
```

## Statistical Validation

### Distribution Analysis

Check if data distributions are reasonable:

```python
def analyze_distributions(dataset_path, datapack_count):
    """Analyze data distributions across datapacks."""

    trace_counts = []
    error_rates = []
    service_counts = []

    for i in range(datapack_count):
        traces = pl.read_parquet(f"{dataset_path}/{i}/trace.parquet")

        trace_counts.append(len(traces))

        error_spans = traces.filter(pl.col("status_code") == "ERROR")
        error_rates.append(len(error_spans) / len(traces) if len(traces) > 0 else 0)

        service_counts.append(traces["service_name"].n_unique())

    import numpy as np

    print("Trace count statistics:")
    print(f"  Mean: {np.mean(trace_counts):.0f}")
    print(f"  Std: {np.std(trace_counts):.0f}")
    print(f"  Min: {np.min(trace_counts)}")
    print(f"  Max: {np.max(trace_counts)}")

    print("\nError rate statistics:")
    print(f"  Mean: {np.mean(error_rates):.2%}")
    print(f"  Std: {np.std(error_rates):.2%}")
    print(f"  Min: {np.min(error_rates):.2%}")
    print(f"  Max: {np.max(error_rates):.2%}")

    print("\nService count statistics:")
    print(f"  Mean: {np.mean(service_counts):.0f}")
    print(f"  Std: {np.std(service_counts):.0f}")

# Analyze
analyze_distributions("data/trainticket-custom-001", 100)
```

### Outlier Detection

Identify anomalous datapacks:

```python
def detect_outliers(dataset_path, datapack_count):
    """Detect outlier datapacks."""

    trace_counts = []

    for i in range(datapack_count):
        traces = pl.read_parquet(f"{dataset_path}/{i}/trace.parquet")
        trace_counts.append((i, len(traces)))

    import numpy as np

    counts = [c for _, c in trace_counts]
    mean = np.mean(counts)
    std = np.std(counts)

    outliers = []
    for i, count in trace_counts:
        z_score = abs((count - mean) / std)
        if z_score > 3:  # More than 3 standard deviations
            outliers.append((i, count, z_score))

    if outliers:
        print(f"Found {len(outliers)} outliers:")
        for i, count, z_score in outliers:
            print(f"  Datapack {i}: {count} traces (z-score: {z_score:.2f})")
    else:
        print("No outliers detected")

# Detect
detect_outliers("data/trainticket-custom-001", 100)
```

## Fault Impact Validation

Verify fault injection had measurable impact:

```python
def validate_fault_impact(datapack_path):
    """Validate fault injection impact."""

    traces = pl.read_parquet(f"{datapack_path}/trace.parquet")
    gt = pl.read_parquet(f"{datapack_path}/ground_truth.parquet")

    fault_service = gt["root_cause_service"][0]
    injection_time = gt["injection_time"][0]
    duration = gt["duration"][0]

    # Pre-injection period
    pre_spans = traces.filter(
        (pl.col("service_name") == fault_service) &
        (pl.col("start_time") < injection_time)
    )

    # During injection period
    during_spans = traces.filter(
        (pl.col("service_name") == fault_service) &
        (pl.col("start_time") >= injection_time) &
        (pl.col("start_time") < injection_time + duration * 1000000)
    )

    # Calculate error rates
    pre_error_rate = len(pre_spans.filter(pl.col("status_code") == "ERROR")) / len(pre_spans) if len(pre_spans) > 0 else 0
    during_error_rate = len(during_spans.filter(pl.col("status_code") == "ERROR")) / len(during_spans) if len(during_spans) > 0 else 0

    print(f"Fault service: {fault_service}")
    print(f"Pre-injection error rate: {pre_error_rate:.2%}")
    print(f"During injection error rate: {during_error_rate:.2%}")
    print(f"Impact: {(during_error_rate - pre_error_rate):.2%}")

    if during_error_rate < pre_error_rate + 0.1:
        print("WARNING: Low fault impact")
        return False

    return True

# Validate
if validate_fault_impact("data/trainticket-custom-001/0"):
    print("Fault impact validated")
```

## Generating Quality Reports

Create comprehensive quality report:

```python
def generate_quality_report(dataset_path, datapack_count):
    """Generate comprehensive quality report."""

    report = {
        "dataset": dataset_path,
        "total_datapacks": datapack_count,
        "valid_datapacks": 0,
        "completeness_issues": 0,
        "schema_issues": 0,
        "quality_issues": 0,
        "fault_impact_issues": 0
    }

    for i in range(datapack_count):
        datapack_path = f"{dataset_path}/{i}"

        # Check completeness
        if not os.path.exists(datapack_path):
            report["completeness_issues"] += 1
            continue

        # Check schema
        schema_errors = check_schema(datapack_path)
        if schema_errors:
            report["schema_issues"] += 1
            continue

        # Check quality
        quality_issues = check_data_quality(datapack_path)
        if quality_issues:
            report["quality_issues"] += 1
            continue

        # Check fault impact
        if not validate_fault_impact(datapack_path):
            report["fault_impact_issues"] += 1
            continue

        report["valid_datapacks"] += 1

    # Print report
    print("=" * 50)
    print("DATASET QUALITY REPORT")
    print("=" * 50)
    print(f"Dataset: {report['dataset']}")
    print(f"Total datapacks: {report['total_datapacks']}")
    print(f"Valid datapacks: {report['valid_datapacks']} ({report['valid_datapacks']/report['total_datapacks']*100:.1f}%)")
    print(f"\nIssues:")
    print(f"  Completeness: {report['completeness_issues']}")
    print(f"  Schema: {report['schema_issues']}")
    print(f"  Quality: {report['quality_issues']}")
    print(f"  Fault impact: {report['fault_impact_issues']}")
    print("=" * 50)

    return report

# Generate report
report = generate_quality_report("data/trainticket-custom-001", 100)
```

## Best Practices

1. **Validate early**: Check quality immediately after collection
2. **Automate checks**: Use scripts for consistent validation
3. **Document issues**: Keep track of problematic datapacks
4. **Fix or remove**: Either fix issues or exclude bad datapacks
5. **Re-validate**: Check quality after any modifications
6. **Version datasets**: Track quality metrics across versions

## Common Issues

### Low Trace Count

**Issue**: Datapack has very few traces.

**Causes**:
- Collection window too short
- Load generation stopped early
- Network issues during collection

**Solution**: Re-run fault injection with longer collection window.

### No Error Spans

**Issue**: No error spans in traces despite fault injection.

**Causes**:
- Fault not applied correctly
- Fault parameters too weak
- Wrong target service

**Solution**: Verify fault was applied and increase severity.

### Schema Mismatch

**Issue**: Missing or incorrect columns.

**Causes**:
- Schema version mismatch
- Data conversion errors
- Incomplete data collection

**Solution**: Re-collect data with correct schema version.

## See Also

- [Dataset Structure](../dataset-structure): Understanding dataset format
- [Retrieve Datasets](../retrieve-datasets): Downloading datasets
- [Sharing Datasets](../sharing-datasets): Making datasets available
