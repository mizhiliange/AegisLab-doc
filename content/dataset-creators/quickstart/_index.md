---
title: Quickstart
weight: 1
---

# Dataset Creator Quickstart

Submit your first fault injection and monitor data collection in under 10 minutes.

## Prerequisites

- Python 3.10 or higher
- Access to AegisLab API endpoint
- API credentials (token or username/password)

## Step 1: Install AegisLab SDK

Install the AegisLab Python SDK:

```bash
pip install rcabench-openapi
```

Or if using the development version from the repository:

```bash
git clone https://github.com/OperationsPAI/AegisLab.git
cd AegisLab/sdk/python
pip install -e .
```

## Step 2: Configure Credentials

Set your AegisLab API endpoint and credentials:

```bash
export AEGISLAB_API_URL="http://10.10.10.220:8080"
export AEGISLAB_TOKEN="your-api-token"
```

Or configure in your Python code:

```python
from rcabench.openapi import Configuration, ApiClient

config = Configuration(
    host="http://10.10.10.220:8080",
    api_key={"Authorization": "Bearer your-api-token"}
)
```

## Step 3: Submit Your First Fault Injection

Create a simple fault injection script:

```python
from rcabench.openapi import ApiClient, Configuration, FaultInjectionApi
from rcabench.openapi.models import DtoSubmitInjectionReq

# Configure API client
config = Configuration(host="http://10.10.10.220:8080")
client = ApiClient(config)
api = FaultInjectionApi(client)

# Define fault injection request
injection_req = DtoSubmitInjectionReq(
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
    duration=60,  # Fault duration in seconds
    description="First fault injection test"
)

# Submit injection
response = api.submit_injection(injection_req)
print(f"Injection submitted! Task ID: {response.task_id}")
```

Save this as `first_injection.py` and run:

```bash
python first_injection.py
```

## Step 4: Monitor Progress

Track your fault injection progress using the task ID:

```python
from rcabench.openapi import TaskApi

task_api = TaskApi(client)

# Get task status
task = task_api.get_task(task_id=response.task_id)
print(f"Status: {task.status}")
print(f"Progress: {task.progress}%")

# Stream trace events
for event in task_api.stream_trace_events(task_id=response.task_id):
    print(f"[{event.timestamp}] {event.message}")
```

## Understanding the Output

Your fault injection will go through these stages:

1. **Pending**: Task queued for execution
2. **Running**: Fault being applied to target service
3. **Collecting**: Traces, metrics, and logs being collected
4. **Processing**: Data being converted to parquet format
5. **Completed**: Dataset ready for retrieval

## Step 5: Retrieve the Dataset

Once completed, retrieve your dataset:

```python
from rcabench.openapi import DatasetApi

dataset_api = DatasetApi(client)

# List available datasets
datasets = dataset_api.list_datasets(benchmark="trainticket")
print(f"Found {len(datasets)} datasets")

# Download specific dataset
dataset = dataset_api.get_dataset(dataset_id=response.dataset_id)
dataset_api.download_dataset(dataset_id=response.dataset_id, output_path="./data")
```

The dataset will contain:
- `traces.parquet`: Distributed traces with spans
- `metrics.parquet`: Time-series metrics
- `logs.parquet`: Structured logs
- `ground_truth.parquet`: Fault injection metadata

## Next Steps

Now that you've submitted your first fault injection:

1. **Understand fault types**: Read [Fault Types](../fault-injection-guide/fault-types) to learn about available fault types
2. **Use the SDK**: Explore [Python SDK](../using-aegislab/python-sdk) for advanced usage
3. **Monitor collection**: Learn about [Monitoring & Collection](../monitoring-collection) to track data quality
4. **Intelligent scheduling**: Use [Pandora](../advanced-topics/genetic-algorithms) for genetic algorithm-based scheduling

## Troubleshooting

**Connection refused**: Verify the AegisLab API endpoint is accessible and running.

**Authentication failed**: Check your API token or credentials are correct.

**Fault not applied**: Ensure the target service exists in the specified namespace and Chaos Mesh is installed.

**No traces collected**: Verify OpenTelemetry instrumentation is enabled on target services.

For more help, see [Troubleshooting](../monitoring-collection/troubleshooting).
