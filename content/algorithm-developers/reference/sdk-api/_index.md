---
title: SDK API
weight: 2
---

Complete API reference for the AegisLab Python SDK.

## Installation

```bash
pip install rcabench-openapi
```

## Configuration

### ApiClient

```python
from rcabench.openapi import ApiClient, Configuration

config = Configuration(
    host="${AEGISLAB_API_URL}",  # Use environment variable
    api_key={"Authorization": "Bearer your-token"}
)
# Default: http://10.10.10.220:32080

client = ApiClient(config)
```

**Configuration Parameters:**
- `host`: API endpoint URL
- `api_key`: Authentication credentials
- `timeout`: Request timeout in seconds (default: 30)
- `verify_ssl`: Verify SSL certificates (default: True)

## API Classes

### AlgorithmApi

Manage algorithm execution and evaluation.

```python
from rcabench.openapi import AlgorithmApi

api = AlgorithmApi(client)
```

#### submit_execution

Submit algorithm for remote execution.

```python
from rcabench.openapi.models import DtoSubmitExecutionReq

request = DtoSubmitExecutionReq(
    algorithm_name="my-rca",
    algorithm_version="v1.0.0",
    dataset="trainticket-pandora-v1",
    datapack_start=0,
    datapack_end=99,
    cpu_limit="2000m",
    memory_limit="4Gi",
    timeout_seconds=3600
)

response = api.submit_execution(request)
print(response.task_id)
```

**Parameters:**
- `algorithm_name` (str, required): Algorithm name
- `algorithm_version` (str): Version tag (default: "latest")
- `dataset` (str, required): Dataset name
- `datapack_start` (int, required): Starting datapack ID
- `datapack_end` (int, required): Ending datapack ID
- `cpu_limit` (str): CPU limit (e.g., "2000m")
- `memory_limit` (str): Memory limit (e.g., "4Gi")
- `timeout_seconds` (int): Timeout in seconds
- `description` (str): Task description
- `tags` (list[str]): Task tags

**Returns:** `DtoSubmitExecutionResponse`
- `task_id` (str): Unique task identifier
- `status` (str): Initial status ("pending")
- `created_at` (datetime): Creation timestamp

### TaskApi

Monitor and manage task execution.

```python
from rcabench.openapi import TaskApi

task_api = TaskApi(client)
```

#### get_task

Get task details and status.

```python
task = task_api.get_task(task_id="task-abc123")

print(f"Status: {task.status}")
print(f"Progress: {task.progress}%")
print(f"Completed: {task.completed_datapacks}/{task.total_datapacks}")
```

**Parameters:**
- `task_id` (str, required): Task identifier

**Returns:** `DtoTask`
- `task_id` (str): Task identifier
- `status` (str): Current status
- `progress` (float): Progress percentage (0-100)
- `total_datapacks` (int): Total number of datapacks
- `completed_datapacks` (int): Completed datapacks
- `failed_datapacks` (int): Failed datapacks
- `started_at` (datetime): Start timestamp
- `completed_at` (datetime): Completion timestamp
- `elapsed_seconds` (float): Elapsed time

#### stream_trace_events

Stream real-time execution events.

```python
for event in task_api.stream_trace_events(task_id="task-abc123"):
    print(f"[{event.timestamp}] {event.level}: {event.message}")
```

**Parameters:**
- `task_id` (str, required): Task identifier
- `level` (str): Filter by log level (INFO, WARNING, ERROR)

**Yields:** `DtoTraceEvent`
- `timestamp` (datetime): Event timestamp
- `level` (str): Log level
- `message` (str): Event message
- `datapack_id` (int): Associated datapack ID

#### cancel_task

Cancel a running task.

```python
task_api.cancel_task(task_id="task-abc123")
```

**Parameters:**
- `task_id` (str, required): Task identifier

**Returns:** `DtoCancelTaskResponse`
- `success` (bool): Cancellation success
- `message` (str): Status message

### ResultsApi

Retrieve evaluation results and metrics.

```python
from rcabench.openapi import ResultsApi

results_api = ResultsApi(client)
```

#### get_task_results

Get aggregate results for a task.

```python
results = results_api.get_task_results(task_id="task-abc123")

print(f"Mean MRR: {results.mean_mrr:.3f}")
print(f"Mean Top-1: {results.mean_top1_accuracy:.3f}")
```

**Parameters:**
- `task_id` (str, required): Task identifier

**Returns:** `DtoTaskResults`
- `task_id` (str): Task identifier
- `algorithm_name` (str): Algorithm name
- `algorithm_version` (str): Algorithm version
- `dataset` (str): Dataset name
- `mean_mrr` (float): Mean reciprocal rank
- `std_mrr` (float): Standard deviation of MRR
- `mean_avg1` (float): Mean Avg@1
- `mean_avg3` (float): Mean Avg@3
- `mean_avg5` (float): Mean Avg@5
- `mean_top1_accuracy` (float): Mean Top-1 accuracy
- `mean_top3_accuracy` (float): Mean Top-3 accuracy
- `mean_top5_accuracy` (float): Mean Top-5 accuracy

#### get_datapack_result

Get results for a specific datapack.

```python
result = results_api.get_datapack_result(
    task_id="task-abc123",
    datapack_id=0
)

print(f"MRR: {result.mrr:.3f}")
for service in result.ranked_services:
    print(f"  {service.name}: {service.score:.3f}")
```

