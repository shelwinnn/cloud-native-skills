# CRD/API 设计

CRD 是 Operator 的长期兼容契约。先把 API 设计清楚，再写 Reconcile；否则 controller 代码会被错误的字段语义拖住。

## 结构原则

- `spec` 表达用户声明的期望状态，不放运行时状态、错误信息、进度、外部资源实际值。
- `status` 表达 controller 观测到的实际状态，不放用户意图。
- 需要状态回写时添加 `// +kubebuilder:subresource:status`。
- 可选字段使用指针或有明确零值语义的类型，并加 `omitempty`。
- 已发布字段不要随意改 JSON tag、含义、必填性或枚举范围；需要破坏性变化时新增 API version 并做 conversion。

## 推荐骨架

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Ready",type=string,JSONPath=`.status.conditions[?(@.type=="Ready")].status`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
type MyApp struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   MyAppSpec   `json:"spec,omitempty"`
    Status MyAppStatus `json:"status,omitempty"`
}

type MyAppSpec struct {
    // Image is the container image to run.
    // +kubebuilder:validation:MinLength=1
    Image string `json:"image"`

    // Replicas is the desired number of pods.
    // +kubebuilder:default:=1
    // +kubebuilder:validation:Minimum=0
    Replicas *int32 `json:"replicas,omitempty"`
}

type MyAppStatus struct {
    ObservedGeneration int64              `json:"observedGeneration,omitempty"`
    Conditions         []metav1.Condition `json:"conditions,omitempty"`
    ReadyReplicas      int32              `json:"readyReplicas,omitempty"`
}
```

## Validation 设计

- 字符串字段用 `MinLength`、`MaxLength`、`Pattern` 或 `Enum` 限制输入。
- 数值字段用 `Minimum`、`Maximum`，避免非法配置进入 Reconcile 后才失败。
- list/map 使用 `+listType=map`、`+listMapKey=<field>` 或 `+listType=set` 明确 merge 语义。
- Kubernetes 1.25+ 可用 CEL 表达跨字段约束和简单不可变字段。

```go
// +kubebuilder:validation:XValidation:rule="self.minReplicas <= self.maxReplicas",message="minReplicas must be <= maxReplicas"
type ScalingPolicy struct {
    MinReplicas int32 `json:"minReplicas"`
    MaxReplicas int32 `json:"maxReplicas"`
}
```

## Status 与 Conditions

- 至少提供 `Ready` condition；复杂资源可加 `Progressing`、`Degraded`、`Available` 或领域特定 condition。
- 每次 status 反映当前 spec 时设置 `ObservedGeneration = metadata.generation`。
- 使用 `meta.SetStatusCondition`，避免每次清空重建 conditions 导致 `lastTransitionTime` 抖动。

```go
meta.SetStatusCondition(&obj.Status.Conditions, metav1.Condition{
    Type:               "Ready",
    Status:             metav1.ConditionTrue,
    Reason:             "Reconciled",
    Message:            "All owned resources are ready",
    ObservedGeneration: obj.Generation,
})
```

## API 版本策略

CRD 一旦发布到生产环境，字段语义就是兼容性契约。破坏性变更必须通过新 API version 引入。

- 版本路径：`v1alpha1`（实验性，可破坏）→ `v1beta1`（稳定 schema，需 conversion webhook）→ `v1`（GA，永久向后兼容）。
- 恰好一个版本标记 `+kubebuilder:storageversion`，作为 etcd 中的存储版本。
- 多版本共存时使用 Hub/Spoke conversion 模式：存储版本实现 `conversion.Hub` 接口，其他版本实现 `ConvertFrom`/`ConvertTo`。

```go
// v1/zz_generated.conversion.go — Hub
func (*MyApp) Hub() {}

// v1alpha1/zz_generated.conversion.go — Spoke
func (dst *MyApp) ConvertFrom(srcRaw conversion.Hub) error {
    src := srcRaw.(*v1.MyApp)
    // 字段转换逻辑
    return autoConvert(src, dst)
}

