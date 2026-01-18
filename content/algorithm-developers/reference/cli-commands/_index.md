---
title: CLI Commands
weight: 1
---

# CLI Commands Reference

Complete reference for rcabench-platform command-line interface.

## Main Command

```bash
./main.py [command] [options]
```

## Evaluation Commands

### eval single

Evaluate one algorithm on a single datapack.

```bash
./main.py eval single <algorithm> <dataset> <datapack_id>
```

**Arguments:**
- `algorithm`: Algorithm name (must be registered)
- `dataset`: Dataset name (e.g., `trainticket-pandora-v1`)
- `datapack_id`: Datapack ID (integer)

**Example:**
```bash
./main.py eval single simplerca trainticket-pandora-v1 0
```

### eval batch

Evaluate one algorithm on multiple datapacks.

```bash
./main.py eval batch <algorithm> <dataset> <start_id> <end_id>
```

**Arguments:**
- `algorithm`: Algorithm name
- `dataset`: Dataset name
- `start_id`: Starting datapack ID (inclusive)
- `end_id`: Ending datapack ID (inclusive)

**Options:**
- `--parallel N`: Run N datapacks in parallel (default: 1)
- `--output DIR`: Output directory (default: `output/`)

**Example:**
```bash
./main.py eval batch simplerca trainticket-pandora-v1 0 99 --parallel 4
```

## Remote Execution Commands

### submit-execution

Submit algorithm for remote execution.

```bash
./main.py submit-execution <algorithm> <dataset> <start_id> <end_id> [options]
```

**Options:**
- `--version VERSION`: Algorithm version (default: `latest`)
- `--cpu LIMIT`: CPU limit (e.g., `2000m`)
- `--memory LIMIT`: Memory limit (e.g., `4Gi`)
- `--timeout SECONDS`: Timeout in seconds
- `--description TEXT`: Task description

**Example:**
```bash
./main.py submit-execution my-rca trainticket-pandora-v1 0 99 \
    --cpu 2000m \
    --memory 4Gi \
    --description "Baseline evaluation"
```

### trace

Monitor task execution progress.

```bash
./main.py trace <task_id> [options]
```

**Options:**
- `--follow`: Stream events in real-time
- `--level LEVEL`: Filter by log level (INFO, WARNING, ERROR)

**Example:**
```bash
./main.py trace task-abc123 --follow
```

### metrics

Retrieve evaluation metrics for a task.

```bash
./main.py metrics <task_id> [options]
```

**Options:**
- `--datapack ID`: Get metrics for specific datapack
- `--format FORMAT`: Output format (json, csv, table)

**Example:**
```bash
./main.py metrics task-abc123 --format table
```

### download-results

Download full results from remote execution.

```bash
./main.py download-results <task_id> --output <directory>
```

**Example:**
```bash
./main.py download-results task-abc123 --output ./results/
```

## Algorithm Management Commands

### upload-algorithm-harbor

Build and upload algorithm to Harbor registry.

```bash
./main.py upload-algorithm-harbor <algorithm> <version>
```

**Arguments:**
- `algorithm`: Algorithm name
- `version`: Version tag (e.g., `v1.0.0`)

**Example:**
```bash
./main.py upload-algorithm-harbor my-rca v1.0.0
```

### list-algorithms

List registered algorithms.

```bash
./main.py list-algorithms
```

**Output:**
```
Available algorithms:
  - simplerca (v1.0.0)
  - my-rca (v1.0.0)
  - baseline-rca (v0.9.0)
```

## Dataset Commands

### list-datasets

List available datasets.

```bash
./main.py list-datasets [options]
```

**Options:**
- `--benchmark NAME`: Filter by benchmark system
- `--format FORMAT`: Output format (table, json)

**Example:**
```bash
./main.py list-datasets --benchmark trainticket
```

### dataset-info

Get detailed information about a dataset.

```bash
./main.py dataset-info <dataset>
```

**Example:**
```bash
./main.py dataset-info trainticket-pandora-v1
```

**Output:**
```
Dataset: trainticket-pandora-v1
Benchmark: trainticket
Total datapacks: 100
Size: 15.2 GB
Created: 2025-12-01
Description: Genetic algorithm-generated fault scenarios
```

## Utility Commands

### self test

Run platform self-tests.

```bash
./main.py self test
```

Verifies:
- Python dependencies
- Data directory access
- Algorithm registry
- Configuration files

### version

Display version information.

```bash
./main.py version
```

### config

Display or modify configuration.

```bash
./main.py config [get|set] <key> [value]
```

**Examples:**
```bash
# Get configuration value
./main.py config get api.endpoint

# Set configuration value
./main.py config set api.endpoint http://10.10.10.220:8080
```

## Environment Variables

### Required

- `ENV_MODE`: Environment mode (`debug`, `dev`, `prod`)

### Optional

- `AEGISLAB_API_URL`: AegisLab API endpoint
- `AEGISLAB_TOKEN`: API authentication token
- `DATA_DIR`: Data directory path (default: `./data`)
- `OUTPUT_DIR`: Output directory path (default: `./output`)

## Configuration File

Configuration is stored in `src/rcabench_platform/v2/config.py`.

Example configuration:

```python
class Config:
    api_endpoint = "http://10.10.10.220:8080"
    data_dir = "./data"
    output_dir = "./output"
    default_cpu_limit = "2000m"
    default_memory_limit = "4Gi"
    default_timeout = 1800
```

## Exit Codes

- `0`: Success
- `1`: General error
- `2`: Invalid arguments
- `3`: Dataset not found
- `4`: Algorithm not found
- `5`: API error
- `6`: Timeout

## Common Usage Patterns

### Local Development Workflow

```bash
# Test on single datapack
./main.py eval single my-rca trainticket-pandora-v1 0

# Test on small batch
./main.py eval batch my-rca trainticket-pandora-v1 0 9

# Full local evaluation
./main.py eval batch my-rca trainticket-pandora-v1 0 99 --parallel 4
```

### Remote Execution Workflow

```bash
# Upload algorithm
./main.py upload-algorithm-harbor my-rca v1.0.0

# Submit execution
./main.py submit-execution my-rca trainticket-pandora-v1 0 99

# Monitor progress
./main.py trace task-abc123 --follow

# Get results
./main.py metrics task-abc123
./main.py download-results task-abc123 --output ./results/
```

### Comparison Workflow

```bash
# Evaluate multiple algorithms
for algo in simplerca my-rca baseline-rca; do
    ./main.py eval batch $algo trainticket-pandora-v1 0 99
done

# Compare results
./main.py compare-results simplerca my-rca baseline-rca
```

## Troubleshooting

### Command Not Found

```
bash: ./main.py: No such file or directory
```

**Solution**: Ensure you're in the rcabench-platform directory.

### Permission Denied

```
bash: ./main.py: Permission denied
```

**Solution**: Make the script executable:
```bash
chmod +x main.py
```

### Import Errors

```
ModuleNotFoundError: No module named 'polars'
```

**Solution**: Install dependencies:
```bash
uv sync --all-extras
```

## See Also

- [SDK API](../sdk-api): Python SDK reference
- [Local Evaluation](../../local-evaluation): Evaluation guide
- [Remote Evaluation](../../remote-evaluation): Remote execution guide
