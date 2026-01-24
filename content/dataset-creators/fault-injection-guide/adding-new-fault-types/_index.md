---
title: Adding New Fault Types
weight: 6
---

This guide documents the complete process for adding new fault types to the AegisLab ecosystem. It covers the full integration chain from Chaos Mesh CRD through chaos-experiment to AegisLab, using RuntimeMutatorChaos as a reference implementation.

## Architecture Overview

New fault types must be integrated across multiple layers:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AegisLab Frontend                           │
│                    (FaultTypePanel.tsx, forms)                      │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         AegisLab Backend                            │
│           (API endpoints, consumer, fault_injection.go)             │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        chaos-experiment                             │
│              (handler, controller, chaos wrapper)                   │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          Chaos Mesh                                 │
│              (CRD, webhook, controller, chaosdaemon)                │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Target Application                             │
│                   (Kubernetes pods/services)                        │
└─────────────────────────────────────────────────────────────────────┘
```

## Reference: RuntimeMutatorChaos

RuntimeMutatorChaos is a JVM-level fault type that mutates Java bytecode at runtime. It demonstrates all aspects of fault type integration:

| Component | Files Modified/Created |
|-----------|----------------------|
| Chaos Mesh CRD | `api/v1alpha1/runtimemutatorchaos_types.go` |
| Chaos Mesh Webhook | `api/v1alpha1/runtimemutatorchaos_webhook.go` |
| Chaos Mesh Controller | `controllers/chaosimpl/runtimemutatorchaos/impl.go` |
| chaosdaemon gRPC | `pkg/chaosdaemon/pb/chaosdaemon.proto`, `jvm_server.go` |
| chaos-experiment Handler | `handler/jvm_runtime_mutator.go` |
| chaos-experiment Controller | `controllers/jvm_runtime_mutator.go` |
| chaos-experiment Wrapper | `chaos/jvm_runtime_mutator.go` |
| AegisLab | Uses generic flow via chaos-experiment |

## Layer 1: Chaos Mesh CRD

### Step 1.1: Define CRD Types

Create the CRD type definition in `api/v1alpha1/<faulttype>_types.go`:

```go
package v1alpha1

import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

// +kubebuilder:object:root=true
// +chaos-mesh:experiment
// +genclient
type RuntimeMutatorChaos struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec   RuntimeMutatorChaosSpec   `json:"spec,omitempty"`
    Status RuntimeMutatorChaosStatus `json:"status,omitempty"`
}

type RuntimeMutatorChaosSpec struct {
    ContainerSelector `json:",inline"`
    Duration          *string                  `json:"duration,omitempty"`
    Action            RuntimeMutatorChaosAction `json:"action"`
    RuntimeMutator    RuntimeMutatorParameter  `json:"runtimeMutator"`
}

type RuntimeMutatorChaosAction string

const (
    RuntimeMutatorConstantAction RuntimeMutatorChaosAction = "constant"
    RuntimeMutatorOperatorAction RuntimeMutatorChaosAction = "operator"
    RuntimeMutatorStringAction   RuntimeMutatorChaosAction = "string"
)

type RuntimeMutatorParameter struct {
    Class    string  `json:"class"`
    Method   string  `json:"method"`
    From     *string `json:"from,omitempty"`
    To       *string `json:"to,omitempty"`
    Strategy *string `json:"strategy,omitempty"`
    Port     *int32  `json:"port,omitempty"`
}
```

Key annotations:
- `+kubebuilder:object:root=true` - Marks this as a CRD root object
- `+chaos-mesh:experiment` - Integrates with Chaos Mesh experiment framework
- `+genclient` - Generates Kubernetes client code

### Step 1.2: Implement Required Interfaces

Implement `GetSelectorSpecs()` for pod selection:

```go
func (obj *RuntimeMutatorChaos) GetSelectorSpecs() map[string]interface{} {
    return map[string]interface{}{
        ".": &obj.Spec.ContainerSelector,
    }
}
```

### Step 1.3: Generate CRD Manifests

```bash
cd chaos-mesh
make generate
```

This generates:
- CRD YAML in `config/crd/bases/chaos-mesh.org_runtimemutatorchaos.yaml`
- DeepCopy methods in `api/v1alpha1/zz_generated.deepcopy.go`

Copy CRD to Helm chart:
```bash
cp config/crd/bases/chaos-mesh.org_runtimemutatorchaos.yaml helm/chaos-mesh/crds/
```

## Layer 2: Chaos Mesh Webhook

### Step 2.1: Create Webhook Validation

Create `api/v1alpha1/<faulttype>_webhook.go`:

```go
package v1alpha1

