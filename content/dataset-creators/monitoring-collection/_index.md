---
title: Monitoring & Collection
weight: 3
---

# Monitoring & Collection

Track fault injection progress and monitor data collection quality.

## Sections

- [Track Progress](track-progress): Monitor fault injection execution
- [Observability Data](observability-data): Understanding collected traces, metrics, and logs
- [Troubleshooting](troubleshooting): Common issues during data collection

## Overview

After submitting a fault injection, you need to:
1. Monitor the injection execution
2. Track data collection progress
3. Verify data quality
4. Troubleshoot any issues

## Quick Start

```python
from rcabench.openapi import ApiClient, Configuration, TaskApi

config = Configuration(host="${AEGISLAB_API_URL}")  # Default: http://10.10.10.220:8080
client = ApiClient(config)
task_api = TaskApi(client)

# Monitor task
task = task_api.get_task(task_id="task-abc123")
print(f"Status: {task.status}")
print(f"Progress: {task.progress}%")

# Stream events
for event in task_api.stream_trace_events(task_id="task-abc123"):
    print(f"[{event.timestamp}] {event.message}")
```

## See Also

- [Using AegisLab](../using-aegislab): Submit fault injections
- [Dataset Management](../dataset-management): Retrieve collected datasets
