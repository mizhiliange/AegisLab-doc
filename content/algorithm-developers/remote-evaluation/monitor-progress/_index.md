---
title: Monitor Progress
weight: 3
---

Track the execution status of your remote evaluation tasks.

## Prerequisites

- Task submitted via API or CLI
- Task ID from submission response
- AegisLab SDK installed

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
print(f"Completed: {task.completed_datapacks}/{task.total_datapacks}")
print(f"Started: {task.started_at}")
print(f"Duration: {task.elapsed_seconds}s")
```

### Stream Trace Events

Monitor real-time execution events:

```python
# Stream trace events
for event in task_api.stream_trace_events(task_id="task-abc123"):
    timestamp = event.timestamp.strftime("%H:%M:%S")
    print(f"[{timestamp}] {event.level}: {event.message}")
```

Example output:

```
[10:30:15] INFO: Task started
[10:30:16] INFO: Loading dataset trainticket-pandora-v1
[10:30:18] INFO: Processing datapack 0
[10:30:25] INFO: Datapack 0 completed (MRR: 0.85)
[10:30:26] INFO: Processing datapack 1
[10:30:33] INFO: Datapack 1 completed (MRR: 0.78)
```

### Poll for Completion

Wait for task to complete:

```python
import time

def wait_for_completion(task_id, poll_interval=10):
    while True:
        task = task_api.get_task(task_id=task_id)

        print(f"Status: {task.status} ({task.progress}%)")

        if task.status in ["completed", "failed", "cancelled"]:
            return task

        time.sleep(poll_interval)

# Wait for task
task = wait_for_completion("task-abc123")
print(f"Final status: {task.status}")
```

## Using the CLI

The rcabench-platform CLI provides monitoring commands:

```bash
# Get task status
./main.py trace task-abc123

# Stream events (follow mode)
./main.py trace task-abc123 --follow

# Get summary
./main.py task-status task-abc123
```

## Task Status Values

- `pending`: Queued, waiting for resources
- `scheduled`: Kubernetes job created
- `running`: Algorithm executing
- `completed`: All datapacks processed successfully
- `failed`: Execution failed (check logs)
- `cancelled`: Task cancelled by user
- `timeout`: Execution exceeded timeout limit

## Progress Tracking

### Overall Progress

```python
task = task_api.get_task(task_id="task-abc123")

print(f"Progress: {task.progress}%")
print(f"Completed: {task.completed_datapacks}/{task.total_datapacks}")
print(f"Failed: {task.failed_datapacks}")
print(f"Remaining: {task.total_datapacks - task.completed_datapacks}")
```

### Per-Datapack Status

```python
# Get detailed datapack status
datapacks = task_api.get_datapack_status(task_id="task-abc123")

for dp in datapacks:
    print(f"Datapack {dp.id}: {dp.status} (MRR: {dp.mrr:.3f})")
```

Example output:

```
Datapack 0: completed (MRR: 0.850)
Datapack 1: completed (MRR: 0.780)
Datapack 2: running (MRR: -)
Datapack 3: pending (MRR: -)
```

## Viewing Logs

### Algorithm Logs

View stdout/stderr from your algorithm:

```python
# Get logs for specific datapack
logs = task_api.get_logs(
    task_id="task-abc123",
    datapack_id=0
)

print(logs.stdout)
print(logs.stderr)
```

### Kubernetes Logs

Access raw Kubernetes pod logs:

```bash
# Using kubectl
kubectl logs -n aegislab job/task-abc123-dp-0

# Follow logs
kubectl logs -n aegislab job/task-abc123-dp-0 -f

# Get logs from all pods
kubectl logs -n aegislab -l task-id=task-abc123
```

## Real-Time Dashboard

Access the web dashboard for visual monitoring:

```
https://aegislab.io/dashboard/tasks/task-abc123
```

The dashboard shows:
- Overall progress bar
- Per-datapack status
- Real-time logs
- Resource usage (CPU, memory)
- Execution timeline

## Notifications

### Email Notifications

Configure email notifications for task completion:

```python
request = DtoSubmitExecutionReq(
    algorithm_name="my-rca",
    dataset="trainticket-pandora-v1",
    datapack_start=0,
    datapack_end=99,

    # Enable notifications
    notify_on_completion=True,
    notify_on_failure=True,
    notification_email="user@example.com"
)
```

### Webhook Notifications

Set up webhook for programmatic notifications:

```python
request = DtoSubmitExecutionReq(
    algorithm_name="my-rca",
    dataset="trainticket-pandora-v1",
    datapack_start=0,
    datapack_end=99,

    # Webhook URL
    webhook_url="https://myapp.com/webhooks/aegislab",
    webhook_events=["completed", "failed"]
)
```

Webhook payload:

```json
{
  "event": "completed",
  "task_id": "task-abc123",
  "status": "completed",
  "algorithm": "my-rca",
  "dataset": "trainticket-pandora-v1",
  "metrics": {
    "mean_mrr": 0.823,
    "mean_top1": 0.712
  },
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

## Resource Monitoring

### CPU and Memory Usage

```python
# Get resource metrics
metrics = task_api.get_resource_metrics(task_id="task-abc123")

print(f"CPU usage: {metrics.cpu_usage_percent}%")
print(f"Memory usage: {metrics.memory_usage_mb} MB")
print(f"Peak memory: {metrics.peak_memory_mb} MB")
```

### Execution Time

```python
task = task_api.get_task(task_id="task-abc123")

print(f"Started: {task.started_at}")
print(f"Completed: {task.completed_at}")
print(f"Duration: {task.elapsed_seconds}s")
print(f"Avg per datapack: {task.avg_datapack_duration}s")
```

## Troubleshooting

### Task Stuck in Pending

```
Status: pending (0%)
```

**Possible causes**:
- Insufficient cluster resources
- High queue backlog
- Resource quota exceeded

**Solution**: Check cluster status or reduce resource limits.

### Task Failed

```
Status: failed
Error: Algorithm exited with code 1
```

**Solution**: Check logs for error details:

```python
logs = task_api.get_logs(task_id="task-abc123", datapack_id=0)
print(logs.stderr)
```

### Slow Execution

```
Progress: 10% after 30 minutes
```

**Solution**:
- Check resource limits (CPU/memory)
- Profile algorithm locally
- Optimize data processing

### Connection Timeout

```
Error: Connection timeout while streaming events
```

**Solution**: Increase timeout or use polling instead of streaming:

```python
# Use polling instead
task = task_api.get_task(task_id="task-abc123")
```

## Best Practices

1. **Poll, don't spam**: Use reasonable poll intervals (10-30 seconds)
2. **Stream for real-time**: Use streaming for active monitoring
3. **Check logs on failure**: Always inspect logs when tasks fail
4. **Monitor resources**: Track CPU/memory to optimize limits
5. **Set up notifications**: Use webhooks for long-running tasks
6. **Cancel stuck tasks**: Don't let failed tasks consume resources

## Next Steps

- [Retrieve Results](../retrieve-results): Download evaluation metrics
- [Troubleshooting](../../troubleshooting): Debug common issues
