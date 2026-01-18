---
title: Algorithm Interface
weight: 1
---

# Algorithm Interface

Complete reference for implementing RCA algorithms using the standardized interface.

## Base Class

All RCA algorithms must inherit from the `Algorithm` base class:

```python
from rcabench_platform.v2.algorithms import Algorithm, AlgorithmArgs, AlgorithmAnswer

class MyRCAAlgorithm(Algorithm):
    def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
        """
        Main entry point for the algorithm.

        Args:
            args: Input data paths and metadata

        Returns:
            AlgorithmAnswer with ranked root cause predictions
        """
        # Your algorithm implementation
        pass
```

## AlgorithmArgs

Input data structure passed to your algorithm:

```python
@dataclass
class AlgorithmArgs:
    # Required paths
    trace_path: str          # Path to traces.parquet
    ground_truth_path: str   # Path to ground_truth.parquet
    output_path: str         # Where to write results

    # Optional paths
    metric_path: Optional[str] = None   # Path to metrics.parquet
    log_path: Optional[str] = None      # Path to logs.parquet

    # Metadata
    dataset_name: str        # Name of the dataset
    datapack_id: str         # Unique identifier for this datapack
    benchmark: str           # Benchmark system (e.g., "trainticket")

    # Additional context
    fault_start_time: Optional[int] = None
    fault_end_time: Optional[int] = None
```

### Example Usage

```python
def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
    # Read trace data
    traces = pl.read_parquet(args.trace_path)

    # Read ground truth for time window
    gt = pl.read_parquet(args.ground_truth_path)
    fault_start = gt.select("fault_start_time").item()
    fault_end = gt.select("fault_end_time").item()

    # Filter to fault window
    fault_traces = traces.filter(
        (pl.col("start_time") >= fault_start) &
        (pl.col("start_time") <= fault_end)
    )

    # Optional: read metrics if available
    if args.metric_path:
        metrics = pl.read_parquet(args.metric_path)
```

## AlgorithmAnswer

Output structure your algorithm must return:

```python
@dataclass
class AlgorithmAnswer:
    # Required: ranked list of suspected root causes
    ranked_services: List[str]

    # Optional: confidence scores for each prediction
    scores: Optional[List[float]] = None

    # Optional: additional metadata
    metadata: Optional[Dict[str, Any]] = None
```

### Example Usage

```python
def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
    # Your algorithm logic
    suspected_services = ["ts-order-service", "ts-payment-service"]
    confidence_scores = [0.95, 0.72]

    return AlgorithmAnswer(
        ranked_services=suspected_services,
        scores=confidence_scores,
        metadata={
            "algorithm_version": "1.0.0",
            "processing_time_ms": 123,
            "features_used": ["error_rate", "latency"]
        }
    )
```

## Complete Example

Here's a complete minimal algorithm:

```python
from rcabench_platform.v2.algorithms import Algorithm, AlgorithmArgs, AlgorithmAnswer
import polars as pl

class ErrorCountRCA(Algorithm):
    """Simple RCA algorithm that ranks services by error count."""

    def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
        try:
            # Read trace data
            traces = pl.read_parquet(args.trace_path)

            # Get fault time window
            gt = pl.read_parquet(args.ground_truth_path)
            fault_start = gt.select("fault_start_time").item()
            fault_end = gt.select("fault_end_time").item()

            # Filter to fault window
            fault_traces = traces.filter(
                (pl.col("start_time") >= fault_start) &
                (pl.col("start_time") <= fault_end)
            )

            # Count errors by service
            error_counts = (
                fault_traces
                .filter(pl.col("status_code") == "ERROR")
                .group_by("service_name")
                .agg(pl.count().alias("error_count"))
                .sort("error_count", descending=True)
            )

            # Extract ranked services
            ranked_services = error_counts["service_name"].to_list()
            scores = error_counts["error_count"].to_list()

            return AlgorithmAnswer(
                ranked_services=ranked_services,
                scores=scores,
                metadata={"method": "error_count"}
            )

        except Exception as e:
            # Return empty result on error
            print(f"Error in algorithm: {e}")
            return AlgorithmAnswer(ranked_services=[])
```

## Algorithm Registration

Register your algorithm in the global registry:

