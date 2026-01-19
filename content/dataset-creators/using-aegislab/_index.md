---
title: Using AegisLab
weight: 3
---

# Using AegisLab

Learn how to interact with AegisLab for fault injection and dataset creation.

## Topics

{{< cards >}}
  {{< card link="frontend-guide" title="Frontend Guide" icon="document" >}}
  {{< card link="python-sdk" title="Python SDK" icon="code" >}}
  {{< card link="submit-injection" title="Submit Injection" icon="beaker" >}}
  {{< card link="batch-operations" title="Batch Operations" icon="document" >}}
{{< /cards >}}

## Overview

AegisLab provides multiple interfaces for fault injection:

- **Web Frontend**: Visual interface for interactive experimentation
- **Python SDK**: Programmatic access for automation and scripting
- **REST API**: Direct HTTP access for custom integrations

## Python SDK

### Installation

```bash
pip install rcabench-openapi
```

Or install from source:

```bash
git clone https://github.com/OperationsPAI/AegisLab.git
cd AegisLab/sdk/python
pip install -e .
```

### Configuration

Configure the SDK with your AegisLab endpoint:

```python
from rcabench.openapi import Configuration, ApiClient

# Configure API client (use your AEGISLAB_API_URL from .env)
config = Configuration(
    host="${AEGISLAB_API_URL}",  # Default: http://10.10.10.220:8080
    api_key={"Authorization": "Bearer your-api-token"}
)

client = ApiClient(config)
```

### Environment Variables

Set environment variables for convenience:

```bash
# Set in .env file or export directly (see .env.example)
export AEGISLAB_API_URL="http://10.10.10.220:8080"  # Default production endpoint
export AEGISLAB_TOKEN="your-api-token"
```

Then use in code:

```python
import os
from rcabench.openapi import Configuration, ApiClient

config = Configuration(
    host=os.getenv("AEGISLAB_API_URL"),
    api_key={"Authorization": f"Bearer {os.getenv('AEGISLAB_TOKEN')}"}
)
```

## Submitting Fault Injection

### Basic Injection

```python
from rcabench.openapi import FaultInjectionApi
from rcabench.openapi.models import DtoSubmitInjectionReq

api = FaultInjectionApi(client)

# Define fault injection
req = DtoSubmitInjectionReq(
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

# Submit injection
response = api.submit_injection(req)
print(f"Task ID: {response.task_id}")
```

### Multiple Faults

Inject multiple faults simultaneously:

```python
req = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[
        {
            "handler": "network_delay",
            "params": {
                "delay": "100ms",
                "target_service": "ts-order-service"
            }
        },
        {
            "handler": "cpu_stress",
            "params": {
                "target_service": "ts-payment-service",
                "workers": 2,
                "load": 80
            }
        }
    ],
    duration=120
)

response = api.submit_injection(req)
```

## Monitoring Progress

### Get Task Status

```python
from rcabench.openapi import TaskApi

task_api = TaskApi(client)

# Get task details
task = task_api.get_task(task_id=response.task_id)

print(f"Status: {task.status}")
print(f"Progress: {task.progress}%")
print(f"Created: {task.created_at}")
print(f"Updated: {task.updated_at}")
```

### Stream Trace Events

Monitor real-time progress:

```python
# Stream trace events
for event in task_api.stream_trace_events(task_id=response.task_id):
    timestamp = event.timestamp
    level = event.level
    message = event.message
    print(f"[{timestamp}] {level}: {message}")
```

### Poll for Completion

```python
import time

def wait_for_completion(task_id, timeout=600, interval=5):
    """Wait for task to complete."""
    start_time = time.time()

    while time.time() - start_time < timeout:
        task = task_api.get_task(task_id=task_id)

        if task.status == "completed":
            print("Task completed successfully")
            return task
        elif task.status == "failed":
            print(f"Task failed: {task.error_message}")
            return task

        print(f"Status: {task.status}, Progress: {task.progress}%")
        time.sleep(interval)

    raise TimeoutError(f"Task did not complete within {timeout}s")

# Use it
task = wait_for_completion(response.task_id)
```

## Managing Datasets

### List Datasets

```python
from rcabench.openapi import DatasetApi

dataset_api = DatasetApi(client)

# List all datasets
datasets = dataset_api.list_datasets()
for ds in datasets:
    print(f"{ds.id}: {ds.name} ({ds.datapack_count} datapacks)")

# Filter by benchmark
datasets = dataset_api.list_datasets(benchmark="trainticket")
```

### Get Dataset Details

```python
# Get specific dataset
dataset = dataset_api.get_dataset(dataset_id="trainticket-pandora-v1")

print(f"Name: {dataset.name}")
print(f"Benchmark: {dataset.benchmark}")
print(f"Datapacks: {dataset.datapack_count}")
print(f"Created: {dataset.created_at}")
print(f"Size: {dataset.size_bytes / 1024 / 1024:.2f} MB")
```

### Download Dataset

```python
# Download entire dataset
dataset_api.download_dataset(
    dataset_id="trainticket-pandora-v1",
    output_path="./data"
)

# Download specific datapack
dataset_api.download_datapack(
    dataset_id="trainticket-pandora-v1",
    datapack_id="0",
    output_path="./data/datapack_0"
)
```

