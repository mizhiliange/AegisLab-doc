---
title: Submit Execution
weight: 2
---

Create and submit evaluation tasks for remote execution on AegisLab infrastructure.

## Prerequisites

- Algorithm uploaded to Harbor registry
- AegisLab SDK installed
- API credentials configured
- Dataset available on the platform

## Using the Python SDK

### Basic Submission

```python
from rcabench.openapi import ApiClient, Configuration
from rcabench.openapi.api import ExecutionsApi
from rcabench.openapi.models import RunAlgorithmReq, ContainerSpec

# Configure API client (use your AEGISLAB_API_URL from .env)
config = Configuration(host="${AEGISLAB_API_URL}")  # Default: http://10.10.10.220:32080
client = ApiClient(config)
executions_api = ExecutionsApi(client)

# Submit execution
request = RunAlgorithmReq(
    algorithm=ContainerSpec(name="my-rca", version="v1.0.0"),
    dataset_name="rcabench_filtered",
    datapack_names=["ts0-ts-auth-service-stress-jv8m9r"],
    project_name="my-project"
)

response = executions_api.run_algorithm(request)
print(f"Execution submitted!")
for item in response.data:
    print(f"  Execution ID: {item.id}")
```

### Batch Submission

Submit multiple evaluations at once:

```python
datasets = ["rcabench_filtered", "rcabench_qps"]
executions = []

for dataset in datasets:
    # Get available datapacks for this dataset
    datapacks_response = datasets_api.list_dataset_versions(dataset_id=dataset)
    datapack_names = [dp.name for dp in datapacks_response.data.items[:10]]  # First 10

    request = RunAlgorithmReq(
        algorithm=ContainerSpec(name="my-rca", version="v1.0.0"),
        dataset_name=dataset,
        datapack_names=datapack_names,
        project_name="my-project"
    )
    response = executions_api.run_algorithm(request)
    executions.extend(response.data)

print(f"Submitted {len(executions)} executions")
```

### Advanced Options

```python
request = DtoSubmitExecutionReq(
    algorithm_name="my-rca",
    algorithm_version="v1.0.0",
    dataset="trainticket-pandora-v1",
    datapack_start=0,
    datapack_end=99,

    # Resource limits
    cpu_limit="2000m",
    memory_limit="4Gi",

    # Timeout
    timeout_seconds=3600,

    # Retry policy
    max_retries=3,

    # Priority
    priority="high",

    # Metadata
    description="Evaluation with optimized parameters",
    tags=["experiment-1", "baseline"]
)

response = api.submit_execution(request)
```

## Using the CLI

The rcabench-platform CLI provides a convenient wrapper:

```bash
cd rcabench-platform

# Submit single evaluation
./main.py submit-execution my-rca trainticket-pandora-v1 0 99

# With version
./main.py submit-execution my-rca:v1.0.0 trainticket-pandora-v1 0 99

# With options
./main.py submit-execution my-rca trainticket-pandora-v1 0 99 \
    --cpu 2000m \
    --memory 4Gi \
    --timeout 3600 \
    --description "Baseline evaluation"
```

## Request Parameters

### Required Parameters

- `algorithm_name`: Name of your algorithm (must match Harbor image name)
- `dataset`: Dataset identifier (e.g., `trainticket-pandora-v1`)
- `datapack_start`: Starting datapack ID (inclusive)
- `datapack_end`: Ending datapack ID (inclusive)

### Optional Parameters

- `algorithm_version`: Image tag (default: `latest`)
- `cpu_limit`: CPU limit (e.g., `1000m`, `2`)
- `memory_limit`: Memory limit (e.g., `2Gi`, `4096Mi`)
- `timeout_seconds`: Execution timeout (default: 1800)
- `max_retries`: Retry attempts on failure (default: 0)
- `priority`: Task priority (`low`, `normal`, `high`)
- `description`: Human-readable description
- `tags`: List of tags for organization

## Resource Allocation

### CPU Limits

- `500m`: Light algorithms, simple heuristics
- `1000m`: Standard algorithms, moderate computation
- `2000m`: Heavy algorithms, ML models
- `4000m`: Very heavy algorithms, deep learning

### Memory Limits

- `1Gi`: Small datasets, simple algorithms
- `2Gi`: Standard usage
- `4Gi`: Large datasets, complex algorithms
- `8Gi`: Very large datasets, memory-intensive algorithms

### Timeout

- `600s` (10 min): Quick algorithms
- `1800s` (30 min): Standard algorithms
- `3600s` (1 hour): Heavy algorithms
- `7200s` (2 hours): Very heavy algorithms

## Execution Lifecycle

1. **Pending**: Task queued, waiting for resources
2. **Scheduled**: Kubernetes job created
3. **Running**: Algorithm executing on datapack
4. **Completed**: Execution finished successfully
5. **Failed**: Execution failed (check logs)

## Response Format

```json
{
  "task_id": "task-abc123",
  "status": "pending",
  "algorithm": "my-rca",
  "version": "v1.0.0",
  "dataset": "trainticket-pandora-v1",
  "datapack_range": "0-99",
  "created_at": "2026-01-18T10:30:00Z",
  "estimated_duration": 1800
}
```

## Checking Execution Status

```python
from rcabench.openapi.api import ExecutionsApi

executions_api = ExecutionsApi(client)

# Get execution details
response = executions_api.get_execution_by_id(execution_id="exec-abc123")
execution = response.data

print(f"State: {execution.state}")  # Pending, Running, Completed, Error
print(f"Algorithm: {execution.algorithm_name}")
print(f"Dataset: {execution.dataset_name}")
```

## Error Handling

```python
from rcabench.openapi.exceptions import ApiException

try:
    response = api.submit_execution(request)
    print(f"Task submitted: {response.task_id}")
except ApiException as e:
    if e.status == 404:
        print("Dataset not found")
    elif e.status == 400:
        print(f"Invalid request: {e.body}")
    elif e.status == 401:
        print("Authentication failed")
    else:
        print(f"API error: {e}")
```

## Best Practices

1. **Test locally first**: Always test on local datasets before remote submission
2. **Start small**: Begin with a small datapack range (e.g., 0-9)
3. **Set appropriate limits**: Don't over-allocate resources
4. **Use versioning**: Tag images with semantic versions
5. **Add descriptions**: Include meaningful descriptions for tracking
6. **Monitor progress**: Check task status regularly
7. **Handle failures**: Implement retry logic for transient failures

## Troubleshooting

### Algorithm Not Found

```
Error: Algorithm 'my-rca' not found in registry
```

**Solution**: Verify the image is pushed to Harbor:

```bash
docker images | grep my-rca
harbor list-images algorithms/my-rca
```

### Dataset Not Found

```
Error: Dataset 'trainticket-pandora-v1' not found
```

**Solution**: List available datasets:

```python
from rcabench.openapi.api import DatasetsApi

datasets_api = DatasetsApi(client)
response = datasets_api.list_datasets()
print([d.name for d in response.data.items])
```

### Resource Quota Exceeded

```
Error: Insufficient resources to schedule task
```

**Solution**: Reduce resource limits or wait for resources to become available.

### Invalid Datapack Range

```
Error: Datapack range 0-999 exceeds dataset size (100 datapacks)
```

**Solution**: Check dataset size:

```python
dataset = dataset_api.get_dataset(name="trainticket-pandora-v1")
print(f"Total datapacks: {dataset.datapack_count}")
```

## Next Steps

- [Monitor Progress](../monitor-progress): Track execution status
- [Retrieve Results](../retrieve-results): Download evaluation metrics
