---
title: API Reference
weight: 1
---

Complete AegisLab API reference for fault injection and dataset management.

## Base URL

```
${AEGISLAB_API_URL}  # Default: http://10.10.10.220:8080
```

## Authentication

All API requests require authentication:

```http
Authorization: Bearer <your-api-token>
```

## Fault Injection API

### Submit Injection

Submit a fault injection request.

**Endpoint:** `POST /api/v1/fault-injection/submit`

**Request Body:**

```json
{
  "benchmark": "trainticket",
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
  "description": "Network delay on order service",
  "collection_window": 120,
  "pre_collection_window": 30
}
```

**Response:**

```json
{
  "task_id": "task-abc123",
  "status": "pending",
  "created_at": "2026-01-18T10:30:00Z"
}
```

### Get Task Status

Get fault injection task status.

**Endpoint:** `GET /api/v1/tasks/{task_id}`

**Response:**

```json
{
  "task_id": "task-abc123",
  "status": "running",
  "progress": 45.5,
  "current_phase": "collecting",
  "started_at": "2026-01-18T10:30:00Z",
  "elapsed_seconds": 120,
  "traces_collected": 5234,
  "spans_collected": 5234,
  "metrics_collected": 2456
}
```

**Status Values:**
- `pending`: Queued
- `preparing`: Setting up
- `injecting`: Applying fault
- `collecting`: Gathering data
- `processing`: Converting to parquet
- `completed`: Finished
- `failed`: Error occurred

### Stream Trace Events

Stream real-time task events.

**Endpoint:** `GET /api/v1/tasks/{task_id}/events`

**Response:** Server-Sent Events (SSE)

```
event: trace
data: {"timestamp": "2026-01-18T10:30:15Z", "level": "INFO", "message": "Task started"}

event: trace
data: {"timestamp": "2026-01-18T10:30:25Z", "level": "INFO", "message": "Fault injection active"}
```

### Cancel Task

Cancel a running task.

**Endpoint:** `POST /api/v1/tasks/{task_id}/cancel`

**Response:**

```json
{
  "success": true,
  "message": "Task cancelled successfully"
}
```

## Dataset API

### List Datasets

List available datasets.

**Endpoint:** `GET /api/v1/datasets`

**Query Parameters:**
- `benchmark` (optional): Filter by benchmark system
- `limit` (optional): Maximum results (default: 100)
- `offset` (optional): Pagination offset (default: 0)

**Response:**

```json
{
  "datasets": [
    {
      "name": "trainticket-pandora-v1",
      "benchmark": "trainticket",
      "datapack_count": 100,
      "size_bytes": 35651584,
      "created_at": "2026-01-15T10:00:00Z",
      "description": "Genetic algorithm-generated scenarios"
    }
  ],
  "total": 1,
  "limit": 100,
  "offset": 0
}
```

### Get Dataset

Get detailed dataset information.

**Endpoint:** `GET /api/v1/datasets/{dataset_name}`

**Response:**

```json
{
  "name": "trainticket-pandora-v1",
  "benchmark": "trainticket",
  "datapack_count": 100,
  "size_bytes": 35651584,
  "created_at": "2026-01-15T10:00:00Z",
  "updated_at": "2026-01-15T15:30:00Z",
  "description": "Genetic algorithm-generated scenarios",
  "version": "1.0.0",
  "fault_types": ["network_delay", "pod_failure", "memory_pressure"],
  "statistics": {
    "total_traces": 1523400,
    "avg_error_rate": 0.42,
    "avg_traces_per_datapack": 15234
  }
}
```

### Download Dataset

Download dataset files.

**Endpoint:** `GET /api/v1/datasets/{dataset_name}/download`

**Query Parameters:**
- `datapack_ids` (optional): Comma-separated datapack IDs (e.g., "0,1,2")

**Response:** ZIP archive containing dataset files

### Register Dataset

Register a new dataset.

**Endpoint:** `POST /api/v1/datasets/register`

**Request Body:**

```json
{
  "name": "trainticket-custom-001",
  "benchmark": "trainticket",
  "datapack_count": 100,
  "description": "Custom fault scenarios",
  "storage_path": "/mnt/jfs/rcabench-platform-v2/trainticket-custom-001",
  "version": "1.0.0",
  "license": "MIT",
  "tags": ["trainticket", "custom"]
}
```