import (
    "fmt"
    "github.com/chaos-mesh/chaos-mesh/api/genericwebhook"
)

type RuntimeMutatorChaosWebhookImpl struct{}

func (impl *RuntimeMutatorChaosWebhookImpl) Validate(obj GenericChaos) error {
    chaos := obj.(*RuntimeMutatorChaos)
    spec := chaos.Spec.RuntimeMutator

    // Validate class and method are specified
    if spec.Class == "" {
        return fmt.Errorf("class is required")
    }
    if spec.Method == "" {
        return fmt.Errorf("method is required")
    }

    // Validate action-specific fields
    switch chaos.Spec.Action {
    case RuntimeMutatorConstantAction:
        if spec.From == nil || spec.To == nil {
            return fmt.Errorf("constant mutation requires 'from' and 'to' fields")
        }
        if spec.Strategy != nil {
            return fmt.Errorf("constant mutation should not have 'strategy' field")
        }
    case RuntimeMutatorOperatorAction, RuntimeMutatorStringAction:
        if spec.Strategy == nil {
            return fmt.Errorf("%s mutation requires 'strategy' field", chaos.Spec.Action)
        }
        if spec.From != nil || spec.To != nil {
            return fmt.Errorf("%s mutation should not have 'from' or 'to' fields", chaos.Spec.Action)
        }
    default:
        return fmt.Errorf("unknown action: %s", chaos.Spec.Action)
    }

    return nil
}

func (impl *RuntimeMutatorChaosWebhookImpl) Default(obj GenericChaos) {
    chaos := obj.(*RuntimeMutatorChaos)
    if chaos.Spec.RuntimeMutator.Port == nil {
        port := int32(9090)
        chaos.Spec.RuntimeMutator.Port = &port
    }
}
```

### Step 2.2: Register Webhook

Add to `controllers/types/types.go`:

```go
func init() {
    // ... existing registrations ...
    all[reflect.TypeOf(v1alpha1.RuntimeMutatorChaos{})] = ChaosObjects{
        Kind: v1alpha1.KindRuntimeMutatorChaos,
        Object: &v1alpha1.RuntimeMutatorChaos{},
        WebhookObject: &v1alpha1.RuntimeMutatorChaosWebhookImpl{},
    }
}
```

## Layer 3: Chaos Mesh Controller

### Step 3.1: Create Controller Implementation

Create `controllers/chaosimpl/runtimemutatorchaos/impl.go`:

```go
package runtimemutatorchaos

import (
    "context"
    "github.com/chaos-mesh/chaos-mesh/api/v1alpha1"
    "github.com/chaos-mesh/chaos-mesh/controllers/chaosimpl/utils"
    "github.com/chaos-mesh/chaos-mesh/pkg/chaosdaemon/pb"
    "go.uber.org/fx"
)

type Impl struct {
    client.Client
    decoder *utils.ContainerRecordDecoder
}

func NewImpl(c client.Client, decoder *utils.ContainerRecordDecoder) *Impl {
    return &Impl{Client: c, decoder: decoder}
}

