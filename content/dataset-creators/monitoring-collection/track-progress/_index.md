---
title: Track Progress
weight: 1
---

Monitor fault injection execution and data collection progress.

## Prerequisites

- Fault injection submitted via API or SDK
- Task ID from submission response

## Using the Python SDK

### Get Task Status

```python
from rcabench.openapi import ApiClient, Configuration
from rcabench.openapi.api import TasksApi

config = Configuration(host="${AEGISLAB_API_URL}")  # Default: http://10.10.10.220:32080
client = ApiClient(config)
tasks_api = TasksApi(client)

# Get task details
response = tasks_api.get_task_by_id(task_id="task-abc123")
task = response.data

print(f"State: {task.state}")  # Pending, Running, Completed, Error, Cancelled
print(f"Type: {task.type}")
print(f"Created: {task.created_at}")
```

### Task States

Fault injection tasks have these states:

| State | Value | Description |
|-------|-------|-------------|
| **Cancelled** | -2 | Task was cancelled by user |
| **Error** | -1 | Task failed with errors |
| **Pending** | 0 | Task queued, waiting for execution |
| **Rescheduled** | 1 | Task rescheduled for later execution |
| **Running** | 2 | Task actively executing (fault injection + data collection) |
| **Completed** | 3 | Task finished successfully, dataset ready |

### Stream Real-Time Events

```python
# Stream trace events
for event in task_api.stream_trace_events(task_id="task-abc123"):
    timestamp = event.timestamp.strftime("%H:%M:%S")
    print(f"[{timestamp}] {event.level}: {event.message}")
```

Example output:

```
[10:30:15] INFO: Task started
[10:30:16] INFO: Starting load generator (10 threads)
[10:30:20] INFO: Load generation stable
[10:30:25] INFO: Applying network delay to ts-order-service
[10:30:25] INFO: Fault injection active
[10:31:25] INFO: Fault injection completed (60s duration)
[10:31:26] INFO: Collecting traces from ClickHouse
[10:31:35] INFO: Collected 15,234 spans
[10:31:36] INFO: Collecting metrics from Prometheus
[10:31:40] INFO: Collected 8,456 metric points
[10:31:41] INFO: Processing data to parquet format
[10:31:50] INFO: Dataset ready: trainticket-custom-001
[10:31:50] INFO: Task completed successfully
```

### Poll for Completion

```python
import time

def wait_for_completion(task_id, poll_interval=10):
    """Wait for task to complete."""
    while True:
        response = tasks_api.get_task_by_id(task_id=task_id)
        task = response.data

        print(f"State: {task.state}")

        # Check terminal states
        if task.state in ["Completed", "Error", "Cancelled"]:
            return task

        time.sleep(poll_interval)

# Wait for task
task = wait_for_completion("task-abc123")
print(f"Final state: {task.state}")
```

## Using the Web Dashboard

Access the web dashboard for visual monitoring:

```
https://aegislab.io/dashboard/tasks/task-abc123
```

The dashboard shows:
- Real-time progress bar
- Current phase and status
- Live event stream
- Resource usage graphs
- Data collection statistics

## Monitoring Multiple Tasks

### Track Batch Submissions

```python
# Submit multiple injections
task_ids = []
for i in range(10):
    response = injections_api.inject_fault(injection_req)
    for item in response.data:
        task_ids.append(item.task_id)

# Monitor all tasks
while task_ids:
    for task_id in task_ids[:]:
        response = tasks_api.get_task_by_id(task_id=task_id)
        task = response.data

        if task.state == "Completed":
            print(f"Task {task_id}: Completed")
            task_ids.remove(task_id)
        elif task.state == "Error":
            print(f"Task {task_id}: Error")
            task_ids.remove(task_id)
        elif task.state == "Cancelled":
            print(f"Task {task_id}: Cancelled")
            task_ids.remove(task_id)
        else:
            print(f"Task {task_id}: {task.state}")

    if task_ids:
        time.sleep(10)
```

### Summary Statistics

```python
# Get statistics for multiple tasks
tasks = [tasks_api.get_task_by_id(task_id=tid).data for tid in task_ids]

completed = sum(1 for t in tasks if t.state == "Completed")
failed = sum(1 for t in tasks if t.state == "Error")
running = sum(1 for t in tasks if t.state == "Running")

print(f"Completed: {completed}/{len(tasks)}")
print(f"Failed: {failed}/{len(tasks)}")
print(f"Running: {running}/{len(tasks)}")
```

