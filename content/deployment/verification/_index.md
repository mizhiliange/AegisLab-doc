---
title: Verification & Troubleshooting
weight: 5
---

## Verification

> [!TIP]
> **Quick Verification**: Run the health check endpoint first. If it returns `"status": "healthy"`, your deployment is successful. Then proceed with detailed smoke tests.

### Health Check Endpoint

The system provides a comprehensive health check at `/system/health`:

```bash
# Get the API service endpoint
kubectl get svc -n exp rcabench-producer

# Check health (replace <api-service> with actual service IP/hostname)
curl http://<api-service>:8080/system/health
```

Response format:

```json
{
  "status": "healthy",
  "timestamp": "2025-01-28T10:00:00Z",
  "version": "1.1.55",
  "uptime": "5ms",
  "services": {
    "database": {
      "status": "healthy",
      "response_time": "5ms"
    },
    "redis": {
      "status": "healthy",
      "response_time": "2ms"
    },
    "jaeger": {
      "status": "healthy",
      "response_time": "10ms"
    },
    "kubernetes": {
      "status": "healthy",
      "response_time": "15ms"
    },
    "buildkit": {
      "status": "healthy",
      "response_time": "3ms"
    }
  }
}
```

### Smoke Tests

Run these commands after deployment:

```bash
# 1. Check platform dependencies are running
kubectl get pods -n chaos-mesh
kubectl get pods -n cert-manager
kubectl get pods -n monitoring
kubectl get pods -n train-ticket0  # Recommended target

# 2. Check AegisLab pods are running
kubectl get pods -n exp

# Expected output:
# NAME                        READY   STATUS    RESTARTS   AGE
# rcabench-producer-xxx       1/1     Running   0          2m
# rcabench-consumer-xxx       1/1     Running   0          2m
# rcabench-mysql-0            1/1     Running   0          3m
# rcabench-redis-0            1/1     Running   0          3m
# rcabench-etcd-0             1/1     Running   0          3m
# rcabench-jaeger-0           1/1     Running   0          3m

# 3. Check service health
curl http://<api-service>/system/health | jq '.status'
# Expected: "healthy"

# 4. Verify database connectivity
kubectl exec -it statefulset/rcabench-mysql -n exp -- \
  mysqladmin ping -h localhost
# Expected: mysqld is alive

# 5. Verify Redis connectivity
kubectl exec -it statefulset/rcabench-redis -n exp -- \
  redis-cli ping
# Expected: PONG

# 6. Test API authentication
curl -X POST http://<api-service>:8080/api/v2/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' | jq '.data.access_token'

# 7. Verify Chaos Mesh is ready
kubectl get workflows -n chaos-mesh
```

## Troubleshooting

### Common Failure Modes

| Failure Mode               | Symptoms                                   | Diagnosis                      | Resolution                                             |
| -------------------------- | ------------------------------------------ | ------------------------------ | ------------------------------------------------------ |
| Chaos Mesh Not Ready       | Injection failures, "workflow not found"   | Check Chaos Mesh pods          | `kubectl get pods -n chaos-mesh` and restart if needed |
| Database Connection Failed | 503 errors, "connection refused"           | Check MySQL StatefulSet, PVC   | `kubectl delete pod rcabench-mysql-0 -n exp`           |
| Redis Unavailable          | Task queue stalled                         | Health check returns unhealthy | `kubectl delete pod rcabench-redis-0 -n exp`           |
| BuildKit Failure           | Container build errors                     | Check BuildKit logs            | Restart BuildKit, verify Harbor certs                  |
| etcd Timeout               | Config not loading                         | Check etcd cluster health      | Reinitialize etcd StatefulSet                          |
| JuiceFS Mount Failed       | Dataset not accessible (Test/Staging only) | Check PV/PVC binding           | Verify JuiceFS secret, remount                         |
| OTEL Demo Not Found        | Cannot create injection targets            | Check OTEL Demo namespace      | `kubectl get pods -n otel-demo0`                       |
| High Memory Usage          | OOMKilled pods                             | Check container limits         | Increase resource limits                               |
| Stuck Tasks                | Tasks in Running state forever             | Check dead letter queue        | Manually cancel stuck tasks                            |

## Support

For issues and questions:

1. Check existing troubleshooting section
2. Review logs in `/app/logs/jobs/`
3. Open a GitHub issue with:
   - Environment (debug/test/staging/prod)
   - Error messages and logs
   - Steps to reproduce
   - Kubernetes version and cluster info
