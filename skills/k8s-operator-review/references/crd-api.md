# CRD/API 审查

审查目标：判断 CRD 是否像 Kubernetes 原生 API 一样稳定、可验证、可演进，而不是把 controller 内部状态暴露给用户。

## 严重问题

- 缺少 `// +kubebuilder:subresource:status`，但 controller 写 `status`；这会导致 status 无法按子资源语义更新。
- `spec` 和 `status` 混用，例如 `spec.phase`、`spec.ready`、`status.desiredReplicas`；这会模糊用户期望和实际状态。
- 关键字段没有 validation，非法输入会进入 Reconcile 后形成永久错误循环。
- 已发布字段发生不兼容变化，例如改 JSON tag、可选改必填、收窄枚举、改变字段语义。

## 高风险

- 可选字段没有清晰零值语义；需要表达“不设置”和“设置为空”差异时没有用指针。
- 任意 `map[string]interface{}`、`interface{}` 或无结构 JSON 被用于核心配置，导致 schema 约束和升级兼容性不可控。
- 有 `replicas` 语义但 scale subresource 缺失，用户无法使用 `kubectl scale` 或 HPA 类工具集成。
- API version 命名不符合 `v1alpha1`、`v1beta1`、`v1` 约定，或多版本 CRD 没有 conversion 策略。

## 中风险

- 缺少 printcolumn，`kubectl get` 无法展示 Ready、Phase、Age 等关键信息。
- 公开字段缺少注释，生成的 CRD description 对用户没有帮助。
- 字段名过于泛化，如 `type`、`mode`、`state`，但没有枚举或文档解释。

## API 版本策略审查

- 多版本 CRD 必须有 conversion 策略（`None`、`Webhook`），不能停留在无策略的多版本状态。
- 有且仅有一个版本标记 `+kubebuilder:storageversion`，存储版本不明确会导致 etcd 数据混乱。
- Conversion webhook（Hub/Spoke 模式）中 Hub 必须是存储版本，Spoke 到 Hub 和 Hub 到 Spoke 的转换函数必须完整测试。
- 版本升级路径要验证：旧版本对象在新版本 controller 下是否正确工作，降级是否安全。
- 已废弃版本不能直接删除，需要经过废弃期和迁移文档。

## Structural Schema 审查

- CRD 必须是 structural schema（不含 `x-kubernetes-preserve-unknown-fields` 在非必要位置），否则会失去 schema 校验、protobuf 序列化和 `kubectl explain` 支持。
- 字段类型必须明确：`integer`、`string`、`boolean`、`object`、`array` 不能混用或省略。
- `+kubebuilder:validation:Pattern` 使用正则时，要检查正则复杂度不会导致 ReDoS。
- `preserve-unknown-fields` 只应用于有意保留用户自定义字段的场景（如 `metadata.annotations`、嵌入式 `RawExtension`），滥用会破坏 schema 校验。

## Condition 命名规范

- Condition `Type` 使用 PascalCase，语义明确，如 `Ready`、`Available`、`Progressing`、`Degraded`。避免使用 `Status`、`State`、`Phase` 等过于泛化的名称。
- Condition `Reason` 使用 CamelCase，描述发生的原因而非状态本身，如 `Progressing`、`DeploymentFailed`、`InvalidSpec`。避免 `Error`、`Failed`、`OK` 等无信息量的 Reason。
- `Message` 是面向人类的可读描述，可以包含细节但不依赖 machine-parseable 信息。
- Condition `Status` 只有 `True`/`False`/`Unknown`，不要自定义枚举值。
- `ObservedGeneration` 必须设为 `obj.Generation`（不是 `obj.ResourceVersion`），用于判断 controller 是否处理了最新 spec。

### kstatus 兼容性

如果 Operator 可能与 ArgoCD、FluxCD 等 GitOps 工具集成，condition 设计应兼容 kstatus 规范：

- 提供 `Ready` 或 `Available` condition，kstatus 用它判断资源健康状态。
- `ObservedGeneration` 与 `Generation` 不一致时，kstatus 认为资源尚未收敛。
- Condition `Status=True` 表示就绪，`False` 表示异常，`Unknown` 表示正在判断。
- 不满足 kstatus 时，GitOps 工具无法自动判断部署健康，需要用户手动配置 health check。

## 修复方向

- 把用户意图放在 `spec`，把观测结果放在 `status`。
- 为所有关键字段添加 `Required`、`Enum`、`Minimum`、`MaxLength`、`Pattern` 或 CEL。
- 使用 `metav1.Condition` 表达状态，并配合 `observedGeneration`。
- 需要破坏性变更时新增 API version 并实现 conversion，而不是直接修改已发布字段。
- Condition Type 和 Reason 遵循命名规范，确保工具链可解析。