## Data Collection Metrics

### Trace Collection

```python
response = tasks_api.get_task_by_id(task_id="task-abc123")
task = response.data

# Task metadata contains collection statistics
print(f"Task ID: {task.id}")
print(f"State: {task.state}")
print(f"Type: {task.type}")
```

### Metrics Collection

```python
print(f"Metric points: {task.metrics_collected}")
print(f"Metric series: {task.metric_series_count}")
print(f"Collection window: {task.metric_window_seconds}s")
```

### Logs Collection

```python
print(f"Log entries: {task.logs_collected}")
print(f"Error logs: {task.error_logs}")
print(f"Warning logs: {task.warning_logs}")
```

## Notifications

### Email Notifications

Configure email notifications for task completion:

```python
request = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[...],
    duration=60,

    # Enable notifications
    notify_on_completion=True,
    notify_on_failure=True,
    notification_email="user@example.com"
)
```

### Webhook Notifications

Set up webhook for programmatic notifications:

```python
request = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[...],
    duration=60,

    # Webhook configuration
    webhook_url="https://myapp.com/webhooks/aegislab",
    webhook_events=["completed", "failed", "data_collected"]
)
```

Webhook payload:

```json
{
  "event": "completed",
  "task_id": "task-abc123",
  "status": "completed",
  "dataset_id": "trainticket-custom-001",
  "traces_collected": 15234,
  "spans_collected": 15234,
  "metrics_collected": 8456,
  "timestamp": "2026-01-18T11:30:00Z"
}
```

## Cancelling Tasks

Cancel a running task using the API or kubectl:

```bash
# Using kubectl to delete the chaos resource
kubectl delete networkchaos,podchaos,stresschaos -n ts -l task-id=task-abc123
```

After cancellation, the task state will be updated to "Cancelled" (-2).

## Viewing Logs

### Task Logs

```python
# Get logs
logs = task_api.get_logs(task_id="task-abc123")

print("STDOUT:")
print(logs.stdout)

print("\nSTDERR:")
print(logs.stderr)
```

### Kubernetes Logs

Access raw Kubernetes pod logs:

```bash
# Get pod name
kubectl get pods -n aegislab -l task-id=task-abc123

# View logs
kubectl logs -n aegislab pod/task-abc123-worker-0

# Follow logs
kubectl logs -n aegislab pod/task-abc123-worker-0 -f
```

## Troubleshooting

### Task Stuck in Pending

```
Status: pending (0%)
```

**Possible causes**:
- High queue backlog
- Insufficient cluster resources
- Load generator not available

**Solution**: Check cluster status or wait for resources.

### Task Failed During Injection

```
Status: failed
Phase: injecting
Error: Failed to apply chaos: pod not found
```

**Solution**: Verify target service exists:

```bash
kubectl get pods -n ts | grep ts-order-service
```

### No Traces Collected

```
Status: completed
Traces collected: 0
```

**Possible causes**:
- OpenTelemetry not configured
- Collector not receiving traces
- Wrong time window

**Solution**: Check observability stack:

```bash
# Check OTEL collector
kubectl logs -n observability otel-collector-0

# Verify traces in ClickHouse
clickhouse-client --query "SELECT count() FROM traces WHERE timestamp > now() - 300"
```

### Incomplete Data Collection

```
Status: completed
Traces collected: 234 (expected ~15000)
```

**Solution**: Check collection window and load generation:

```python
# Increase collection window
request = DtoSubmitInjectionReq(
    benchmark="trainticket",
    handler_nodes=[...],
    duration=60,
    collection_window=120  # Collect for 2 minutes after injection
)
```

## Best Practices

1. **Monitor actively**: Check progress regularly during execution
2. **Set up notifications**: Use webhooks for long-running tasks
3. **Verify data quality**: Check collection metrics after completion
4. **Handle failures**: Implement retry logic for transient failures
5. **Log everything**: Keep detailed logs for debugging

## Next Steps

- [Observability Data](../observability-data): Understanding collected data
- [Troubleshooting](../troubleshooting): Debug collection issues
- [Dataset Management](../../dataset-management): Retrieve datasets
