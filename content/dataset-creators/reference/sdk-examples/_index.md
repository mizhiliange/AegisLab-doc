---
title: SDK Examples
weight: 3
---

# SDK Examples

Common code examples for using the AegisLab Python SDK.

## Setup

### Installation

```bash
pip install rcabench-openapi
```

### Configuration

```python
from rcabench.openapi import ApiClient, Configuration

config = Configuration(
    host="${AEGISLAB_API_URL}",  # Default: http://10.10.10.220:8080
    api_key={"Authorization": "Bearer your-api-token"}
)

client = ApiClient(config)
```

## Fault Injection Examples

### Basic Network Delay

```python
from rcabench.openapi import FaultInjectionApi
from rcabench.openapi.models import DtoSubmitInjectionReq

fault_api = FaultInjectionApi(client)

request = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[
        {
            "handler": "network_delay",
            "params": {
                "delay": "100ms",
                "target_service": "ts-order-service",
                "target_namespace": "ts"
            }
        }
    ],
    duration=60,
    description="Network delay test"
)

response = fault_api.submit_injection(request)
print(f"Task ID: {response.task_id}")
```

### Pod Failure

```python
request = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[
        {
            "handler": "pod_failure",
            "params": {
                "target_service": "ts-payment-service",
                "target_namespace": "ts"
            }
        }
    ],
    duration=30
)

response = fault_api.submit_injection(request)
```

### Memory Pressure

```python
request = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[
        {
            "handler": "memory_pressure",
            "params": {
                "size": "512MB",
                "workers": 4,
                "target_service": "ts-order-service",
                "target_namespace": "ts"
            }
        }
    ],
    duration=60
)

response = fault_api.submit_injection(request)
```

### Multiple Faults

```python
request = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[
        {
            "handler": "network_delay",
            "params": {
                "delay": "100ms",
                "target_service": "ts-order-service",
                "target_namespace": "ts"
            }
        },
        {
            "handler": "cpu_stress",
            "params": {
                "workers": 2,
                "target_service": "ts-payment-service",
                "target_namespace": "ts"
            }
        }
    ],
    duration=60
)

response = fault_api.submit_injection(request)
```

## Monitoring Examples

### Poll Task Status

```python
from rcabench.openapi import TaskApi
import time

task_api = TaskApi(client)

def wait_for_completion(task_id):
    while True:
        task = task_api.get_task(task_id=task_id)

        print(f"Status: {task.status} ({task.progress}%)")

        if task.status in ["completed", "failed", "cancelled"]:
            return task

        time.sleep(10)

task = wait_for_completion("task-abc123")
print(f"Final status: {task.status}")
```

### Stream Events

```python
for event in task_api.stream_trace_events(task_id="task-abc123"):
    print(f"[{event.timestamp}] {event.level}: {event.message}")
```

### Get Collection Metrics

```python
task = task_api.get_task(task_id="task-abc123")

print(f"Traces collected: {task.traces_collected}")
print(f"Spans collected: {task.spans_collected}")
print(f"Metrics collected: {task.metrics_collected}")
print(f"Logs collected: {task.logs_collected}")
```

## Dataset Examples

### List Datasets

```python
from rcabench.openapi import DatasetApi

dataset_api = DatasetApi(client)

# List all datasets
datasets = dataset_api.list_datasets()

for ds in datasets:
    print(f"{ds.name}: {ds.datapack_count} datapacks")

# Filter by benchmark
trainticket_datasets = dataset_api.list_datasets(benchmark="trainticket")
```

### Get Dataset Info

```python
dataset = dataset_api.get_dataset(name="trainticket-pandora-v1")

print(f"Name: {dataset.name}")
print(f"Datapacks: {dataset.datapack_count}")
print(f"Size: {dataset.size_bytes / 1024**3:.2f} GB")
print(f"Created: {dataset.created_at}")
```

### Download Dataset

```python
dataset_api.download_dataset(
    dataset_id="trainticket-pandora-v1",
    output_path="./data/"
)

print("Dataset downloaded successfully")
```

### Download Specific Datapacks