```python
# In rcabench_platform/v2/cli/main.py
from rcabench_platform.v2.algorithms.my_algorithm import MyRCAAlgorithm

def register_builtin_algorithms():
    """Register all built-in algorithms."""
    registry = AlgorithmRegistry.get_instance()

    # Register your algorithm
    registry.register("my-rca", MyRCAAlgorithm)
```

## Best Practices

### Error Handling

Always handle errors gracefully:

```python
def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
    try:
        # Your algorithm logic
        pass
    except FileNotFoundError as e:
        print(f"Data file not found: {e}")
        return AlgorithmAnswer(ranked_services=[])
    except Exception as e:
        print(f"Unexpected error: {e}")
        return AlgorithmAnswer(ranked_services=[])
```

### Data Validation

Validate input data before processing:

```python
def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
    traces = pl.read_parquet(args.trace_path)

    # Check if data is empty
    if traces.is_empty():
        return AlgorithmAnswer(ranked_services=[])

    # Check required columns exist
    required_cols = ["service_name", "status_code", "start_time"]
    if not all(col in traces.columns for col in required_cols):
        print(f"Missing required columns")
        return AlgorithmAnswer(ranked_services=[])
```

### Memory Efficiency

Use lazy evaluation for large datasets:

```python
def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
    # Use scan instead of read for lazy evaluation
    traces = pl.scan_parquet(args.trace_path)

    # Build query without executing
    result = (
        traces
        .filter(pl.col("status_code") == "ERROR")
        .group_by("service_name")
        .agg(pl.count())
        .collect()  # Execute only when needed
    )
```

### Logging

Use print statements for logging (captured by platform):

```python
def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
    print(f"Processing dataset: {args.dataset_name}")
    print(f"Datapack ID: {args.datapack_id}")

    # Your algorithm logic

    print(f"Found {len(ranked_services)} suspected services")
    return AlgorithmAnswer(ranked_services=ranked_services)
```

## Advanced Features

### Using Multiple Data Sources

Combine traces, metrics, and logs:

```python
def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
    # Read all available data
    traces = pl.read_parquet(args.trace_path)

    metrics = None
    if args.metric_path:
        metrics = pl.read_parquet(args.metric_path)

    logs = None
    if args.log_path:
        logs = pl.read_parquet(args.log_path)

    # Combine insights from all sources
    trace_suspects = analyze_traces(traces)

    if metrics:
        metric_suspects = analyze_metrics(metrics)
        # Merge results

    if logs:
        log_suspects = analyze_logs(logs)
        # Merge results
```

### Configurable Parameters

Support configuration via metadata:

```python
class ConfigurableRCA(Algorithm):
    def __init__(self, threshold: float = 0.5, window_size: int = 60):
        self.threshold = threshold
        self.window_size = window_size

    def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
        # Use self.threshold and self.window_size
        pass
```

### Caching Intermediate Results

Cache expensive computations:

```python
from functools import lru_cache

class CachedRCA(Algorithm):
    @lru_cache(maxsize=128)
    def compute_service_graph(self, trace_data_hash: str):
        # Expensive computation
        pass

    def __call__(self, args: AlgorithmArgs) -> AlgorithmAnswer:
        traces = pl.read_parquet(args.trace_path)

        # Create hash of trace data
        data_hash = hash(traces.to_pandas().to_json())

        # Use cached result if available
        graph = self.compute_service_graph(data_hash)
```

## Testing Your Algorithm

Test locally before submission:

```python
# test_my_algorithm.py
from rcabench_platform.v2.algorithms import AlgorithmArgs
from my_algorithm import MyRCAAlgorithm

def test_algorithm():
    args = AlgorithmArgs(
        trace_path="data/test/traces.parquet",
        ground_truth_path="data/test/ground_truth.parquet",
        output_path="output/",
        dataset_name="test",
        datapack_id="0",
        benchmark="trainticket"
    )

    algo = MyRCAAlgorithm()
    result = algo(args)

    assert len(result.ranked_services) > 0
    print(f"Test passed! Predicted: {result.ranked_services}")

if __name__ == "__main__":
    test_algorithm()
```

## Next Steps

- [Data Formats](data-formats): Understanding input data schemas
- [Containerization](containerization): Package your algorithm for deployment
- [Local Evaluation](../local-evaluation): Test on real datasets
