---
title: Containerization
weight: 3
---

# Containerization

Package your RCA algorithm as a Docker container for remote execution on AegisLab.

## Overview

Containerization allows your algorithm to:
- Run on AegisLab's Kubernetes cluster
- Execute in isolated environments
- Use specific dependencies and versions
- Scale across multiple nodes

## Prerequisites

- Docker or Podman installed
- Your algorithm tested locally
- Access to container registry (Docker Hub, Harbor, etc.)

## Container Structure

Your algorithm container must include:

```
my-rca-algorithm/
├── Dockerfile              # Container definition
├── entrypoint.sh          # Execution script
├── info.toml              # Algorithm metadata
├── requirements.txt       # Python dependencies
└── algorithm.py           # Your algorithm code
```

## Creating the Dockerfile

### Basic Dockerfile

```dockerfile
FROM python:3.10-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy algorithm code
COPY algorithm.py .
COPY entrypoint.sh .

# Make entrypoint executable
RUN chmod +x entrypoint.sh

# Set entrypoint
ENTRYPOINT ["/app/entrypoint.sh"]
```

### With rcabench-platform

```dockerfile
FROM python:3.10-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    && rm -rf /var/lib/apt/lists/*

# Install rcabench-platform
RUN pip install --no-cache-dir git+https://github.com/OperationsPAI/rcabench-platform.git

# Copy algorithm code
COPY algorithm.py /app/
COPY entrypoint.sh /app/

RUN chmod +x /app/entrypoint.sh

ENTRYPOINT ["/app/entrypoint.sh"]
```

## Creating the Entrypoint Script

The entrypoint script receives arguments from AegisLab and executes your algorithm:

```bash
#!/bin/bash
set -e

# Parse arguments
TRACE_PATH=$1
GROUND_TRUTH_PATH=$2
OUTPUT_PATH=$3
METRIC_PATH=${4:-""}
LOG_PATH=${5:-""}

# Run algorithm
python3 /app/algorithm.py \
    --trace-path "$TRACE_PATH" \
    --ground-truth-path "$GROUND_TRUTH_PATH" \
    --output-path "$OUTPUT_PATH" \
    --metric-path "$METRIC_PATH" \
    --log-path "$LOG_PATH"
```

### With rcabench-platform CLI

```bash
#!/bin/bash
set -e

# Arguments passed by AegisLab
ALGORITHM_NAME=$1
DATASET_NAME=$2
DATAPACK_ID=$3
OUTPUT_PATH=$4

# Run using rcabench-platform CLI
cd /app/rcabench-platform
./main.py eval single "$ALGORITHM_NAME" "$DATASET_NAME" "$DATAPACK_ID" \
    --output-path "$OUTPUT_PATH"
```

## Creating info.toml

Metadata file describing your algorithm:

```toml
[algorithm]
name = "my-rca"
version = "1.0.0"
description = "My RCA algorithm using error analysis"
author = "Your Name"
email = "your.email@example.com"

[algorithm.requirements]
python = ">=3.10"
memory = "2Gi"
cpu = "1000m"

[algorithm.parameters]
# Optional: configurable parameters
threshold = 0.5
window_size = 60

[algorithm.outputs]
# Expected output format
format = "json"
schema = "AlgorithmAnswer"
```

## Algorithm Code

Your algorithm code should accept command-line arguments:

```python
# algorithm.py
import argparse
import polars as pl
from dataclasses import asdict
import json

from rcabench_platform.v2.algorithms import Algorithm, AlgorithmArgs, AlgorithmAnswer

class MyRCAAlgorithm(Algorithm):
    def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
        # Your algorithm logic
        traces = pl.read_parquet(args.trace_path)

        # Analyze traces
        error_counts = (
            traces
            .filter(pl.col("status_code") == "ERROR")
            .group_by("service_name")
            .agg(pl.count().alias("error_count"))
            .sort("error_count", descending=True)
        )

        ranked_services = error_counts["service_name"].to_list()

        return AlgorithmAnswer(
            ranked_services=ranked_services,
            metadata={"algorithm": "my-rca", "version": "1.0.0"}
        )

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--trace-path", required=True)
    parser.add_argument("--ground-truth-path", required=True)
    parser.add_argument("--output-path", required=True)
    parser.add_argument("--metric-path", default=None)
    parser.add_argument("--log-path", default=None)

    args = parser.parse_args()

    # Create AlgorithmArgs
    algo_args = AlgorithmArgs(
        trace_path=args.trace_path,
        ground_truth_path=args.ground_truth_path,
        output_path=args.output_path,
        metric_path=args.metric_path,
        log_path=args.log_path,
        dataset_name="unknown",
        datapack_id="unknown",
        benchmark="unknown"
    )

    # Run algorithm
    algorithm = MyRCAAlgorithm()
    result = algorithm(algo_args)

    # Write output
    output_file = f"{args.output_path}/result.json"
    with open(output_file, "w") as f:
        json.dump(asdict(result), f, indent=2)

    print(f"Results written to {output_file}")

if __name__ == "__main__":
    main()
```