```python
dataset_api.download_datapacks(
    dataset_id="trainticket-pandora-v1",
    datapack_ids=[0, 1, 2, 3, 4],
    output_path="./data/"
)
```

## Batch Operations

### Submit Multiple Injections

```python
fault_types = ["network_delay", "pod_failure", "memory_pressure"]
task_ids = []

for fault_type in fault_types:
    request = DtoSubmitInjectionReq(
        benchmark="trainticket",
        handler_nodes=[
            {
                "handler": fault_type,
                "params": {
                    "target_service": "ts-order-service",
                    "target_namespace": "ts",
                    # Add fault-specific params
                }
            }
        ],
        duration=60
    )

    response = fault_api.submit_injection(request)
    task_ids.append(response.task_id)

print(f"Submitted {len(task_ids)} tasks")
```

### Monitor Multiple Tasks

```python
def monitor_tasks(task_ids):
    completed = []

    while len(completed) < len(task_ids):
        for task_id in task_ids:
            if task_id in completed:
                continue

            task = task_api.get_task(task_id=task_id)

            if task.status == "completed":
                print(f"Task {task_id}: Completed")
                completed.append(task_id)
            elif task.status == "failed":
                print(f"Task {task_id}: Failed")
                completed.append(task_id)
            else:
                print(f"Task {task_id}: {task.status} ({task.progress}%)")

        if len(completed) < len(task_ids):
            time.sleep(10)

monitor_tasks(task_ids)
```

### Parallel Downloads

```python
from concurrent.futures import ThreadPoolExecutor

def download_dataset(name):
    dataset_api.download_dataset(
        dataset_id=name,
        output_path=f"./data/{name}/"
    )
    return name

datasets = ["trainticket-pandora-v1", "otel-demo-v1", "media-ms-v1"]

with ThreadPoolExecutor(max_workers=3) as executor:
    futures = [executor.submit(download_dataset, name) for name in datasets]

    for future in futures:
        name = future.result()
        print(f"Downloaded: {name}")
```

## Error Handling

### Basic Error Handling

```python
from rcabench.openapi.exceptions import ApiException

try:
    response = fault_api.submit_injection(request)
    print(f"Task ID: {response.task_id}")
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

### Retry Logic

```python
import time

def submit_with_retry(request, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = fault_api.submit_injection(request)
            return response
        except ApiException as e:
            if attempt < max_retries - 1:
                print(f"Attempt {attempt + 1} failed, retrying...")
                time.sleep(5)
            else:
                raise

response = submit_with_retry(request)
```

## Complete Workflow Example

```python
from rcabench.openapi import ApiClient, Configuration, FaultInjectionApi, TaskApi, DatasetApi
from rcabench.openapi.models import DtoSubmitInjectionReq
import time

# Setup
config = Configuration(host="${AEGISLAB_API_URL}")  # Default: http://10.10.10.220:8080
client = ApiClient(config)

fault_api = FaultInjectionApi(client)
task_api = TaskApi(client)
dataset_api = DatasetApi(client)

# Submit injection
request = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[
        {
            "handler": "network_delay",
            "params": {
                "delay": "100ms",
                "target_service": "ts-order-service",
                "target_namespace": "ts"
            }
        }
    ],
    duration=60
)

response = fault_api.submit_injection(request)
task_id = response.task_id
print(f"Task submitted: {task_id}")

# Monitor progress
while True:
    task = task_api.get_task(task_id=task_id)
    print(f"Status: {task.status} ({task.progress}%)")

    if task.status in ["completed", "failed"]:
        break

    time.sleep(10)

# Get dataset
if task.status == "completed":
    dataset_id = task.dataset_id
    print(f"Dataset ID: {dataset_id}")

    # Download dataset
    dataset_api.download_dataset(
        dataset_id=dataset_id,
        output_path="./data/"
    )

    print("Dataset downloaded successfully")
```

## See Also

- [API Reference](../api-reference): Complete API documentation
- [Fault Catalog](../fault-catalog): Available fault types
- [Using AegisLab](../../using-aegislab): SDK usage guide
