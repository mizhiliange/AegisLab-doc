---
title: Retrieve Results
weight: 4
---

# Retrieve Results

Download evaluation metrics and outputs from completed remote executions.

## Prerequisites

- Completed task execution
- Task ID from submission
- AegisLab SDK installed

## Using the Python SDK

### Get Evaluation Metrics

```python
from rcabench.openapi import ApiClient, Configuration, ResultsApi

config = Configuration(host="http://10.10.10.220:8080")
client = ApiClient(config)
results_api = ResultsApi(client)

# Get results for a task
results = results_api.get_task_results(task_id="task-abc123")

print(f"Algorithm: {results.algorithm_name}")
print(f"Dataset: {results.dataset}")
print(f"Mean MRR: {results.mean_mrr:.3f}")
print(f"Mean Top-1 Accuracy: {results.mean_top1_accuracy:.3f}")
print(f"Mean Top-3 Accuracy: {results.mean_top3_accuracy:.3f}")
```

### Download Detailed Results

```python
# Download full results as JSON
results_api.download_results(
    task_id="task-abc123",
    output_path="./results/task-abc123.json"
)

# Load and analyze
import json
with open("./results/task-abc123.json") as f:
    data = json.load(f)

# Per-datapack results
for dp in data["datapack_results"]:
    print(f"Datapack {dp['id']}: MRR={dp['mrr']:.3f}")
```

### Get Per-Datapack Results

```python
# Get results for specific datapack
datapack_result = results_api.get_datapack_result(
    task_id="task-abc123",
    datapack_id=0
)

print(f"MRR: {datapack_result.mrr}")
print(f"Top-1 Accuracy: {datapack_result.top1_accuracy}")
print(f"Predicted root causes:")
for i, service in enumerate(datapack_result.ranked_services, 1):
    print(f"  {i}. {service.name} (score: {service.score:.3f})")

print(f"Ground truth: {datapack_result.ground_truth}")
```

## Using the CLI

```bash
cd rcabench-platform

# Get summary metrics
./main.py metrics task-abc123

# Download full results
./main.py download-results task-abc123 --output ./results/

# Get specific datapack
./main.py metrics task-abc123 --datapack 0
```

## Results Format

### Summary Metrics

```json
{
  "task_id": "task-abc123",
  "algorithm": "my-rca",
  "version": "v1.0.0",
  "dataset": "trainticket-pandora-v1",
  "total_datapacks": 100,
  "completed_datapacks": 100,
  "failed_datapacks": 0,
  "metrics": {
    "mean_mrr": 0.823,
    "std_mrr": 0.142,
    "mean_avg1": 0.712,
    "mean_avg3": 2.034,
    "mean_avg5": 3.156,
    "mean_top1_accuracy": 0.712,
    "mean_top3_accuracy": 0.887,
    "mean_top5_accuracy": 0.945
  },
  "execution_time": {
    "total_seconds": 1847,
    "avg_per_datapack": 18.47
  }
}
```

### Per-Datapack Results

```json
{
  "datapack_id": 0,
  "status": "completed",
  "mrr": 0.850,
  "avg1": 0.750,
  "avg3": 2.100,
  "top1_accuracy": 0.750,
  "top3_accuracy": 0.900,
  "ranked_services": [
    {"name": "ts-order-service", "score": 0.95, "rank": 1},
    {"name": "ts-payment-service", "score": 0.72, "rank": 2},
    {"name": "ts-user-service", "score": 0.43, "rank": 3}
  ],
  "ground_truth": ["ts-order-service"],
  "execution_time_seconds": 18.3
}
```

## Comparing Results

### Compare Multiple Algorithms

```python
# Get results for multiple algorithms
algorithms = ["simplerca", "my-rca", "baseline-rca"]
comparison = []

for algo in algorithms:
    task_id = task_ids[algo]  # Map of algorithm to task ID
    results = results_api.get_task_results(task_id=task_id)
    comparison.append({
        "algorithm": algo,
        "mrr": results.mean_mrr,
        "top1": results.mean_top1_accuracy,
        "top3": results.mean_top3_accuracy
    })

# Display comparison
import pandas as pd
df = pd.DataFrame(comparison)
print(df.to_string(index=False))
```

Output:

```
    algorithm    mrr   top1   top3
    simplerca  0.823  0.712  0.887
       my-rca  0.856  0.745  0.912
 baseline-rca  0.701  0.623  0.801
```

### Statistical Significance

```python
from scipy import stats

# Get per-datapack MRR values
algo1_mrrs = [dp.mrr for dp in results_api.get_all_datapack_results("task-1")]
algo2_mrrs = [dp.mrr for dp in results_api.get_all_datapack_results("task-2")]

# Paired t-test
t_stat, p_value = stats.ttest_rel(algo1_mrrs, algo2_mrrs)

print(f"t-statistic: {t_stat:.3f}")
print(f"p-value: {p_value:.4f}")

if p_value < 0.05:
    print("Difference is statistically significant")
else:
    print("Difference is not statistically significant")
```

## Exporting Results

### Export to CSV

```python
import csv

results = results_api.get_task_results(task_id="task-abc123")
datapack_results = results_api.get_all_datapack_results(task_id="task-abc123")

with open("results.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["datapack_id", "mrr", "top1_accuracy", "top3_accuracy"])

    for dp in datapack_results:
        writer.writerow([dp.id, dp.mrr, dp.top1_accuracy, dp.top3_accuracy])
```

