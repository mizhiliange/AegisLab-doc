---
title: Quickstart
weight: 1
---

# Algorithm Developer Quickstart

Get started with RCA algorithm development in under 15 minutes.

## Prerequisites

- Python 3.10 or higher
- Git
- Basic understanding of distributed tracing concepts

## Step 1: Install rcabench-platform

Clone and install the rcabench-platform:

```bash
git clone https://github.com/OperationsPAI/rcabench-platform.git
cd rcabench-platform
uv sync --all-extras
```

This installs the platform with all dependencies including algorithm development tools.

## Step 2: Verify Installation

Test the installation by running the self-test:

```bash
./main.py self test
```

You should see output indicating successful tests for the core platform components.

## Step 3: Download Sample Dataset

For this quickstart, we'll use a small sample dataset. Create a data directory and download the sample:

```bash
mkdir -p data/sample
# Sample dataset will be provided - for now, use existing datasets
# If you have JuiceFS access:
# sudo juicefs mount redis://10.10.10.119:6379/1 /mnt/jfs -d
# ln -s /mnt/jfs/rcabench-platform-v2 data/
```

## Step 4: Run Your First Algorithm

Run the built-in `simplerca` algorithm on a sample dataset:

```bash
./main.py eval single simplerca trainticket-pandora-v1 0
```

This command:
- Evaluates the `simplerca` algorithm
- Uses the `trainticket-pandora-v1` dataset
- Processes datapack `0` (first fault injection scenario)

## Understanding the Output

The evaluation will output metrics like:

```
MRR: 0.85
Avg@1: 0.75
Avg@3: 2.1
Top-1 Accuracy: 0.75
Top-3 Accuracy: 0.90
```

**Metrics explained:**
- **MRR (Mean Reciprocal Rank)**: Average of 1/rank for the first correct answer (higher is better, max 1.0)
- **Avg@k**: Average number of correct answers in top-k predictions
- **Top-k Accuracy**: Percentage of cases where at least one correct answer appears in top-k

## Step 5: Explore the Algorithm Code

Look at the `simplerca` implementation to understand the algorithm interface:

```bash
cat src/rcabench_platform/v2/algorithms/simplerca.py
```

Key components:
- `Algorithm` base class with `__call__` method
- `AlgorithmArgs` input containing traces, metrics, and metadata
- `AlgorithmAnswer` output with ranked root cause predictions

## Next Steps

Now that you've run your first algorithm:

1. **Understand the data format**: Read [Data Formats](../development-guide/data-formats) to learn about input parquet files
2. **Implement your algorithm**: Follow the [Development Guide](../development-guide) to create your own RCA algorithm
3. **Evaluate locally**: Use [Local Evaluation](../local-evaluation) to test on multiple datasets
4. **Submit remotely**: Package and submit via [Remote Evaluation](../remote-evaluation)

## Troubleshooting

**Dataset not found**: Ensure you have access to datasets via JuiceFS mount or download sample datasets.

**Import errors**: Verify installation with `uv sync --all-extras` and check Python version (3.10+).

**Permission errors**: Check file permissions on the data directory and ensure JuiceFS is mounted correctly.

For more help, see [Troubleshooting](../troubleshooting).
