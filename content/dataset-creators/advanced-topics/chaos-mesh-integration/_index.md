---
title: Chaos Mesh Integration
weight: 3
---


Deep dive into how AegisLab integrates with Chaos Mesh for fault injection.

## Overview

Chaos Mesh is a cloud-native chaos engineering platform that provides various types of chaos experiments. AegisLab uses Chaos Mesh as the underlying engine for fault injection.

## Architecture

```
AegisLab → chaos-experiment → Chaos Mesh CRDs → Kubernetes
    ↓            ↓                  ↓              ↓
  API      Handlers           NetworkChaos    Target Pods
           Controllers        PodChaos
           Specs              StressChaos
```

## Chaos Mesh CRDs

### NetworkChaos

Inject network faults:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-example
  namespace: ts
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - ts
    labelSelectors:
      app: ts-order-service
  delay:
    latency: "100ms"
    correlation: "0"
    jitter: "10ms"
  duration: "60s"
```

**Actions:**
- `delay`: Add network latency
- `loss`: Drop packets
- `corrupt`: Corrupt packets
- `duplicate`: Duplicate packets
- `partition`: Network partition
- `bandwidth`: Limit bandwidth

### PodChaos

Inject pod-level faults:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-failure-example
  namespace: ts
spec:
  action: pod-failure
  mode: one
  selector:
    namespaces:
      - ts
    labelSelectors:
      app: ts-order-service
  duration: "60s"
```

**Actions:**
- `pod-failure`: Kill pod
- `pod-kill`: Kill pod permanently
- `container-kill`: Kill specific container

### StressChaos

Inject resource stress:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: memory-stress-example
  namespace: ts
spec:
  mode: one
  selector:
    namespaces:
      - ts
    labelSelectors:
      app: ts-order-service
  stressors:
    memory:
      workers: 4
      size: "256MB"
  duration: "60s"
```

**Stressors:**
- `cpu`: CPU stress
- `memory`: Memory stress

### JVMChaos

Inject JVM-level faults:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: JVMChaos
metadata:
  name: jvm-exception-example
  namespace: ts
spec:
  action: exception
  mode: one
  selector:
    namespaces:
      - ts
    labelSelectors:
      app: ts-order-service
  class: "com.example.OrderService"
  method: "createOrder"
  exception: "java.lang.RuntimeException('Injected exception')"
  duration: "60s"
```

**Actions:**
- `exception`: Throw exception
- `gc`: Trigger garbage collection
- `latency`: Add method latency
- `return`: Modify return value
- `stress`: CPU/memory stress

## chaos-experiment Integration

### Three-Layer Architecture

**Layer 1: Specs** (`chaos/`)
Low-level CRD builders:

```go
// chaos/network.go
func NetworkDelay(namespace, service, delay string) *v1alpha1.NetworkChaos {
    return &v1alpha1.NetworkChaos{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("network-delay-%s", service),
            Namespace: namespace,
        },
        Spec: v1alpha1.NetworkChaosSpec{
            Action: v1alpha1.DelayAction,
            Mode:   v1alpha1.OnePodMode,
            Selector: v1alpha1.PodSelector{
                Namespaces: []string{namespace},
                LabelSelectors: map[string]string{
                    "app": service,
                },
            },
            Delay: &v1alpha1.DelaySpec{
                Latency: delay,
            },
            Duration: pointer.String("60s"),
        },
    }
}
```

**Layer 2: Controllers** (`controllers/`)
Orchestration functions:

```go
// controllers/network.go
func InjectNetworkDelay(ctx context.Context, params NetworkDelayParams) error {
    // Create chaos spec
    chaos := chaos.NetworkDelay(params.Namespace, params.Service, params.Delay)

    // Apply to cluster
    if err := applyChaosMesh(ctx, chaos); err != nil {
        return err
    }

    // Wait for chaos to be active
    return waitForChaosActive(ctx, chaos.Name, chaos.Namespace)
}
```

**Layer 3: Handlers** (`handler/`)
Intelligent generation:

```go
// handler/trainticket.go
func HandleTrainTicketFault(node HandlerNode) error {
    handler := node.Handler
    params := node.Params

    switch handler {
    case "network_delay":
        return handleNetworkDelay(params)
    case "pod_failure":
        return handlePodFailure(params)
    case "memory_pressure":
        return handleMemoryPressure(params)
    default:
        return fmt.Errorf("unknown handler: %s", handler)
    }
}

func handleNetworkDelay(params map[string]interface{}) error {
    service := params["target_service"].(string)
    delay := params["delay"].(string)
    namespace := params["target_namespace"].(string)

    // Use system metadata to validate service
    if !trainticket.IsValidService(service) {
        return fmt.Errorf("invalid service: %s", service)
    }

    // Inject fault
    return controllers.InjectNetworkDelay(context.Background(), controllers.NetworkDelayParams{
        Namespace: namespace,
        Service:   service,
        Delay:     delay,
    })
}
```