**Response:**

```json
{
  "dataset_id": "trainticket-custom-001",
  "status": "registered"
}
```

## Benchmark API

### List Benchmarks

List available benchmark systems.

**Endpoint:** `GET /api/v1/benchmarks`

**Response:**

```json
{
  "benchmarks": [
    {
      "name": "trainticket",
      "display_name": "TrainTicket",
      "description": "Train ticket booking system",
      "service_count": 42,
      "namespace": "ts"
    },
    {
      "name": "otel-demo",
      "display_name": "OTEL Demo",
      "description": "OpenTelemetry Demo Application",
      "service_count": 11,
      "namespace": "otel-demo"
    }
  ]
}
```

### Register Benchmark

Register a custom benchmark system.

**Endpoint:** `POST /api/v1/benchmarks/register`

**Request Body:**

```json
{
  "name": "yoursystem",
  "display_name": "Your System",
  "description": "Custom microservices application",
  "namespace": "yoursystem",
  "service_count": 6,
  "metadata": {
    "services": ["frontend", "api-gateway", "user-service"],
    "endpoints": {
      "frontend": ["GET /", "GET /products"],
      "api-gateway": ["GET /api/v1/*"]
    }
  }
}
```

**Response:**

```json
{
  "benchmark_id": "yoursystem",
  "status": "registered"
}
```

## Error Responses

All endpoints may return error responses:

**400 Bad Request:**

```json
{
  "error": "Invalid request",
  "message": "Missing required field: benchmark",
  "code": "INVALID_REQUEST"
}
```

**401 Unauthorized:**

```json
{
  "error": "Unauthorized",
  "message": "Invalid or missing API token",
  "code": "UNAUTHORIZED"
}
```

**404 Not Found:**

```json
{
  "error": "Not found",
  "message": "Dataset 'trainticket-custom-001' not found",
  "code": "NOT_FOUND"
}
```

**500 Internal Server Error:**

```json
{
  "error": "Internal server error",
  "message": "Failed to process request",
  "code": "INTERNAL_ERROR"
}
```

## Rate Limiting

API requests are rate limited:
- **Default**: 100 requests per minute
- **Burst**: 200 requests per minute

Rate limit headers:

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1705584060
```

## Pagination

List endpoints support pagination:

```http
GET /api/v1/datasets?limit=50&offset=100
```

Response includes pagination metadata:

```json
{
  "datasets": [...],
  "total": 250,
  "limit": 50,
  "offset": 100,
  "has_more": true
}
```

## Webhooks

Configure webhooks for task events:

**Webhook Payload:**

```json
{
  "event": "task.completed",
  "task_id": "task-abc123",
  "status": "completed",
  "dataset_id": "trainticket-custom-001",
  "timestamp": "2026-01-18T11:30:00Z",
  "data": {
    "traces_collected": 15234,
    "metrics_collected": 8456
  }
}
```

**Webhook Events:**
- `task.created`
- `task.started`
- `task.completed`
- `task.failed`
- `task.cancelled`
- `data.collected`

## SDK Usage

### Python SDK

```python
from rcabench.openapi import ApiClient, Configuration, FaultInjectionApi

config = Configuration(
    host="${AEGISLAB_API_URL}",  # Default: http://10.10.10.220:8080
    api_key={"Authorization": "Bearer your-token"}
)

client = ApiClient(config)
api = FaultInjectionApi(client)

# Submit injection
response = api.submit_injection(request)
```

### cURL Examples

**Submit Injection:**

```bash
curl -X POST ${AEGISLAB_API_URL}/api/v1/fault-injection/submit \
  -H "Authorization: Bearer your-token" \
  -H "Content-Type: application/json" \
  -d '{
    "benchmark": "trainticket",
    "handler_nodes": [{"handler": "network_delay", "params": {...}}],
    "duration": 60
  }'
# Default AEGISLAB_API_URL: http://10.10.10.220:8080
```

**Get Task Status:**

```bash
curl ${AEGISLAB_API_URL}/api/v1/tasks/task-abc123 \
  -H "Authorization: Bearer your-token"
# Default AEGISLAB_API_URL: http://10.10.10.220:8080
```

## See Also

- [Fault Catalog](../fault-catalog): Available fault types
- [SDK Examples](../sdk-examples): Code examples
- [Using AegisLab](../../using-aegislab): SDK usage guide
