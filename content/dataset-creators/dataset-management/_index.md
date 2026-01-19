---
title: Dataset Management
weight: 4
---

# Dataset Management

Retrieve, organize, and share datasets created through fault injection.

## Sections

- [Retrieve Datasets](retrieve-datasets): Download and access datasets
- [Dataset Structure](dataset-structure): Understanding dataset organization
- [Quality Checks](quality-checks): Validating dataset quality
- [Sharing Datasets](sharing-datasets): Making datasets available to others

## Overview

After completing fault injection and data collection, you can:
1. Retrieve datasets via API or JuiceFS
2. Validate data quality
3. Organize datasets for algorithm evaluation
4. Share datasets with the research community

## Quick Start

```python
from rcabench.openapi import ApiClient, Configuration, DatasetApi

config = Configuration(host="${AEGISLAB_API_URL}")  # Default: http://10.10.10.220:8080
client = ApiClient(config)
dataset_api = DatasetApi(client)

# List datasets
datasets = dataset_api.list_datasets(benchmark="trainticket")

# Download dataset
dataset_api.download_dataset(
    dataset_id="trainticket-custom-001",
    output_path="./data/"
)
```

## See Also

- [Monitoring & Collection](../monitoring-collection): Track data collection
- [Algorithm Developers](../../algorithm-developers): Using datasets for evaluation
