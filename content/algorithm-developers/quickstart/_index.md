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

Run the built-in `random` algorithm on a sample dataset:

```bash
./main.py eval single random rcabench_filtered ts0-ts-auth-service-stress-jv8m9r
```

This command:
- Evaluates the `random` algorithm (randomly ranks all services)
- Uses the `rcabench_filtered` dataset
- Processes the specified datapack (fault injection scenario)

You should see output indicating successful execution:

```
[INFO] enter run_single
[DEBUG] enter Random.__call__
[DEBUG] found 33 service names
[DEBUG] exit Random.__call__ duration=0.173140s
[DEBUG] len(answers)=33
[DEBUG] hit: {'level': 'service', 'name': 'ts-auth-service', 'rank': 22, ...}
[INFO] exit run_single duration=0.408342s
```

## Understanding the Output

The evaluation produces two key outputs:

1. **Ranked Services** (`output.parquet`): A list of all services ranked by suspected root cause probability
2. **Performance Metrics** (`perf.parquet`): Evaluation metrics comparing predictions to ground truth

Example metrics from the random algorithm:

```
MRR: 0.045455
Avg@1: 0.0
Avg@3: 0.0
Avg@5: 0.0
AC@1: 0.0
AC@3: 0.0
AC@5: 0.0
```

**Metrics explained:**
- **MRR (Mean Reciprocal Rank)**: Average of 1/rank for the first correct answer (higher is better, max 1.0)
- **Avg@k**: Average number of correct answers in top-k predictions
- **AC@k (Top-k Accuracy)**: Percentage of cases where at least one correct answer appears in top-k

The random algorithm performs poorly (as expected), but demonstrates the evaluation framework works correctly.

## Step 5: Explore the Algorithm Code

Look at the `random` implementation to understand the algorithm interface:

```bash
cat src/rcabench_platform/v2/algorithms/random_.py
```

Key components:
- `Algorithm` base class with `__call__` method
- `AlgorithmArgs` input containing dataset name, datapack ID, and input folder path
- `AlgorithmAnswer` output with ranked root cause predictions (level, name, rank)
- The algorithm reads parquet files from `input_folder` and returns a list of ranked services

## Next Steps

Now that you've run your first algorithm:

1. **Understand the data format**: Read [Data Formats](../development-guide/data-formats) to learn about input parquet files
2. **Implement your algorithm**: Follow the [Development Guide](../development-guide) to create your own RCA algorithm
3. **Evaluate locally**: Use [Local Evaluation](../local-evaluation) to test on multiple datasets
4. **Submit remotely**: Package and submit via [Remote Evaluation](../remote-evaluation)

## Troubleshooting

### Dataset Not Found Error

**Error**: `FileNotFoundError: No such file or directory ... data/rcabench-platform-v2/data/...`

**Solution**: Ensure datasets are properly mounted and symlinked:

```bash
# Check if JuiceFS is mounted
ls /mnt/jfs/rcabench-platform-v2

# Create symlink if missing
cd rcabench-platform
mkdir -p data
ln -s /mnt/jfs/rcabench-platform-v2 data/
```

### Datapack Not Found Error

**Error**: `AssertionError: Labels for datapack '0' not found in dataset 'xxx'`

**Cause**: Datapacks use descriptive names (not numeric IDs like `0`).

**Solution**: List available datapacks and use the full name:

```bash
# List all datasets
./main.py eval show-datasets

# List datapacks in a dataset
ls data/rcabench-platform-v2/data/rcabench_filtered/

# Use the full datapack name
./main.py eval single random rcabench_filtered ts0-ts-auth-service-stress-jv8m9r
```

### Algorithm Not Supported for Dataset

**Error**: `NotImplementedError` when running algorithm on certain datasets

**Cause**: The `random` algorithm only supports datasets starting with `rcabench` or `rcaeval`.

**Solution**: Use a compatible dataset:

```bash
# These work with random algorithm
./main.py eval single random rcabench_filtered <datapack-name>
./main.py eval single random rcaeval_re2_tt <datapack-name>

# Check available datasets
./main.py eval show-datasets
```

### Import Errors

**Error**: `ModuleNotFoundError` or import errors

**Solution**: Verify installation with correct Python version:

```bash
# Check Python version (must be 3.10+)
python --version

# Reinstall dependencies
uv sync --all-extras

# Verify installation
./main.py self test
```

### Permission Errors

**Error**: Permission denied when accessing data directory

**Solution**: Check file permissions and JuiceFS mount:

```bash
# Check JuiceFS mount status
mount | grep juicefs

# Remount if necessary
sudo juicefs mount redis://10.10.10.119:6379/1 /mnt/jfs -d

# Check data directory permissions
ls -la data/
```

For more help, see [Troubleshooting](../troubleshooting).
