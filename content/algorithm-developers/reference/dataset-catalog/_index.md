---
title: Dataset Catalog
weight: 3
---

# Dataset Catalog

Available datasets for algorithm evaluation.

## TrainTicket Datasets

### trainticket-pandora-v1

**Description**: Genetic algorithm-generated fault scenarios using Pandora scheduler.

**Characteristics:**
- **Datapacks**: 100
- **Size**: 15.2 GB
- **Benchmark**: TrainTicket (40+ microservices)
- **Fault Types**: Network delay, pod failure, memory pressure, CPU stress, JVM exceptions
- **Generation Method**: Genetic algorithm with SLO violation and diagnosis difficulty objectives

**Fault Distribution:**
- Network faults: 35%
- Resource faults: 30%
- Pod failures: 20%
- JVM faults: 15%

**Use Cases:**
- Baseline evaluation
- Algorithm comparison
- Genetic algorithm research

**Example:**
```bash
./main.py eval single simplerca trainticket-pandora-v1 0
```

### trainticket-random-v1

**Description**: Randomly generated fault scenarios for baseline comparison.

**Characteristics:**
- **Datapacks**: 200
- **Size**: 28.5 GB
- **Benchmark**: TrainTicket
- **Fault Types**: All supported fault types with uniform distribution
- **Generation Method**: Random sampling

**Use Cases:**
- Baseline comparison with Pandora
- Robustness testing
- Coverage analysis

## OTEL Demo Datasets

### otel-demo-v1

**Description**: Fault scenarios on OpenTelemetry Demo application.

**Characteristics:**
- **Datapacks**: 50
- **Size**: 8.3 GB
- **Benchmark**: OTEL Demo (11 microservices)
- **Fault Types**: Network delay, pod failure, resource pressure
- **Generation Method**: Manual curation

**Services:**
- Frontend, Cart, Checkout, Currency, Email, Payment, Product Catalog, Recommendation, Shipping, Ad, Feature Flag

**Use Cases:**
- Cross-benchmark evaluation
- Smaller-scale testing
- OTEL instrumentation research

## Media Microservices Datasets

### media-ms-v1

**Description**: Fault scenarios on DeathStarBench Media Microservices.

**Characteristics:**
- **Datapacks**: 75
- **Size**: 12.1 GB
- **Benchmark**: Media Microservices (13 services)
- **Fault Types**: Network delay, pod failure, database issues
- **Generation Method**: Targeted scenario design

**Use Cases:**
- Media processing workloads
- Database-heavy scenarios
- Complex dependency chains

## Dataset Structure

All datasets follow the same structure:

```
dataset-name/
├── 0/
│   ├── trace.parquet
│   ├── metrics.parquet
│   ├── log.parquet
│   └── ground_truth.parquet
├── 1/
│   ├── trace.parquet
│   ├── metrics.parquet
│   ├── log.parquet
│   └── ground_truth.parquet
└── ...
```

### File Formats

**trace.parquet**
- Distributed traces with spans
- Columns: trace_id, span_id, parent_span_id, service_name, operation_name, start_time, end_time, status_code, attributes

**metrics.parquet**
- Time-series metrics
- Columns: timestamp, service_name, metric_name, value, labels

**log.parquet**
- Structured logs
- Columns: timestamp, service_name, level, message, attributes

**ground_truth.parquet**
- Fault injection metadata
- Columns: root_cause_service, fault_type, fault_params, injection_time, duration

## Dataset Statistics

### Complexity Metrics

| Dataset | Avg Services | Avg Spans/Trace | Avg Trace Depth | Fault Diversity |
|---------|--------------|-----------------|-----------------|-----------------|
| trainticket-pandora-v1 | 42 | 156 | 8 | High |
| trainticket-random-v1 | 42 | 148 | 8 | Medium |
| otel-demo-v1 | 11 | 45 | 5 | Medium |
| media-ms-v1 | 13 | 67 | 6 | High |

### Difficulty Metrics

| Dataset | Avg MRR (SimpleRCA) | Diagnosis Difficulty | SLO Violation Rate |
|---------|---------------------|----------------------|-------------------|
| trainticket-pandora-v1 | 0.823 | High | 87% |
| trainticket-random-v1 | 0.891 | Medium | 65% |
| otel-demo-v1 | 0.856 | Medium | 72% |
| media-ms-v1 | 0.798 | High | 81% |

## Accessing Datasets

### Via JuiceFS (Recommended)

```bash
# Mount JuiceFS (use your JUICEFS_REDIS_URL from .env)
sudo juicefs mount ${JUICEFS_REDIS_URL} /mnt/jfs -d
# Default: redis://10.10.10.119:6379/1

# Create symlink
cd rcabench-platform/data
ln -s /mnt/jfs/rcabench-platform-v2 ./
```

### Via Download

```bash
# Download specific dataset
./main.py download-dataset trainticket-pandora-v1 --output ./data/
```

## Dataset Selection Guide

### For Algorithm Development

**Start with**: `trainticket-pandora-v1` (0-9)
- Small subset for quick iteration
- Representative fault scenarios
- Good baseline performance

**Progress to**: `trainticket-pandora-v1` (0-99)
- Full evaluation
- Comprehensive coverage
- Publication-ready results

### For Cross-Benchmark Evaluation

**Use**: `otel-demo-v1` and `media-ms-v1`
- Different system architectures
- Varying complexity levels
- Generalization testing

### For Robustness Testing

**Use**: `trainticket-random-v1`
- Uniform fault distribution
- Edge case coverage
- Stress testing

## Creating Custom Datasets

See [Dataset Creators Guide](../../dataset-creators) for creating your own datasets through fault injection.

## Dataset Versioning

Datasets follow semantic versioning:
- `v1.0.0`: Initial release
- `v1.1.0`: Additional datapacks or fault types
- `v2.0.0`: Breaking changes to schema or structure

## Citation

If you use these datasets in research, please cite:

```bibtex
@inproceedings{aegislab2025,
  title={AegisLab: A Comprehensive Fault Injection and RCA Evaluation Platform},
  author={...},
  booktitle={...},
  year={2025}
}
```

## See Also

- [Data Formats](../../development-guide/data-formats): Detailed schema documentation
- [Local Evaluation](../../local-evaluation): Using datasets for evaluation
- [Dataset Creators](../../dataset-creators): Creating new datasets
