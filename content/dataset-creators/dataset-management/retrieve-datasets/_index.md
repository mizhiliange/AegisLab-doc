---
title: Retrieve Datasets
weight: 1
---

Download and access datasets created through fault injection.

## Prerequisites

- Completed fault injection task
- Dataset ID or task ID
- AegisLab SDK installed or JuiceFS access

## Using the Python SDK

### List Available Datasets

```python
from rcabench.openapi import ApiClient, Configuration, DatasetApi

config = Configuration(host="${AEGISLAB_API_URL}")  # Default: http://10.10.10.220:8080
client = ApiClient(config)
dataset_api = DatasetApi(client)

# List all datasets
datasets = dataset_api.list_datasets()

for ds in datasets:
    print(f"{ds.name}: {ds.datapack_count} datapacks, {ds.size_bytes / 1024**3:.2f} GB")

# Filter by benchmark
trainticket_datasets = dataset_api.list_datasets(benchmark="trainticket")
```

### Get Dataset Information

```python
# Get detailed information
dataset = dataset_api.get_dataset(name="trainticket-custom-001")

print(f"Name: {dataset.name}")
print(f"Benchmark: {dataset.benchmark}")
print(f"Datapacks: {dataset.datapack_count}")
print(f"Size: {dataset.size_bytes / 1024**3:.2f} GB")
print(f"Created: {dataset.created_at}")
print(f"Description: {dataset.description}")
```

### Download Dataset

```python
# Download entire dataset
dataset_api.download_dataset(
    dataset_id="trainticket-custom-001",
    output_path="./data/"
)

# Download specific datapacks
dataset_api.download_datapacks(
    dataset_id="trainticket-custom-001",
    datapack_ids=[0, 1, 2, 3, 4],
    output_path="./data/"
)
```

### Get Dataset from Task

```python
from rcabench.openapi import TaskApi

task_api = TaskApi(client)

# Get task details
task = task_api.get_task(task_id="task-abc123")

if task.status == "completed":
    dataset_id = task.dataset_id
    print(f"Dataset ID: {dataset_id}")

    # Download dataset
    dataset_api.download_dataset(
        dataset_id=dataset_id,
        output_path="./data/"
    )
```

## Using JuiceFS

### Mount JuiceFS

```bash
# Mount shared storage (use your JUICEFS_REDIS_URL from .env)
sudo juicefs mount ${JUICEFS_REDIS_URL} /mnt/jfs -d
# Default: redis://10.10.10.119:6379/1

# Create symlink in your project
cd your-project/data
ln -s /mnt/jfs/rcabench-platform-v2 ./
```

### Access Datasets

```bash
# List datasets
ls /mnt/jfs/rcabench-platform-v2/

# Access specific dataset
ls /mnt/jfs/rcabench-platform-v2/trainticket-custom-001/

# Copy dataset locally (optional)
cp -r /mnt/jfs/rcabench-platform-v2/trainticket-custom-001 ./data/
```

### Direct Access

```python
import polars as pl

# Read directly from JuiceFS mount
traces = pl.read_parquet("/mnt/jfs/rcabench-platform-v2/trainticket-custom-001/0/trace.parquet")
```

## Dataset Structure

Downloaded datasets follow this structure:

```
trainticket-custom-001/
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
├── ...
└── metadata.json
```

### metadata.json

```json
{
  "dataset_name": "trainticket-custom-001",
  "benchmark": "trainticket",
  "datapack_count": 100,
  "created_at": "2026-01-18T10:30:00Z",
  "description": "Custom fault scenarios for TrainTicket",
  "fault_types": ["network_delay", "pod_failure", "memory_pressure"],
  "generation_method": "manual",
  "version": "1.0.0"
}
```

## Accessing Data

### Load Traces

```python
import polars as pl

# Load traces from specific datapack
traces = pl.read_parquet("data/trainticket-custom-001/0/trace.parquet")

print(f"Loaded {len(traces)} spans")
print(traces.head())
```

### Load Metrics

```python
# Load metrics
metrics = pl.read_parquet("data/trainticket-custom-001/0/metrics.parquet")

print(f"Loaded {len(metrics)} metric points")
print(metrics.head())
```

### Load Ground Truth

```python
# Load ground truth
gt = pl.read_parquet("data/trainticket-custom-001/0/ground_truth.parquet")

print(f"Root cause: {gt['root_cause_service'][0]}")
print(f"Fault type: {gt['fault_type'][0]}")
print(f"Injection time: {gt['injection_time'][0]}")
```