**Parameters:**
- `task_id` (str, required): Task identifier
- `datapack_id` (int, required): Datapack ID

**Returns:** `DtoDatapackResult`
- `datapack_id` (int): Datapack ID
- `status` (str): Execution status
- `mrr` (float): Mean reciprocal rank
- `top1_accuracy` (float): Top-1 accuracy
- `top3_accuracy` (float): Top-3 accuracy
- `ranked_services` (list[DtoRankedService]): Ranked services
- `ground_truth` (list[str]): Ground truth services
- `execution_time_seconds` (float): Execution time

#### download_results

Download full results as JSON.

```python
results_api.download_results(
    task_id="task-abc123",
    output_path="./results/task-abc123.json"
)
```

**Parameters:**
- `task_id` (str, required): Task identifier
- `output_path` (str, required): Output file path

### DatasetApi

Manage and query datasets.

```python
from rcabench.openapi import DatasetApi

dataset_api = DatasetApi(client)
```

#### list_datasets

List available datasets.

```python
datasets = dataset_api.list_datasets(benchmark="trainticket")

for ds in datasets:
    print(f"{ds.name}: {ds.datapack_count} datapacks")
```

**Parameters:**
- `benchmark` (str): Filter by benchmark system

**Returns:** `list[DtoDataset]`
- `name` (str): Dataset name
- `benchmark` (str): Benchmark system
- `datapack_count` (int): Number of datapacks
- `size_bytes` (int): Total size in bytes
- `created_at` (datetime): Creation timestamp
- `description` (str): Dataset description

#### get_dataset

Get detailed dataset information.

```python
dataset = dataset_api.get_dataset(name="trainticket-pandora-v1")

print(f"Datapacks: {dataset.datapack_count}")
print(f"Size: {dataset.size_bytes / 1024**3:.2f} GB")
```

**Parameters:**
- `name` (str, required): Dataset name

**Returns:** `DtoDataset`

### FaultInjectionApi

Submit and manage fault injection tasks (for dataset creators).

```python
from rcabench.openapi import FaultInjectionApi

fault_api = FaultInjectionApi(client)
```

#### submit_injection

Submit fault injection request.

```python
from rcabench.openapi.models import DtoSubmitInjectionReq

request = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[
        {
            "handler": "network_delay",
            "params": {
                "delay": "100ms",
                "target_service": "ts-order-service"
            }
        }
    ],
    duration=60
)

response = fault_api.submit_injection(request)
print(response.task_id)
```

## Data Models

### DtoSubmitExecutionReq

```python
class DtoSubmitExecutionReq:
    algorithm_name: str
    algorithm_version: str = "latest"
    dataset: str
    datapack_start: int
    datapack_end: int
    cpu_limit: str = "2000m"
    memory_limit: str = "4Gi"
    timeout_seconds: int = 1800
    description: str = ""
    tags: list[str] = []
```

### DtoTask

```python
class DtoTask:
    task_id: str
    status: str
    progress: float
    total_datapacks: int
    completed_datapacks: int
    failed_datapacks: int
    started_at: datetime
    completed_at: datetime
    elapsed_seconds: float
```

### DtoTaskResults

```python
class DtoTaskResults:
    task_id: str
    algorithm_name: str
    dataset: str
    mean_mrr: float
    std_mrr: float
    mean_top1_accuracy: float
    mean_top3_accuracy: float
```

### DtoDatapackResult

```python
class DtoDatapackResult:
    datapack_id: int
    status: str
    mrr: float
    top1_accuracy: float
    ranked_services: list[DtoRankedService]
    ground_truth: list[str]
```

### DtoRankedService

```python
class DtoRankedService:
    name: str
    score: float
    rank: int
```

## Error Handling

```python
from rcabench.openapi.exceptions import ApiException

try:
    response = api.submit_execution(request)
except ApiException as e:
    if e.status == 404:
        print("Resource not found")
    elif e.status == 400:
        print(f"Bad request: {e.body}")
    elif e.status == 401:
        print("Authentication failed")
    else:
        print(f"API error: {e}")
```

## Complete Example

```python
from rcabench.openapi import ApiClient, Configuration, AlgorithmApi, TaskApi, ResultsApi
from rcabench.openapi.models import DtoSubmitExecutionReq
import time

# Configure client
config = Configuration(host="http://10.10.10.220:32080")
client = ApiClient(config)

# Submit execution
algo_api = AlgorithmApi(client)
request = DtoSubmitExecutionReq(
    algorithm_name="my-rca",
    dataset="trainticket-pandora-v1",
    datapack_start=0,
    datapack_end=99
)
response = algo_api.submit_execution(request)
task_id = response.task_id

# Monitor progress
task_api = TaskApi(client)
while True:
    task = task_api.get_task(task_id=task_id)
    print(f"Progress: {task.progress}%")

    if task.status in ["completed", "failed"]:
        break

    time.sleep(10)

# Get results
if task.status == "completed":
    results_api = ResultsApi(client)
    results = results_api.get_task_results(task_id=task_id)
    print(f"Mean MRR: {results.mean_mrr:.3f}")
```

## See Also

- [CLI Commands](../cli-commands): Command-line interface reference
- [Remote Evaluation](../../remote-evaluation): Remote execution guide