## Batch Operations

### Submit Multiple Injections

```python
# Define multiple injection requests
injections = [
    {
        "handler": "network_delay",
        "params": {"delay": "50ms", "target_service": "ts-order-service"}
    },
    {
        "handler": "network_delay",
        "params": {"delay": "100ms", "target_service": "ts-order-service"}
    },
    {
        "handler": "network_delay",
        "params": {"delay": "200ms", "target_service": "ts-order-service"}
    }
]

# Submit all injections
task_ids = []
for injection in injections:
    req = DtoSubmitInjectionReq(
        benchmark="trainticket",
        handler_nodes=[injection],
        duration=60
    )
    response = api.submit_injection(req)
    task_ids.append(response.task_id)

print(f"Submitted {len(task_ids)} injections")
```

### Monitor Multiple Tasks

```python
import concurrent.futures

def check_task_status(task_id):
    """Check status of a single task."""
    task = task_api.get_task(task_id=task_id)
    return {
        "task_id": task_id,
        "status": task.status,
        "progress": task.progress
    }

# Check all tasks in parallel
with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
    futures = [executor.submit(check_task_status, tid) for tid in task_ids]
    results = [f.result() for f in concurrent.futures.as_completed(futures)]

for result in results:
    print(f"Task {result['task_id']}: {result['status']} ({result['progress']}%)")
```

## Error Handling

### Handle API Errors

```python
from rcabench.openapi.exceptions import ApiException

try:
    response = api.submit_injection(req)
except ApiException as e:
    print(f"API Error: {e.status} - {e.reason}")
    print(f"Body: {e.body}")
except Exception as e:
    print(f"Unexpected error: {e}")
```

### Retry Logic

```python
import time
from rcabench.openapi.exceptions import ApiException

def submit_with_retry(api, req, max_retries=3, backoff=2):
    """Submit injection with exponential backoff retry."""
    for attempt in range(max_retries):
        try:
            return api.submit_injection(req)
        except ApiException as e:
            if e.status >= 500 and attempt < max_retries - 1:
                wait_time = backoff ** attempt
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise

response = submit_with_retry(api, req)
```

## Advanced Usage

### Custom Timeout

```python
# Configure custom timeout
config = Configuration(
    host="http://10.10.10.220:8080",
    timeout=300  # 5 minutes
)
```

### Async Operations

```python
import asyncio
from rcabench.openapi import AsyncFaultInjectionApi

async def submit_async():
    async with AsyncApiClient(config) as client:
        api = AsyncFaultInjectionApi(client)
        response = await api.submit_injection(req)
        return response

# Run async
response = asyncio.run(submit_async())
```

### Pagination

```python
# List datasets with pagination
page = 1
page_size = 20

while True:
    datasets = dataset_api.list_datasets(
        page=page,
        page_size=page_size
    )

    if not datasets:
        break

    for ds in datasets:
        print(f"{ds.id}: {ds.name}")

    page += 1
```

## Complete Example

Here's a complete workflow:

```python
#!/usr/bin/env python3
"""Complete fault injection workflow example."""

import os
import time
from rcabench.openapi import Configuration, ApiClient
from rcabench.openapi import FaultInjectionApi, TaskApi, DatasetApi
from rcabench.openapi.models import DtoSubmitInjectionReq

def main():
    # Configure client
    config = Configuration(
        host=os.getenv("AEGISLAB_API_URL", "http://10.10.10.220:8080")
    )
    client = ApiClient(config)

    # Initialize APIs
    fault_api = FaultInjectionApi(client)
    task_api = TaskApi(client)
    dataset_api = DatasetApi(client)

    # Submit fault injection
    print("Submitting fault injection...")
    req = DtoSubmitInjectionReq(
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
        duration=60,
        description="Example fault injection"
    )

    response = fault_api.submit_injection(req)
    task_id = response.task_id
    print(f"Task submitted: {task_id}")

    # Monitor progress
    print("\nMonitoring progress...")
    while True:
        task = task_api.get_task(task_id=task_id)
        print(f"Status: {task.status}, Progress: {task.progress}%")

        if task.status in ["completed", "failed"]:
            break

        time.sleep(5)

    # Check result
    if task.status == "completed":
        print("\nTask completed successfully!")

        # Get dataset
        dataset_id = task.dataset_id
        dataset = dataset_api.get_dataset(dataset_id=dataset_id)
        print(f"Dataset created: {dataset.name}")
        print(f"Datapacks: {dataset.datapack_count}")

        # Download dataset
        print("\nDownloading dataset...")
        dataset_api.download_dataset(
            dataset_id=dataset_id,
            output_path="./output"
        )
        print("Download complete!")
    else:
        print(f"\nTask failed: {task.error_message}")

if __name__ == "__main__":
    main()
```

## Next Steps

- [Submit Injection](submit-injection): Detailed injection request format
- [Batch Operations](batch-operations): Advanced batch processing
- [Monitoring & Collection](../monitoring-collection): Track data collection