func (impl *Impl) Apply(ctx context.Context, index int, records []*v1alpha1.Record, obj v1alpha1.InnerObject) (v1alpha1.Phase, error) {
    chaos := obj.(*v1alpha1.RuntimeMutatorChaos)

    // Decode container info
    decodedContainer, err := impl.decoder.DecodeContainerRecord(ctx, records[index], chaos)
    if err != nil {
        return v1alpha1.NotInjected, err
    }

    // Build gRPC request
    req := &pb.RuntimeMutatorRequest{
        ContainerId:   decodedContainer.ContainerId,
        Action:        string(chaos.Spec.Action),
        ClassName:     chaos.Spec.RuntimeMutator.Class,
        MethodName:    chaos.Spec.RuntimeMutator.Method,
        EnterNS:       true,
    }

    // Add action-specific fields
    if chaos.Spec.RuntimeMutator.From != nil {
        req.FromValue = *chaos.Spec.RuntimeMutator.From
    }
    if chaos.Spec.RuntimeMutator.To != nil {
        req.ToValue = *chaos.Spec.RuntimeMutator.To
    }
    if chaos.Spec.RuntimeMutator.Strategy != nil {
        req.Strategy = *chaos.Spec.RuntimeMutator.Strategy
    }
    if chaos.Spec.RuntimeMutator.Port != nil {
        req.Port = *chaos.Spec.RuntimeMutator.Port
    }

    // Call chaosdaemon
    daemonClient := pb.NewChaosDaemonClient(decodedContainer.DaemonClient)
    _, err = daemonClient.InstallRuntimeMutator(ctx, req)
    if err != nil {
        return v1alpha1.NotInjected, err
    }

    return v1alpha1.Injected, nil
}

func (impl *Impl) Recover(ctx context.Context, index int, records []*v1alpha1.Record, obj v1alpha1.InnerObject) (v1alpha1.Phase, error) {
    chaos := obj.(*v1alpha1.RuntimeMutatorChaos)

    decodedContainer, err := impl.decoder.DecodeContainerRecord(ctx, records[index], chaos)
    if err != nil {
        return v1alpha1.NotRecovered, err
    }

    req := &pb.RuntimeMutatorRequest{
        ContainerId: decodedContainer.ContainerId,
        EnterNS:     true,
    }

    daemonClient := pb.NewChaosDaemonClient(decodedContainer.DaemonClient)
    _, err = daemonClient.UninstallRuntimeMutator(ctx, req)
    if err != nil {
        // Log but don't fail - agent may not be running
        log.Error(err, "failed to uninstall runtime mutator")
    }

    return v1alpha1.NotInjected, nil
}

var Module = fx.Module(
    "runtimemutatorchaos",
    fx.Provide(
        fx.Annotated{
            Name:   "impl",
            Target: NewImpl,
        },
    ),
)
```

### Step 3.2: Register Controller in fx Module

Add to `controllers/chaosimpl/fx.go`:

```go
import "github.com/chaos-mesh/chaos-mesh/controllers/chaosimpl/runtimemutatorchaos"

var Module = fx.Options(
    // ... existing modules ...
    runtimemutatorchaos.Module,
)
```

## Layer 4: chaosdaemon gRPC

### Step 4.1: Define Protobuf Messages

Add to `pkg/chaosdaemon/pb/chaosdaemon.proto`:

```protobuf
service ChaosDaemon {
    // ... existing RPCs ...
    rpc InstallRuntimeMutator(RuntimeMutatorRequest) returns (RuntimeMutatorResponse);
    rpc UninstallRuntimeMutator(RuntimeMutatorRequest) returns (RuntimeMutatorResponse);
}

message RuntimeMutatorRequest {
    string container_id = 1;
    string action = 2;
    string class_name = 3;
    string method_name = 4;
    string from_value = 5;
    string to_value = 6;
    string strategy = 7;
    int32 port = 8;
    bool enter_ns = 9;
}

