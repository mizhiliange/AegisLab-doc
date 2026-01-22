---
title: Custom Benchmarks
weight: 1
---

Integrate AegisLab with your own microservices applications.

## Overview

While AegisLab provides built-in support for TrainTicket, OTEL Demo, and Media Microservices, you can integrate custom benchmark systems for fault injection and RCA evaluation.

## Prerequisites

Your microservices application should have:
- Kubernetes deployment
- OpenTelemetry instrumentation
- Service mesh or network connectivity
- Observability stack (traces, metrics, logs)

## Integration Steps

### 1. Define System Metadata

Create system configuration file:

```python
# chaos-experiment/internal/yoursystem/metadata.py

class YourSystemMetadata:
    """Metadata for your custom system."""

    # Service list
    SERVICES = [
        "frontend",
        "api-gateway",
        "user-service",
        "order-service",
        "payment-service",
        "notification-service"
    ]

    # Service endpoints
    ENDPOINTS = {
        "frontend": ["GET /", "GET /products", "POST /checkout"],
        "api-gateway": ["GET /api/v1/*"],
        "user-service": ["GET /users/:id", "POST /users"],
        "order-service": ["GET /orders/:id", "POST /orders"],
        "payment-service": ["POST /payments"],
        "notification-service": ["POST /notifications"]
    }

    # Network dependencies
    DEPENDENCIES = {
        "frontend": ["api-gateway"],
        "api-gateway": ["user-service", "order-service"],
        "order-service": ["payment-service", "notification-service"],
        "payment-service": [],
        "notification-service": []
    }

    # JVM services (for JVM chaos)
    JVM_SERVICES = ["user-service", "order-service", "payment-service"]

    # Database services
    DATABASE_SERVICES = {
        "user-service": "postgres",
        "order-service": "mysql",
        "payment-service": "postgres"
    }
```

### 2. Register System

Register your system with AegisLab:

```python
# Register custom benchmark
from rcabench.openapi import ApiClient, Configuration, BenchmarkApi
from rcabench.openapi.models import DtoRegisterBenchmarkReq

config = Configuration(host="${AEGISLAB_API_URL}")  # Default: http://10.10.10.220:32080
client = ApiClient(config)
benchmark_api = BenchmarkApi(client)

request = DtoRegisterBenchmarkReq(
    name="yoursystem",
    display_name="Your System",
    description="Custom microservices application",
    namespace="yoursystem",
    service_count=6,
    metadata={
        "services": YourSystemMetadata.SERVICES,
        "endpoints": YourSystemMetadata.ENDPOINTS,
        "dependencies": YourSystemMetadata.DEPENDENCIES
    }
)

response = benchmark_api.register_benchmark(request)
print(f"Benchmark registered: {response.benchmark_id}")
```

### 3. Deploy Application

Deploy your application to Kubernetes:

```bash
# Create namespace
kubectl create namespace yoursystem

# Deploy services
kubectl apply -f deployments/ -n yoursystem

# Verify deployment
kubectl get pods -n yoursystem
```

### 4. Configure Observability

#### OpenTelemetry Instrumentation

Ensure services export traces:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  template:
    spec:
      containers:
      - name: user-service
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://otel-collector:4317"
        - name: OTEL_SERVICE_NAME
          value: "user-service"
        - name: OTEL_TRACES_SAMPLER
          value: "always_on"
```

#### Metrics Collection

Configure Prometheus scraping:

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: yoursystem-metrics
spec:
  selector:
    matchLabels:
      app: yoursystem
  endpoints:
  - port: metrics
    interval: 15s
```

### 5. Create Load Generator

Implement load generator for your system:

```python
# loadgenerator/yoursystem_load.py

import requests
import random
import time
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Configure OTEL
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)
span_processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4317"))
trace.get_tracer_provider().add_span_processor(span_processor)

class YourSystemLoadGenerator:
    def __init__(self, base_url):
        self.base_url = base_url

    def generate_load(self):
        """Generate realistic load."""
        with tracer.start_as_current_span("load_generation"):
            # Simulate user behavior
            behavior = random.choice([
                self.browse_products,
                self.create_order,
                self.check_order_status
            ])
            behavior()

    def browse_products(self):
        """Browse products."""
        with tracer.start_as_current_span("browse_products"):
            response = requests.get(f"{self.base_url}/products")
            return response.status_code == 200

    def create_order(self):
        """Create order."""
        with tracer.start_as_current_span("create_order"):
            # Get user
            user_id = random.randint(1, 1000)
            response = requests.get(f"{self.base_url}/api/v1/users/{user_id}")

            if response.status_code == 200:
                # Create order
                order_data = {
                    "user_id": user_id,
                    "items": [{"product_id": random.randint(1, 100), "quantity": 1}]
                }
                response = requests.post(f"{self.base_url}/api/v1/orders", json=order_data)
                return response.status_code == 201

            return False

    def check_order_status(self):
        """Check order status."""
        with tracer.start_as_current_span("check_order_status"):
            order_id = random.randint(1, 10000)
            response = requests.get(f"{self.base_url}/api/v1/orders/{order_id}")
            return response.status_code in [200, 404]

# Run load generator
if __name__ == "__main__":
    generator = YourSystemLoadGenerator("http://yoursystem-gateway:8080")

    while True:
        generator.generate_load()
        time.sleep(random.uniform(0.5, 2.0))
```