### Export to Excel

```python
import pandas as pd

# Create DataFrame
data = []
for dp in datapack_results:
    data.append({
        "Datapack": dp.id,
        "MRR": dp.mrr,
        "Top-1": dp.top1_accuracy,
        "Top-3": dp.top3_accuracy,
        "Top-5": dp.top5_accuracy,
        "Ground Truth": ", ".join(dp.ground_truth),
        "Top Prediction": dp.ranked_services[0].name if dp.ranked_services else "N/A"
    })

df = pd.DataFrame(data)
df.to_excel("results.xlsx", index=False)
```

### Generate Report

```python
from datetime import datetime

# Generate markdown report
report = f"""# Evaluation Report

**Algorithm**: {results.algorithm_name} v{results.algorithm_version}
**Dataset**: {results.dataset}
**Date**: {datetime.now().strftime("%Y-%m-%d %H:%M:%S")}

## Summary Metrics

- **Mean MRR**: {results.mean_mrr:.3f} ± {results.std_mrr:.3f}
- **Mean Top-1 Accuracy**: {results.mean_top1_accuracy:.3f}
- **Mean Top-3 Accuracy**: {results.mean_top3_accuracy:.3f}
- **Mean Top-5 Accuracy**: {results.mean_top5_accuracy:.3f}

## Execution Details

- **Total Datapacks**: {results.total_datapacks}
- **Completed**: {results.completed_datapacks}
- **Failed**: {results.failed_datapacks}
- **Total Time**: {results.execution_time.total_seconds}s
- **Avg per Datapack**: {results.execution_time.avg_per_datapack:.2f}s

## Top Performing Datapacks

"""

# Add top 5 datapacks by MRR
sorted_dps = sorted(datapack_results, key=lambda x: x.mrr, reverse=True)[:5]
for i, dp in enumerate(sorted_dps, 1):
    report += f"{i}. Datapack {dp.id}: MRR={dp.mrr:.3f}\n"

with open("report.md", "w") as f:
    f.write(report)
```

## Visualizing Results

### MRR Distribution

```python
import matplotlib.pyplot as plt

mrrs = [dp.mrr for dp in datapack_results]

plt.figure(figsize=(10, 6))
plt.hist(mrrs, bins=20, edgecolor='black')
plt.xlabel('MRR')
plt.ylabel('Frequency')
plt.title(f'MRR Distribution - {results.algorithm_name}')
plt.axvline(results.mean_mrr, color='red', linestyle='--', label=f'Mean: {results.mean_mrr:.3f}')
plt.legend()
plt.savefig('mrr_distribution.png')
```

### Algorithm Comparison

```python
import matplotlib.pyplot as plt

algorithms = ["simplerca", "my-rca", "baseline-rca"]
mrrs = [0.823, 0.856, 0.701]
top1s = [0.712, 0.745, 0.623]

x = range(len(algorithms))
width = 0.35

fig, ax = plt.subplots(figsize=(10, 6))
ax.bar([i - width/2 for i in x], mrrs, width, label='MRR')
ax.bar([i + width/2 for i in x], top1s, width, label='Top-1 Accuracy')

ax.set_xlabel('Algorithm')
ax.set_ylabel('Score')
ax.set_title('Algorithm Comparison')
ax.set_xticks(x)
ax.set_xticklabels(algorithms)
ax.legend()

plt.savefig('algorithm_comparison.png')
```

## Downloading Algorithm Outputs

If your algorithm produces additional outputs (plots, logs, intermediate results):

```python
# Download all outputs
results_api.download_outputs(
    task_id="task-abc123",
    output_path="./outputs/"
)

# Directory structure:
# outputs/
#   0/
#     output.json
#     debug.log
#     visualization.png
#   1/
#     output.json
#     debug.log
#     visualization.png
```

## Troubleshooting

### Results Not Available

```
Error: Results not found for task task-abc123
```

**Solution**: Verify task is completed:

```python
task = task_api.get_task(task_id="task-abc123")
print(f"Status: {task.status}")
```

### Incomplete Results

```
Warning: Only 95/100 datapacks have results
```

**Solution**: Check for failed datapacks:

```python
task = task_api.get_task(task_id="task-abc123")
print(f"Failed datapacks: {task.failed_datapacks}")

# Get failure details
for dp_id in task.failed_datapack_ids:
    logs = task_api.get_logs(task_id="task-abc123", datapack_id=dp_id)
    print(f"Datapack {dp_id} error: {logs.stderr}")
```

### Download Timeout

```
Error: Timeout while downloading results
```

**Solution**: Download in chunks or increase timeout:

```python
# Download specific datapacks
for dp_id in range(0, 100, 10):
    results_api.download_datapack_results(
        task_id="task-abc123",
        datapack_start=dp_id,
        datapack_end=dp_id+9,
        output_path=f"./results/batch_{dp_id}/"
    )
```

## Best Practices

1. **Verify completion**: Always check task status before retrieving results
2. **Handle failures**: Check for failed datapacks and investigate errors
3. **Export early**: Download results promptly after completion
4. **Compare fairly**: Use same dataset and datapack range for comparisons
5. **Statistical testing**: Use proper statistical tests for significance
6. **Document results**: Generate reports with metadata and context

## Next Steps

- [Examples](../../examples): Study example algorithms
- [Troubleshooting](../../troubleshooting): Debug common issues
- [Reference](../../reference): Complete API reference
