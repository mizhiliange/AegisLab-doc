---
title: Remote Evaluation
weight: 4
---

Submit your algorithm for remote execution on AegisLab's Kubernetes infrastructure.

## Overview

Remote evaluation allows you to:
- Run algorithms on large-scale datasets without local resources
- Execute on production-grade infrastructure
- Compare results with other algorithms
- Access shared datasets via JuiceFS
- Track execution progress and retrieve results

## Prerequisites

- Algorithm containerized and tested locally
- Docker image pushed to Harbor registry
- AegisLab SDK installed (`pip install rcabench-openapi`)
- API credentials configured

## Workflow

1. **Upload Algorithm**: Push Docker image to Harbor registry
2. **Submit Execution**: Create evaluation task via API
3. **Monitor Progress**: Track execution status and logs
4. **Retrieve Results**: Download evaluation metrics and outputs

## Quick Example

```python
from rcabench.openapi import ApiClient, Configuration, AlgorithmApi

# Configure API client (use your AEGISLAB_API_URL from .env)
config = Configuration(host="${AEGISLAB_API_URL}")  # Default: http://10.10.10.220:32080
client = ApiClient(config)
api = AlgorithmApi(client)

# Submit evaluation
response = api.submit_execution(
    algorithm_name="my-rca",
    dataset="trainticket-pandora-v1",
    datapack_range="0-99"
)

print(f"Task ID: {response.task_id}")
```

## Next Steps

- [Upload Algorithm](upload-algorithm): Push your Docker image to Harbor
- [Submit Execution](submit-execution): Create evaluation tasks
- [Monitor Progress](monitor-progress): Track execution status
- [Retrieve Results](retrieve-results): Download metrics and outputs