## Building the Container

### Build Image

```bash
# Build the image
docker build -t my-rca:1.0.0 .

# Test locally
docker run --rm \
    -v $(pwd)/data:/data \
    my-rca:1.0.0 \
    /data/traces.parquet \
    /data/ground_truth.parquet \
    /data/output
```

### Tag for Registry

```bash
# Tag for Docker Hub
docker tag my-rca:1.0.0 username/my-rca:1.0.0

# Tag for Harbor
docker tag my-rca:1.0.0 harbor.example.com/project/my-rca:1.0.0
```

### Push to Registry

```bash
# Login to registry
docker login

# Push image
docker push username/my-rca:1.0.0
```

## Testing the Container

### Local Test

Create a test script:

```bash
#!/bin/bash
# test_container.sh

# Prepare test data
mkdir -p test_data/output

# Run container
docker run --rm \
    -v $(pwd)/test_data:/data \
    my-rca:1.0.0 \
    /data/traces.parquet \
    /data/ground_truth.parquet \
    /data/output \
    /data/metrics.parquet \
    /data/logs.parquet

# Check output
cat test_data/output/result.json
```

### Verify Output Format

```python
# verify_output.py
import json

with open("test_data/output/result.json") as f:
    result = json.load(f)

# Verify required fields
assert "ranked_services" in result
assert isinstance(result["ranked_services"], list)
assert len(result["ranked_services"]) > 0

print("✓ Output format is valid")
```

## Uploading to AegisLab

Use the AegisLab SDK to register your algorithm:

```python
from rcabench.openapi import ApiClient, Configuration, AlgorithmApi
from rcabench.openapi.models import DtoRegisterAlgorithmReq

config = Configuration(host="http://aegislab-api:8080")
client = ApiClient(config)
api = AlgorithmApi(client)

# Register algorithm
req = DtoRegisterAlgorithmReq(
    name="my-rca",
    version="1.0.0",
    image="username/my-rca:1.0.0",
    description="My RCA algorithm",
    metadata={
        "author": "Your Name",
        "email": "your.email@example.com"
    }
)

response = api.register_algorithm(req)
print(f"Algorithm registered with ID: {response.algorithm_id}")
```

## Best Practices

### Keep Images Small

```dockerfile
# Use slim base images
FROM python:3.10-slim

# Clean up after apt-get
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Use --no-cache-dir with pip
RUN pip install --no-cache-dir -r requirements.txt

# Use multi-stage builds for compiled dependencies
FROM python:3.10-slim as builder
RUN pip install --user package

FROM python:3.10-slim
COPY --from=builder /root/.local /root/.local
```

### Pin Dependencies

```txt
# requirements.txt
polars==0.19.0
numpy==1.24.0
scikit-learn==1.3.0
```

### Handle Errors Gracefully

```python
def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
    try:
        # Algorithm logic
        pass
    except Exception as e:
        print(f"Error: {e}", file=sys.stderr)
        # Return empty result instead of crashing
        return AlgorithmAnswer(ranked_services=[])
```

### Log Progress

```python
def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
    print(f"Loading traces from {args.trace_path}")
    traces = pl.read_parquet(args.trace_path)
    print(f"Loaded {len(traces)} traces")

    print("Analyzing error patterns...")
    # Analysis logic

    print(f"Found {len(ranked_services)} suspected services")
    return AlgorithmAnswer(ranked_services=ranked_services)
```

### Set Resource Limits

In `info.toml`:

```toml
[algorithm.requirements]
memory = "4Gi"      # Maximum memory
cpu = "2000m"       # 2 CPU cores
timeout = 300       # 5 minutes timeout
```

## Troubleshooting

### Container Fails to Start

```bash
# Check container logs
docker logs <container-id>

# Run interactively for debugging
docker run -it --entrypoint /bin/bash my-rca:1.0.0
```

### Permission Errors

```dockerfile
# Create non-root user
RUN useradd -m -u 1000 algorithm
USER algorithm
```

### Missing Dependencies

```bash
# Test dependencies in container
docker run -it my-rca:1.0.0 python -c "import polars; print(polars.__version__)"
```

### Large Image Size

```bash
# Check image size
docker images my-rca:1.0.0

# Analyze layers
docker history my-rca:1.0.0
```

## Next Steps

- [Remote Evaluation](../remote-evaluation): Submit your containerized algorithm
- [Examples](../examples): Study example containerized algorithms
- [Troubleshooting](../troubleshooting): Common containerization issues