func (src *MyApp) ConvertTo(dstRaw conversion.Hub) error {
    dst := dstRaw.(*v1.MyApp)
    return autoConvert(src, dst)
}
```

- Conversion webhook 需要在 `main.go` 中注册，与普通 webhook 一样需要证书管理。
- 已发布字段的 JSON tag、必填性、枚举范围不能缩小；需要删除或重命名字段时，在新版本中标记废弃并在旧版本中保留。

## Scale Subresource

如果 CRD 的 `spec` 中有 `replicas` 字段，声明 scale subresource 可以让用户使用 `kubectl scale` 操作：

```go
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.readyReplicas,selectorpath=.status.selector
```

只在资源确实有"副本数"语义时才启用。没有 replicas 概念的资源不需要这个 subresource。

## Structural Schema 规则

Kubernetes 要求所有 CRD schema 是 structural 的（每个字段都有明确类型）。违反会导致 CRD 创建失败或 kubectl 行为异常。

- 每个字段必须有类型声明。
- 不要在嵌套层级滥用 `x-kubernetes-preserve-unknown-fields`。
- 需要接受任意 JSON 的字段，使用 `runtime.RawExtension` 并标记 `+kubebuilder:pruning:PreserveUnknownFields`。

```go
// 接受任意 JSON 的字段
// +kubebuilder:pruning:PreserveUnknownFields
Config *runtime.RawExtension `json:"config,omitempty"`
```

- 不要用 `map[string]interface{}`，它会绕过 OpenAPI schema。如果确实需要灵活结构，用 `apiextensionsv1.JSON`。

## 常用字段模式

可复用的字段定义模式，减少重复设计：

```go
// 镜像引用
type ImageSpec struct {
    // +kubebuilder:validation:MinLength=1
    Repository string `json:"repository"`
    // +kubebuilder:default:="latest"
    Tag string `json:"tag,omitempty"`
    // +kubebuilder:validation:Enum=Always;Never;IfNotPresent
    // +kubebuilder:default:=IfNotPresent
    PullPolicy corev1.PullPolicy `json:"pullPolicy,omitempty"`
}

// 资源需求（直接复用 core 类型）
Resources corev1.ResourceRequirements `json:"resources,omitempty"`

// 服务引用
type ServiceRef struct {
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name"`
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=65535
    Port int32 `json:"port"`
}
```

## Condition 命名规范

Condition 的 Type 和 Reason 是机器可读标识符，命名不规范会导致监控告警和 GitOps 工具无法正确识别状态。

- **Type**：PascalCase，如 `Ready`、`Progressing`、`Degraded`、`Available`。不要用小写或混合格式。
- **Reason**：CamelCase，不含空格和下划线，如 `AllComponentsReady`、`DeploymentFailed`、`DatabaseUnavailable`。不要用 `"Error"` 这种过于笼统的 Reason。
- **Message**：面向人类的完整描述，包含足够上下文让用户能定位问题。不要只写 `"ok"` 或 `"error"`。

```go
// 标准条件类型常量
const (
    ConditionTypeReady       = "Ready"
    ConditionTypeProgressing = "Progressing"
    ConditionTypeDegraded    = "Degraded"
    ConditionTypeAvailable   = "Available"
)
```

### kstatus 兼容性

需要与 Argo CD / Flux 等 GitOps 工具集成时，建议遵循 [kstatus](https://github.com/kubernetes-sigs/cli-utils/tree/master/pkg/kstatus) 规范：

- `Ready=True`：资源完全就绪。
- `Reconciling=True`：正在处理中。
- `Stalled=True`：处于无法继续的错误状态。

## Webhook 取舍

- 能用 OpenAPI validation/CEL 表达的约束，不要优先写 webhook。
- Defaulting webhook 必须幂等，多次调用不能追加重复数据或产生副作用。
- Validating webhook 应返回 `field.Error`，让用户看到具体字段。
- Webhook 中避免调用外部系统；如必须调用，要设置短超时，并判断失败是否真的应该阻断 API 请求。