## Batch Download

### Download Multiple Datasets

```python
dataset_names = [
    "trainticket-custom-001",
    "trainticket-custom-002",
    "trainticket-custom-003"
]

for name in dataset_names:
    print(f"Downloading {name}...")
    dataset_api.download_dataset(
        dataset_id=name,
        output_path=f"./data/{name}/"
    )
    print(f"Downloaded {name}")
```

### Parallel Download

```python
from concurrent.futures import ThreadPoolExecutor

def download_dataset(name):
    dataset_api.download_dataset(
        dataset_id=name,
        output_path=f"./data/{name}/"
    )
    return name

# Download in parallel
with ThreadPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(download_dataset, name) for name in dataset_names]

    for future in futures:
        name = future.result()
        print(f"Completed: {name}")
```

## Incremental Updates

### Check for New Datapacks

```python
# Get current dataset info
dataset = dataset_api.get_dataset(name="trainticket-custom-001")
current_count = dataset.datapack_count

# Check local count
import os
local_count = len([d for d in os.listdir("data/trainticket-custom-001") if d.isdigit()])

if current_count > local_count:
    print(f"New datapacks available: {current_count - local_count}")

    # Download new datapacks
    new_ids = range(local_count, current_count)
    dataset_api.download_datapacks(
        dataset_id="trainticket-custom-001",
        datapack_ids=list(new_ids),
        output_path="./data/trainticket-custom-001/"
    )
```

## Storage Considerations

### Disk Space

Check available disk space before downloading:

```python
import shutil

# Check available space
total, used, free = shutil.disk_usage("./data")

print(f"Free space: {free / 1024**3:.2f} GB")

# Get dataset size
dataset = dataset_api.get_dataset(name="trainticket-custom-001")
dataset_size = dataset.size_bytes / 1024**3

if free < dataset_size * 1.1:  # 10% buffer
    print(f"WARNING: Insufficient disk space")
    print(f"Required: {dataset_size:.2f} GB")
    print(f"Available: {free / 1024**3:.2f} GB")
```

### Compression

Compress datasets for storage:

```bash
# Compress dataset
tar -czf trainticket-custom-001.tar.gz data/trainticket-custom-001/

# Decompress
tar -xzf trainticket-custom-001.tar.gz
```

## Troubleshooting

### Download Failed

```
Error: Failed to download dataset
```

**Solution**: Check network connection and retry:

```python
import time

max_retries = 3
for attempt in range(max_retries):
    try:
        dataset_api.download_dataset(
            dataset_id="trainticket-custom-001",
            output_path="./data/"
        )
        break
    except Exception as e:
        print(f"Attempt {attempt + 1} failed: {e}")
        if attempt < max_retries - 1:
            time.sleep(5)
```

### Dataset Not Found

```
Error: Dataset 'trainticket-custom-001' not found
```

**Solution**: Verify dataset name:

```python
# List all datasets
datasets = dataset_api.list_datasets()
print([ds.name for ds in datasets])
```

### Incomplete Download

```
Error: Missing files in dataset
```

**Solution**: Verify download and re-download if needed:

```python
import os

def verify_dataset(dataset_path, expected_count):
    """Verify dataset completeness."""
    for i in range(expected_count):
        datapack_path = f"{dataset_path}/{i}"
        required_files = ["trace.parquet", "metrics.parquet", "log.parquet", "ground_truth.parquet"]

        for file in required_files:
            file_path = f"{datapack_path}/{file}"
            if not os.path.exists(file_path):
                print(f"Missing: {file_path}")
                return False

    return True

# Verify
if not verify_dataset("data/trainticket-custom-001", 100):
    print("Dataset incomplete, re-downloading...")
    dataset_api.download_dataset(
        dataset_id="trainticket-custom-001",
        output_path="./data/"
    )
```

## Best Practices

1. **Use JuiceFS for large datasets**: Avoid downloading if you can access directly
2. **Download incrementally**: Only download new datapacks as needed
3. **Verify completeness**: Check all files are present after download
4. **Compress for archival**: Use tar.gz for long-term storage
5. **Monitor disk space**: Ensure sufficient space before downloading
6. **Use parallel downloads**: Speed up batch downloads with threading

## Next Steps

- [Dataset Structure](../dataset-structure): Understanding dataset organization
- [Quality Checks](../quality-checks): Validating dataset quality
- [Algorithm Developers](../../algorithm-developers): Using datasets for evaluation