message RuntimeMutatorResponse {
    string status = 1;
}
```

Generate code:
```bash
make proto
```

### Step 4.2: Implement gRPC Methods

Add to `pkg/chaosdaemon/jvm_server.go`:

```go
func (s *DaemonServer) InstallRuntimeMutator(ctx context.Context, req *pb.RuntimeMutatorRequest) (*pb.RuntimeMutatorResponse, error) {
    log := log.WithValues("containerId", req.ContainerId)

    // Get container PID
    pid, err := s.criClient.GetPidFromContainerID(ctx, req.ContainerId)
    if err != nil {
        return nil, err
    }

    // Find Java process
    javaPid, err := findJavaPid(pid)
    if err != nil {
        return nil, err
    }

    // Build agent arguments
    agentArgs := fmt.Sprintf("mutator_action=%s,mutator_class=%s,mutator_method=%s",
        req.Action, req.ClassName, req.MethodName)

    if req.FromValue != "" {
        agentArgs += fmt.Sprintf(",mutator_from=%s", req.FromValue)
    }
    if req.ToValue != "" {
        agentArgs += fmt.Sprintf(",mutator_to=%s", req.ToValue)
    }
    if req.Strategy != "" {
        agentArgs += fmt.Sprintf(",mutator_strategy=%s", req.Strategy)
    }
    if req.Port > 0 {
        agentArgs += fmt.Sprintf(",mutator_port=%d", req.Port)
    }

    // Attach agent using jattach
    agentPath := filepath.Join(os.Getenv("BYTEMAN_HOME"), "mutator-agent.jar")
    cmd := exec.CommandContext(ctx, "jattach", strconv.Itoa(javaPid), "load", "instrument", "false",
        fmt.Sprintf("%s=%s", agentPath, agentArgs))

    if req.EnterNS {
        cmd = s.nsenter(ctx, pid, cmd)
    }

    output, err := cmd.CombinedOutput()
    if err != nil {
        log.Error(err, "failed to install runtime mutator", "output", string(output))
        return nil, err
    }

    return &pb.RuntimeMutatorResponse{Status: "installed"}, nil
}

func (s *DaemonServer) UninstallRuntimeMutator(ctx context.Context, req *pb.RuntimeMutatorRequest) (*pb.RuntimeMutatorResponse, error) {
    // Re-attach with disable command
    pid, err := s.criClient.GetPidFromContainerID(ctx, req.ContainerId)
    if err != nil {
        return nil, err
    }

    javaPid, err := findJavaPid(pid)
    if err != nil {
        return &pb.RuntimeMutatorResponse{Status: "not_found"}, nil
    }

    agentPath := filepath.Join(os.Getenv("BYTEMAN_HOME"), "mutator-agent.jar")
    cmd := exec.CommandContext(ctx, "jattach", strconv.Itoa(javaPid), "load", "instrument", "false",
        fmt.Sprintf("%s=enabled=false,clear=true", agentPath))

    if req.EnterNS {
        cmd = s.nsenter(ctx, pid, cmd)
    }

    _, _ = cmd.CombinedOutput() // Best effort

    return &pb.RuntimeMutatorResponse{Status: "uninstalled"}, nil
}
```

## Layer 5: chaos-experiment Handler

### Step 5.1: Create Handler Spec

Create `handler/jvm_runtime_mutator.go`:

```go
package handler

import (
    "context"
    "fmt"
    chaosmeshv1alpha1 "github.com/chaos-mesh/chaos-mesh/api/v1alpha1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

type JVMRuntimeMutatorSpec struct {
    Duration       int    `range:"1-60"`
    System         int    `range:"0-0" dynamic:"true"`
    MethodIdx      int    `range:"0-0" dynamic:"true"`
    MutationType   string `range:"constant,operator,string"`
    MutationOpt    int    `range:"0-10"`
    MutationFrom   string `description:"Original value for constant mutation"`
    MutationTo     string `description:"Replacement value for constant mutation"`
    MutationStrategy string `description:"Strategy for operator/string mutation"`
}

func (spec *JVMRuntimeMutatorSpec) Create(cli client.Client, opt ...Option) (string, error) {
    o := buildOptions(opt...)

    // Get target method info
    methodInfo := GetMethodByIndex(o.Namespace, o.BenchmarkName, spec.MethodIdx)
    if methodInfo == nil {
        return "", fmt.Errorf("method not found at index %d", spec.MethodIdx)
    }

    // Build mutation parameters
    mutator := chaosmeshv1alpha1.RuntimeMutatorParameter{
        Class:  methodInfo.ClassName,
        Method: methodInfo.MethodName,
    }

    // Set action-specific fields
    action := chaosmeshv1alpha1.RuntimeMutatorChaosAction(spec.MutationType)
    switch action {
    case chaosmeshv1alpha1.RuntimeMutatorConstantAction:
        mutator.From = &spec.MutationFrom
        mutator.To = &spec.MutationTo
    case chaosmeshv1alpha1.RuntimeMutatorOperatorAction, chaosmeshv1alpha1.RuntimeMutatorStringAction:
        strategy := getMutationStrategy(action, spec.MutationOpt)
        mutator.Strategy = &strategy
    }

    // Create CRD
    chaos := &chaosmeshv1alpha1.RuntimeMutatorChaos{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("jrm-%s", uuid.NewString()[:8]),
            Namespace: o.Namespace,
        },
        Spec: chaosmeshv1alpha1.RuntimeMutatorChaosSpec{
            ContainerSelector: chaosmeshv1alpha1.ContainerSelector{
                PodSelector: chaosmeshv1alpha1.PodSelector{
                    Selector: chaosmeshv1alpha1.PodSelectorSpec{
                        GenericSelectorSpec: chaosmeshv1alpha1.GenericSelectorSpec{
                            Namespaces:     []string{o.Namespace},
                            LabelSelectors: methodInfo.LabelSelectors,
                        },
                    },
                    Mode: chaosmeshv1alpha1.OneMode,
                },
            },
            Action:         action,
            RuntimeMutator: mutator,
            Duration:       ptr.String(fmt.Sprintf("%dm", spec.Duration)),
        },
    }

    err := cli.Create(context.Background(), chaos)
    return chaos.Name, err
}
```

### Step 5.2: Register in Handler Maps

Add to `handler/handler.go`:

```go
const (
    // ... existing types ...
    JVMRuntimeMutator ChaosType = 32
)

