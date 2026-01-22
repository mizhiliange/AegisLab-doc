---
title: Genetic Algorithms
weight: 2
---

Use Pandora's genetic algorithm for intelligent fault injection scheduling.

## Overview

Pandora uses genetic algorithms to evolve fault injection scenarios that:
- Maximize SLO violations (high error rates, latency)
- Maximize diagnosis difficulty (low MRR for RCA algorithms)
- Generate diverse and challenging datasets

## How It Works

### Genetic Algorithm Basics

1. **Population**: Set of fault injection scenarios (individuals)
2. **Fitness**: Score based on SLO violations + diagnosis difficulty
3. **Selection**: Choose best individuals for breeding
4. **Crossover**: Combine two individuals to create offspring
5. **Mutation**: Random changes to introduce diversity
6. **Evolution**: Repeat for multiple generations

### Pandora Architecture

```
Bootstrap → Generate → Evaluate → Select → Breed → Mutate → Repeat
    ↓           ↓          ↓          ↓        ↓        ↓
  Random    Submit to   Calculate   Choose   Combine  Random
  samples   AegisLab    fitness     best     parents  changes
```

## Configuration

### Basic Configuration

```toml
# config/config.toml

[algorithm]
population_size = 50
generations = 20
mutation_rate = 0.1
crossover_rate = 0.8

[algorithm.fitness]
slo_weight = 0.5
diagnosis_weight = 0.5
timeout = 300

[algorithm.thread]
max_workers = 10
min_task_pool_size = 20

[benchmark]
name = "trainticket"
namespace = "ts"
gateway_url = "${TRAINTICKET_GATEWAY_URL}"  # Default: http://10.10.10.220:30080

[aegislab]
api_url = "${AEGISLAB_API_URL}"  # Default: http://10.10.10.220:32080
api_token = "your-token"
```

### Fitness Configuration

Control what makes a "good" fault scenario:

```toml
[algorithm.fitness]
# SLO violation weight (0-1)
slo_weight = 0.5

# Diagnosis difficulty weight (0-1)
diagnosis_weight = 0.5

# Minimum error rate to consider valid
min_error_rate = 0.1

# Timeout for fitness evaluation (seconds)
timeout = 300

# RCA algorithms to test against
rca_algorithms = ["simplerca", "microcause", "cloudranger"]
```

## Running Pandora

### Bootstrap Phase

Generate initial random population:

```bash
cd Pandora

# Bootstrap with 100 random samples
make bootstrap ENV=prod RANDOM_COUNT=100

# Or use Python directly
uv run python -m cli.bootstrap \
    --config config/config.toml \
    --count 100 \
    --output bootstrap_results.json
```

This creates initial population by:
1. Randomly selecting fault types
2. Randomly selecting target services
3. Randomly selecting fault parameters
4. Submitting to AegisLab for evaluation

### Generation Phase

Evolve population using genetic algorithm:

```bash
# Run genetic algorithm
make generate CONFIG_PATH=config/config.toml

# Or use Python directly
uv run python -m cli.generate \
    --config config/config.toml \
    --generations 20 \
    --output generation_results.json
```

### Monitoring Progress

Monitor evolution progress:

```bash
# Check status via Flask endpoint
curl http://localhost:8080/status

# View WandB dashboard
# Navigate to: https://wandb.ai/your-project/pandora

# Check logs
tail -f logs/pandora.log
```

## Individual Representation

### Fault Scenario Structure

Each individual represents a fault injection scenario:

```python
{
    "handler_nodes": [
        {
            "handler": "network_delay",
            "params": {
                "delay": "100ms",
                "target_service": "ts-order-service",
                "target_namespace": "ts"
            }
        }
    ],
    "duration": 60,
    "fitness": {
        "slo_score": 0.85,
        "diagnosis_score": 0.42,
        "total": 0.635
    }
}
```

### Genes

Genes that can be evolved:
- **Fault type**: network_delay, pod_failure, memory_pressure, etc.
- **Target service**: Which service to inject fault into
- **Fault parameters**: delay amount, memory limit, CPU limit, etc.
- **Duration**: How long fault lasts

## Fitness Evaluation

### SLO Score

Measures impact on system performance:

```python
def calculate_slo_score(traces, metrics):
    """Calculate SLO violation score."""

    # Error rate
    error_rate = count_errors(traces) / total_requests(traces)

    # Latency increase
    baseline_latency = get_baseline_latency(metrics)
    fault_latency = get_fault_latency(metrics)
    latency_increase = (fault_latency - baseline_latency) / baseline_latency

    # Success rate decrease
    baseline_success = get_baseline_success_rate(metrics)
    fault_success = get_fault_success_rate(metrics)
    success_decrease = (baseline_success - fault_success) / baseline_success

    # Combine metrics
    slo_score = (
        0.4 * error_rate +
        0.3 * latency_increase +
        0.3 * success_decrease
    )

    return min(slo_score, 1.0)
```

### Diagnosis Score

Measures difficulty for RCA algorithms:

```python
def calculate_diagnosis_score(individual, rca_results):
    """Calculate diagnosis difficulty score."""

    mrr_scores = []

    for algo_name, result in rca_results.items():
        # Get MRR for this algorithm
        mrr = result.get("mrr", 0.0)
        mrr_scores.append(mrr)

    # Lower MRR = harder to diagnose = higher score
    avg_mrr = sum(mrr_scores) / len(mrr_scores)
    diagnosis_score = 1.0 - avg_mrr

    return diagnosis_score
```