## Applying Chaos

### Programmatic Application

```go
import (
    "context"
    "github.com/chaos-mesh/chaos-mesh/api/v1alpha1"
    "k8s.io/client-go/kubernetes/scheme"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

func applyChaosMesh(ctx context.Context, chaos client.Object) error {
    // Get Kubernetes client
    k8sClient, err := getK8sClient()
    if err != nil {
        return err
    }

    // Register Chaos Mesh scheme
    v1alpha1.AddToScheme(scheme.Scheme)

    // Create chaos experiment
    if err := k8sClient.Create(ctx, chaos); err != nil {
        return err
    }

    return nil
}
```

### Using kubectl

```bash
# Apply chaos directly
kubectl apply -f network-delay.yaml

# View chaos experiments
kubectl get networkchaos -n ts

# Describe chaos
kubectl describe networkchaos network-delay-example -n ts

# Delete chaos
kubectl delete networkchaos network-delay-example -n ts
```

## Chaos Lifecycle

### 1. Creation

```bash
# Chaos experiment created
kubectl get networkchaos -n ts
NAME                    AGE
network-delay-example   1s
```

### 2. Injection

```bash
# Chaos becomes active
kubectl describe networkchaos network-delay-example -n ts
Status:
  Conditions:
    Status: True
    Type:   AllInjected
```

### 3. Duration

Chaos remains active for specified duration (e.g., 60s).

### 4. Recovery

```bash
# Chaos automatically removed after duration
kubectl get networkchaos -n ts
No resources found in ts namespace.
```

## Advanced Features

### Scheduling

Schedule chaos experiments:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: Schedule
metadata:
  name: scheduled-chaos
  namespace: ts
spec:
  schedule: "@every 1h"
  type: NetworkChaos
  networkChaos:
    action: delay
    mode: one
    selector:
      namespaces:
        - ts
      labelSelectors:
        app: ts-order-service
    delay:
      latency: "100ms"
    duration: "5m"
```

### Workflow

Chain multiple chaos experiments:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: Workflow
metadata:
  name: chaos-workflow
  namespace: ts
spec:
  entry: network-then-pod
  templates:
    - name: network-then-pod
      templateType: Serial
      children:
        - network-delay
        - pod-failure
    - name: network-delay
      templateType: NetworkChaos
      networkChaos:
        action: delay
        mode: one
        selector:
          namespaces:
            - ts
          labelSelectors:
            app: ts-order-service
        delay:
          latency: "100ms"
        duration: "30s"
    - name: pod-failure
      templateType: PodChaos
      podChaos:
        action: pod-failure
        mode: one
        selector:
          namespaces:
            - ts
          labelSelectors:
            app: ts-payment-service
        duration: "30s"
```

## Monitoring Chaos

### Check Status

```bash
# Get chaos status
kubectl get networkchaos -n ts -o wide

# View events
kubectl get events -n ts | grep chaos

# Check pod status
kubectl get pods -n ts -l app=ts-order-service
```

### Chaos Dashboard

Access Chaos Mesh dashboard:

```bash
# Port forward
kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333

# Open browser
open http://localhost:2333
```

## Troubleshooting

### Chaos Not Applied

**Check selector:**

```bash
# Verify pods match selector
kubectl get pods -n ts -l app=ts-order-service

# If no pods match, chaos won't be applied
```

**Check Chaos Mesh:**

```bash
# Verify Chaos Mesh is running
kubectl get pods -n chaos-mesh

# Check controller logs
kubectl logs -n chaos-mesh chaos-controller-manager-0
```

### Chaos Stuck

**Delete manually:**

```bash
# Force delete
kubectl delete networkchaos network-delay-example -n ts --force --grace-period=0
```

### Permission Errors

**Check RBAC:**

```bash
# Verify service account has permissions
kubectl auth can-i create networkchaos --as=system:serviceaccount:aegislab:aegislab-worker -n ts
```

## Best Practices

1. **Use labels**: Ensure pods have proper labels for targeting
2. **Test selectors**: Verify selectors match intended pods
3. **Set duration**: Always specify duration to avoid indefinite chaos
4. **Monitor impact**: Watch metrics during chaos injection
5. **Clean up**: Remove chaos experiments after use
6. **Use namespaces**: Isolate chaos to specific namespaces

## See Also

- [Fault Injection Guide](../../fault-injection-guide): Basic fault injection
- [Custom Benchmarks](../custom-benchmarks): Using custom systems
- [Genetic Algorithms](../genetic-algorithms): Intelligent fault scheduling
- [Chaos Mesh Documentation](https://chaos-mesh.org/docs/): Official Chaos Mesh docs