var ChaosTypeMap = map[ChaosType]string{
    // ... existing mappings ...
    JVMRuntimeMutator: "JVMRuntimeMutator",
}

var ChaosNameMap = map[string]ChaosType{
    // ... existing mappings ...
    "JVMRuntimeMutator": JVMRuntimeMutator,
}

var SpecMap = map[ChaosType]any{
    // ... existing specs ...
    JVMRuntimeMutator: JVMRuntimeMutatorSpec{},
}

var ChaosHandlers = map[ChaosType]Injection{
    // ... existing handlers ...
    JVMRuntimeMutator: &JVMRuntimeMutatorSpec{},
}
```

### Step 5.3: Add to InjectionConf

Add field to `InjectionConf` struct:

```go
type InjectionConf struct {
    // ... existing fields ...
    JVMRuntimeMutator JVMRuntimeMutatorSpec `range:"0-3"`
}
```

## Layer 6: chaos-experiment Controller

### Step 6.1: Create Controller Functions

Create `controllers/jvm_runtime_mutator.go`:

```go
package controllers

import (
    "context"
    chaosmeshv1alpha1 "github.com/chaos-mesh/chaos-mesh/api/v1alpha1"
    "github.com/chaos-mesh/chaos-mesh/chaos"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

func CreateJVMRuntimeMutatorChaos(
    cli client.Client,
    opts ...chaos.RuntimeMutatorOption,
) (string, error) {
    spec := chaos.GenerateRuntimeMutatorChaosSpec(opts...)

    chaos := &chaosmeshv1alpha1.RuntimeMutatorChaos{
        ObjectMeta: metav1.ObjectMeta{
            Name:      spec.Name,
            Namespace: spec.Namespace,
        },
        Spec: spec.Spec,
    }

    err := cli.Create(context.Background(), chaos)
    return chaos.Name, err
}

func AddJVMRuntimeMutatorWorkflowNodes(
    tasks []*chaosmeshv1alpha1.Task,
    opts ...chaos.RuntimeMutatorOption,
) []*chaosmeshv1alpha1.Task {
    spec := chaos.GenerateRuntimeMutatorChaosSpec(opts...)

    task := &chaosmeshv1alpha1.Task{
        Name: spec.Name,
        Type: chaosmeshv1alpha1.TypeJVMChaos,
        EmbedChaos: &chaosmeshv1alpha1.EmbedChaos{
            RuntimeMutatorChaos: &spec.Spec,
        },
    }

    return append(tasks, task)
}

func ScheduleJVMRuntimeMutator(
    cli client.Client,
    opts ...chaos.RuntimeMutatorOption,
) (string, error) {
    spec := chaos.GenerateRuntimeMutatorChaosSpec(opts...)

    schedule := &chaosmeshv1alpha1.Schedule{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "schedule-" + spec.Name,
            Namespace: spec.Namespace,
        },
        Spec: chaosmeshv1alpha1.ScheduleSpec{
            Schedule: "@every 1m",
            Type:     chaosmeshv1alpha1.ScheduleTypeRuntimeMutatorChaos,
            ScheduleItem: chaosmeshv1alpha1.ScheduleItem{
                RuntimeMutatorChaos: &spec.Spec,
            },
        },
    }

    err := cli.Create(context.Background(), schedule)
    return schedule.Name, err
}
```

## Layer 7: chaos-experiment Chaos Wrapper

### Step 7.1: Create Option Functions

Create `chaos/jvm_runtime_mutator.go`:

```go
package chaos