### Combined Fitness

```python
def calculate_fitness(individual, config):
    """Calculate total fitness score."""

    slo_score = calculate_slo_score(individual.traces, individual.metrics)
    diagnosis_score = calculate_diagnosis_score(individual, individual.rca_results)

    # Weighted combination
    fitness = (
        config.slo_weight * slo_score +
        config.diagnosis_weight * diagnosis_score
    )

    return fitness
```

## Genetic Operators

### Selection

Tournament selection chooses parents:

```python
def tournament_selection(population, tournament_size=3):
    """Select individual using tournament selection."""

    # Randomly select tournament_size individuals
    tournament = random.sample(population, tournament_size)

    # Return the one with highest fitness
    return max(tournament, key=lambda x: x.fitness)
```

### Crossover

Combine two parents to create offspring:

```python
def crossover(parent1, parent2):
    """Single-point crossover."""

    # Choose crossover point
    point = random.randint(0, len(parent1.genes))

    # Create offspring
    offspring1 = parent1.genes[:point] + parent2.genes[point:]
    offspring2 = parent2.genes[:point] + parent1.genes[point:]

    return offspring1, offspring2
```

### Mutation

Random changes to introduce diversity:

```python
def mutate(individual, mutation_rate=0.1):
    """Mutate individual genes."""

    if random.random() < mutation_rate:
        # Choose random gene to mutate
        gene_idx = random.randint(0, len(individual.genes) - 1)

        # Mutate based on gene type
        if gene_idx == 0:  # Fault type
            individual.genes[0] = random.choice(FAULT_TYPES)
        elif gene_idx == 1:  # Target service
            individual.genes[1] = random.choice(SERVICES)
        elif gene_idx == 2:  # Fault parameter
            individual.genes[2] = mutate_parameter(individual.genes[2])

    return individual
```

## Advanced Configuration

### Multi-Objective Optimization

Optimize for multiple objectives:

```toml
[algorithm.fitness]
# Multiple objectives with weights
objectives = [
    { name = "error_rate", weight = 0.3 },
    { name = "latency_increase", weight = 0.2 },
    { name = "diagnosis_difficulty", weight = 0.3 },
    { name = "fault_diversity", weight = 0.2 }
]
```

### Adaptive Mutation

Adjust mutation rate based on progress:

```toml
[algorithm.mutation]
initial_rate = 0.2
final_rate = 0.05
adaptive = true
```

### Elitism

Preserve best individuals:

```toml
[algorithm.selection]
elitism = true
elite_size = 5  # Keep top 5 individuals
```

## Results Analysis

### View Best Individuals

```python
# Load results
import json

with open("generation_results.json") as f:
    results = json.load(f)

# Get best individuals
best = sorted(results["individuals"], key=lambda x: x["fitness"], reverse=True)[:10]

for i, individual in enumerate(best, 1):
    print(f"{i}. Fitness: {individual['fitness']:.3f}")
    print(f"   Fault: {individual['handler_nodes'][0]['handler']}")
    print(f"   Target: {individual['handler_nodes'][0]['params']['target_service']}")
    print()
```

### Visualize Evolution

```python
import matplotlib.pyplot as plt

# Plot fitness over generations
generations = [g["generation"] for g in results["history"]]
avg_fitness = [g["avg_fitness"] for g in results["history"]]
max_fitness = [g["max_fitness"] for g in results["history"]]

plt.figure(figsize=(10, 6))
plt.plot(generations, avg_fitness, label="Average Fitness")
plt.plot(generations, max_fitness, label="Max Fitness")
plt.xlabel("Generation")
plt.ylabel("Fitness")
plt.title("Fitness Evolution")
plt.legend()
plt.savefig("fitness_evolution.png")
```

### Export Dataset

Export evolved scenarios as dataset:

```bash
# Export best individuals
uv run python -m cli.export \
    --input generation_results.json \
    --output trainticket-pandora-v2 \
    --top 100
```

## Best Practices

1. **Start with bootstrap**: Generate diverse initial population
2. **Tune fitness weights**: Balance SLO violations vs diagnosis difficulty
3. **Monitor convergence**: Stop if fitness plateaus
4. **Use elitism**: Preserve best individuals
5. **Increase diversity**: Use higher mutation rate if population converges too quickly
6. **Validate results**: Manually inspect top individuals

## Troubleshooting

### Population Converges Too Quickly

**Solution**: Increase mutation rate or population size:

```toml
[algorithm]
population_size = 100  # Increase from 50
mutation_rate = 0.2    # Increase from 0.1
```

### Low Fitness Scores

**Solution**: Check if faults are being applied correctly:

```bash
# Check AegisLab logs
kubectl logs -n aegislab aegislab-consumer-0

# Verify Chaos Mesh
kubectl get networkchaos -n ts
```

### Slow Evolution

**Solution**: Increase parallelism:

```toml
[algorithm.thread]
max_workers = 20       # Increase from 10
min_task_pool_size = 40  # Increase from 20
```

## See Also

- [Fault Injection Guide](../../fault-injection-guide): Basic fault injection
- [Custom Benchmarks](../custom-benchmarks): Using custom systems
- [Chaos Mesh Integration](../chaos-mesh-integration): Chaos engineering details
