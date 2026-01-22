---
title: Quickstart
weight: 1
---

Submit your first fault injection and monitor data collection in under 10 minutes.

## Prerequisites

- Python 3.10 or higher
- Access to AegisLab API endpoint
- API credentials (token or username/password)

## Step 1: Install AegisLab SDK

Install the AegisLab Python SDK:

```bash
pip install rcabench
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
# Set your AegisLab API URL (see .env.example for defaults)
export AEGISLAB_API_URL="http://10.10.10.220:32080"  # Default production endpoint
export AEGISLAB_TOKEN="your-api-token"
```

Or configure in your Python code:

```python
from rcabench.openapi import Configuration, ApiClient

config = Configuration(
    host="${AEGISLAB_API_URL}",  # Use environment variable, default: http://10.10.10.220:32080
    api_key={"Authorization": "Bearer your-api-token"}
)
```

## Step 3: Submit Your First Fault Injection

Create a simple fault injection script:

```python
from rcabench.openapi import ApiClient, Configuration
from rcabench.openapi.api import InjectionsApi
from rcabench.openapi.models import SubmitInjectionReq, ChaosNode, ContainerSpec

# Configure API client
config = Configuration(host="http://10.10.10.220:32080")
client = ApiClient(config)
api = InjectionsApi(client)

# Define fault injection specs using ChaosNode structure
# This creates a network delay fault on ts-order-service
fault_spec = ChaosNode(
    name="NetworkDelay",
    children={
        "service": ChaosNode(name="ts-order-service"),
        "delay": ChaosNode(name="100ms"),
        "namespace": ChaosNode(name="ts")
    }
)

# Define fault injection request
injection_req = SubmitInjectionReq(
    project_name="my-project",
    benchmark=ContainerSpec(name="trainticket"),
    pedestal=ContainerSpec(name="loadgenerator"),
    specs=[[fault_spec]],  # 2D array: each sub-array is a batch of parallel faults
    interval=5,  # Total experiment interval in minutes
    pre_duration=1,  # Normal data collection before fault injection (minutes)
)

# Submit injection
response = api.inject_fault(injection_req)
print(f"Injection submitted!")
for item in response.data:
    print(f"  Task ID: {item.task_id}, Trace ID: {item.trace_id}")
```

Save this as `first_injection.py` and run:

```bash
python first_injection.py
```

## Step 4: Monitor Progress

Track your fault injection progress using the task ID:

```python
from rcabench.openapi.api import TasksApi

tasks_api = TasksApi(client)

# Get task status
task = tasks_api.get_task_by_id(task_id=item.task_id)
print(f"State: {task.data.state}")  # Pending, Running, Completed, Error, Cancelled
```

## Understanding the Output

Your fault injection will go through these states:

1. **Pending** (0): Task queued for execution
2. **Rescheduled** (1): Task rescheduled for later execution
3. **Running** (2): Fault being applied and data being collected
4. **Completed** (3): Dataset ready for retrieval
5. **Error** (-1): Task failed with errors
6. **Cancelled** (-2): Task was cancelled

## Step 5: Retrieve the Dataset

Once completed, retrieve your dataset:

```python
from rcabench.openapi.api import DatasetsApi

datasets_api = DatasetsApi(client)

# List available datasets
response = datasets_api.list_datasets()
print(f"Found {len(response.data.items)} datasets")

# Get specific dataset by ID
dataset = datasets_api.get_dataset_by_id(dataset_id="your-dataset-id")
print(f"Dataset: {dataset.data.name}")

# Download dataset version
datasets_api.download_dataset_version(
    dataset_id="your-dataset-id",
    version_id="your-version-id"
)
```

The dataset will contain:
- `abnormal_traces.parquet`: Distributed traces during fault injection
- `normal_traces.parquet`: Baseline traces before fault injection
- `abnormal_metrics.parquet`: Time-series metrics during fault
- `normal_metrics.parquet`: Baseline metrics
- `abnormal_logs.parquet`: Structured logs during fault
- `normal_logs.parquet`: Baseline logs
- `injection.json`: Fault injection metadata and ground truth
- `conclusion.parquet`: Span-level performance comparison

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