import (
    chaosmeshv1alpha1 "github.com/chaos-mesh/chaos-mesh/api/v1alpha1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type RuntimeMutatorOption func(*RuntimeMutatorChaosSpec)

type RuntimeMutatorChaosSpec struct {
    Name      string
    Namespace string
    Spec      chaosmeshv1alpha1.RuntimeMutatorChaosSpec
}

func WithRuntimeMutatorAction(action string) RuntimeMutatorOption {
    return func(spec *RuntimeMutatorChaosSpec) {
        spec.Spec.Action = chaosmeshv1alpha1.RuntimeMutatorChaosAction(action)
    }
}

func WithRuntimeMutatorClass(class string) RuntimeMutatorOption {
    return func(spec *RuntimeMutatorChaosSpec) {
        spec.Spec.RuntimeMutator.Class = class
    }
}

func WithRuntimeMutatorMethod(method string) RuntimeMutatorOption {
    return func(spec *RuntimeMutatorChaosSpec) {
        spec.Spec.RuntimeMutator.Method = method
    }
}

func WithRuntimeMutatorFrom(from string) RuntimeMutatorOption {
    return func(spec *RuntimeMutatorChaosSpec) {
        spec.Spec.RuntimeMutator.From = &from
    }
}

func WithRuntimeMutatorTo(to string) RuntimeMutatorOption {
    return func(spec *RuntimeMutatorChaosSpec) {
        spec.Spec.RuntimeMutator.To = &to
    }
}

func WithRuntimeMutatorStrategy(strategy string) RuntimeMutatorOption {
    return func(spec *RuntimeMutatorChaosSpec) {
        spec.Spec.RuntimeMutator.Strategy = &strategy
    }
}

func WithRuntimeMutatorPort(port int32) RuntimeMutatorOption {
    return func(spec *RuntimeMutatorChaosSpec) {
        spec.Spec.RuntimeMutator.Port = &port
    }
}

func GenerateRuntimeMutatorChaosSpec(opts ...RuntimeMutatorOption) *RuntimeMutatorChaosSpec {
    spec := &RuntimeMutatorChaosSpec{
        Name:      "jrm-" + uuid.NewString()[:8],
        Namespace: "default",
        Spec: chaosmeshv1alpha1.RuntimeMutatorChaosSpec{
            ContainerSelector: chaosmeshv1alpha1.ContainerSelector{
                PodSelector: chaosmeshv1alpha1.PodSelector{
                    Mode: chaosmeshv1alpha1.OneMode,
                },
            },
        },
    }

    for _, opt := range opts {
        opt(spec)
    }

    return spec
}
```

## Layer 8: AegisLab Integration

### Automatic Integration via chaos-experiment

AegisLab automatically supports new fault types registered in chaos-experiment through its generic fault injection flow. No AegisLab code changes are required if:

1. The fault type is registered in `ChaosTypeMap`, `SpecMap`, and `ChaosHandlers`
2. The spec struct has proper struct tags for parameter ranges
3. The `Create()` method is implemented

The generic flow in `fault_injection.go`:

```go
func executeFaultInjection(task *Task) error {
    // 1. Convert task payload to chaos.Node tree
    node := chaos.ParseTaskPayload(task.Payload)

    // 2. Convert Node to typed InjectionConf
    conf := chaos.NodeToStruct[chaos.InjectionConf](node)

    // 3. Get display config for groundtruth
    displayConfig := conf.GetDisplayConfig()

    // 4. Get groundtruth info
    groundtruth := conf.GetGroundtruth()

    // 5. Create chaos via registered handler
    chaosName, err := chaos.BatchCreate(conf, k8sClient)

    return err
}
```

### Frontend Integration

Add the fault type to `FaultTypePanel.tsx`:

```typescript
const categoryMap: Record<string, string> = {
    // ... existing mappings ...
    'JVMRuntimeMutator': 'JVM',
};

const faultTypeIcons: Record<string, React.ReactNode> = {
    // ... existing icons ...
    jvm: <BugOutlined />,
};

const faultTypeColors: Record<string, string> = {
    // ... existing colors ...
    jvm: 'magenta',
};
```

## Testing New Fault Types

### Unit Tests

Create tests for each layer:

```go
// handler test
func TestJVMRuntimeMutatorSpec_Create(t *testing.T) {
    spec := &JVMRuntimeMutatorSpec{
        Duration:     5,
        MutationType: "constant",
        MutationFrom: "100",
        MutationTo:   "0",
    }

    name, err := spec.Create(fakeClient, WithNamespace("test"))
    require.NoError(t, err)
    require.NotEmpty(t, name)
}

// controller test
func TestCreateJVMRuntimeMutatorChaos(t *testing.T) {
    name, err := CreateJVMRuntimeMutatorChaos(
        fakeClient,
        WithRuntimeMutatorAction("constant"),
        WithRuntimeMutatorClass("com.example.Test"),
        WithRuntimeMutatorMethod("getValue"),
        WithRuntimeMutatorFrom("100"),
        WithRuntimeMutatorTo("0"),
    )
    require.NoError(t, err)
    require.NotEmpty(t, name)
}
```

### Integration Tests

Test end-to-end with a real cluster:

```bash
# Apply CRD directly
kubectl apply -f - <<EOF
apiVersion: chaos-mesh.org/v1alpha1
kind: RuntimeMutatorChaos
metadata:
  name: test-constant-mutation
  namespace: test
spec:
  mode: one
  selector:
    namespaces:
      - test
    labelSelectors:
      app: java-app
  action: constant
  runtimeMutator:
    class: com.example.Calculator
    method: add
    from: "100"
    to: "0"
  duration: "60s"
EOF

# Verify chaos status
kubectl get runtimemutatorchaos test-constant-mutation -n test -o yaml
```

## Troubleshooting

### CRD Not Found

```bash
# Check CRD is installed
kubectl get crd | grep runtimemutatorchaos

# If missing, install via Helm
helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh
```

### Agent Injection Failed

Check chaosdaemon logs:

```bash
kubectl logs -n chaos-mesh -l app.kubernetes.io/component=chaos-daemon --tail=100
```

Common issues:
- Java process not found in container
- Agent JAR not available
- JVM attach API disabled

### Mutation Not Taking Effect

1. Verify class/method names are correct
2. Check agent logs in target container
3. Ensure class is loaded after agent injection (or use retransform)

### Recovery Issues

The recovery mechanism uses jattach to disable mutations. If this fails:
- Restart the target pod
- Check if jattach is available in chaos-daemon container

## Best Practices

1. **Follow existing patterns**: Use JVMChaos as a reference for JVM-level faults
2. **Validate thoroughly**: Add webhook validation for all required fields
3. **Handle errors gracefully**: Recovery should not fail if agent is not running
4. **Document parameters**: Add descriptions to struct tags for UI tooltips
5. **Test incrementally**: Test each layer independently before integration
6. **Use proper struct tags**: `range`, `dynamic`, and `description` tags enable automatic UI generation

## Related Documentation

- [RuntimeMutatorChaos CRD Reference](#runtimemutatorchaos-crd-reference)
- [Fault Types Reference](../fault-types)
- [Chaos Mesh Helm Chart Release](../../../../chaos-mesh/RELEASE.md)
- [Handler Node Format](../handler-node-format)