### 6. Submit Fault Injection

Submit fault injection for your custom system:

```python
from rcabench.openapi import FaultInjectionApi
from rcabench.openapi.models import DtoSubmitInjectionReq

fault_api = FaultInjectionApi(client)

request = DtoSubmitInjectionReq(
    benchmark="yoursystem",
    handler_nodes=[
        {
            "handler": "network_delay",
            "params": {
                "delay": "100ms",
                "target_service": "order-service",
                "target_namespace": "yoursystem"
            }
        }
    ],
    duration=60,
    description="Network delay on order service"
)

response = fault_api.submit_injection(request)
print(f"Task ID: {response.task_id}")
```

## Advanced Configuration

### Custom Fault Handlers

Create custom fault handlers for your system:

```python
# chaos-experiment/handler/yoursystem_handler.py

def inject_custom_fault(params):
    """Custom fault injection logic."""

    fault_type = params.get("fault_type")
    target_service = params.get("target_service")

    if fault_type == "database_latency":
        # Inject database latency
        return inject_database_latency(target_service, params)
    elif fault_type == "cache_miss":
        # Simulate cache miss
        return inject_cache_miss(target_service, params)
    else:
        raise ValueError(f"Unknown fault type: {fault_type}")

def inject_database_latency(service, params):
    """Inject database latency."""
    delay = params.get("delay", "100ms")

    # Create network chaos targeting database
    chaos_spec = {
        "apiVersion": "chaos-mesh.org/v1alpha1",
        "kind": "NetworkChaos",
        "metadata": {
            "name": f"db-latency-{service}",
            "namespace": "yoursystem"
        },
        "spec": {
            "action": "delay",
            "mode": "one",
            "selector": {
                "namespaces": ["yoursystem"],
                "labelSelectors": {
                    "app": service
                }
            },
            "delay": {
                "latency": delay,
                "correlation": "0"
            },
            "direction": "to",
            "target": {
                "selector": {
                    "namespaces": ["yoursystem"],
                    "labelSelectors": {
                        "app": "postgres"  # or "mysql"
                    }
                },
                "mode": "all"
            },
            "duration": "60s"
        }
    }

    return chaos_spec
```

### Custom Metrics

Define custom metrics for your system:

```python
# Custom metrics collection
CUSTOM_METRICS = {
    "order_processing_time": {
        "query": "histogram_quantile(0.95, rate(order_processing_duration_seconds_bucket[5m]))",
        "type": "gauge"
    },
    "payment_success_rate": {
        "query": "rate(payment_success_total[5m]) / rate(payment_attempts_total[5m])",
        "type": "gauge"
    },
    "cache_hit_rate": {
        "query": "rate(cache_hits_total[5m]) / rate(cache_requests_total[5m])",
        "type": "gauge"
    }
}
```

## Best Practices

1. **Start simple**: Begin with basic fault types (network delay, pod failure)
2. **Instrument thoroughly**: Ensure comprehensive OTEL instrumentation
3. **Test load generator**: Verify realistic traffic patterns
4. **Monitor baseline**: Collect baseline metrics before fault injection
5. **Document dependencies**: Maintain accurate service dependency graph
6. **Validate data**: Check collected traces and metrics quality

## Example: Complete Integration

```bash
# 1. Deploy application
kubectl apply -f deployments/ -n yoursystem

# 2. Deploy load generator
kubectl apply -f loadgenerator.yaml -n yoursystem

# 3. Verify observability
kubectl logs -n observability otel-collector-0 | grep yoursystem

# 4. Register benchmark
python register_benchmark.py

# 5. Submit fault injection
python submit_injection.py

# 6. Monitor progress
python monitor_task.py

# 7. Retrieve dataset
python retrieve_dataset.py
```

## Troubleshooting

### No Traces Collected

Check OTEL configuration:

```bash
# Verify OTEL env vars
kubectl get deployment -n yoursystem user-service -o yaml | grep OTEL

# Check collector logs
kubectl logs -n observability otel-collector-0
```

### Fault Not Applied

Verify Chaos Mesh can target your services:

```bash
# Check service labels
kubectl get pods -n yoursystem --show-labels

# Verify chaos experiment
kubectl get networkchaos -n yoursystem
```

## See Also

- [Fault Injection Guide](../../fault-injection-guide): Basic fault injection
- [Genetic Algorithms](../genetic-algorithms): Intelligent fault scheduling
- [Chaos Mesh Integration](../chaos-mesh-integration): Chaos engineering details
