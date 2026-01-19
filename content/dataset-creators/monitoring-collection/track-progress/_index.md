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
from rcabench.openapi import ApiClient, Configuration, TaskApi

config = Configuration(host="${AEGISLAB_API_URL}")  # Default: http://10.10.10.220:8080
client = ApiClient(config)
task_api = TaskApi(client)

# Get task details
task = task_api.get_task(task_id="task-abc123")

print(f"Status: {task.status}")
print(f"Progress: {task.progress}%")
print(f"Phase: {task.current_phase}")
print(f"Started: {task.started_at}")
print(f"Elapsed: {task.elapsed_seconds}s")
```

### Task Phases

Fault injection tasks go through these phases:

1. **Pending**: Task queued, waiting for execution
2. **Preparing**: Setting up environment and load generation
3. **Injecting**: Applying fault to target system
4. **Collecting**: Gathering traces, metrics, and logs
5. **Processing**: Converting data to parquet format
6. **Completed**: Dataset ready for retrieval
7. **Failed**: Execution failed (check logs)

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
        task = task_api.get_task(task_id=task_id)

        print(f"Status: {task.status} - {task.current_phase} ({task.progress}%)")

        if task.status in ["completed", "failed", "cancelled"]:
            return task

        time.sleep(poll_interval)

# Wait for task
task = wait_for_completion("task-abc123")
print(f"Final status: {task.status}")
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
    response = fault_api.submit_injection(injection_req)
    task_ids.append(response.task_id)

# Monitor all tasks
while task_ids:
    for task_id in task_ids[:]:
        task = task_api.get_task(task_id=task_id)

        if task.status == "completed":
            print(f"Task {task_id}: Completed")
            task_ids.remove(task_id)
        elif task.status == "failed":
            print(f"Task {task_id}: Failed - {task.error_message}")
            task_ids.remove(task_id)
        else:
            print(f"Task {task_id}: {task.status} ({task.progress}%)")

    if task_ids:
        time.sleep(10)
```

### Summary Statistics

```python
# Get statistics for multiple tasks
tasks = [task_api.get_task(task_id=tid) for tid in task_ids]

completed = sum(1 for t in tasks if t.status == "completed")
failed = sum(1 for t in tasks if t.status == "failed")
running = sum(1 for t in tasks if t.status == "running")

print(f"Completed: {completed}/{len(tasks)}")
print(f"Failed: {failed}/{len(tasks)}")
print(f"Running: {running}/{len(tasks)}")
```

## Data Collection Metrics

### Trace Collection

```python
task = task_api.get_task(task_id="task-abc123")

print(f"Traces collected: {task.traces_collected}")
print(f"Spans collected: {task.spans_collected}")
print(f"Error spans: {task.error_spans}")
print(f"Trace duration: {task.trace_duration_seconds}s")
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

Cancel a running task:

```python
# Cancel task
task_api.cancel_task(task_id="task-abc123")

# Verify cancellation
task = task_api.get_task(task_id="task-abc123")
print(f"Status: {task.status}")  # Should be "cancelled"
```

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
