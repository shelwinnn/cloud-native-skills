# Webhook 审查

审查目标：确认 webhook 只承担 API admission 该承担的职责，可靠、快速、幂等，并给用户清晰错误。

## 严重问题

- Validating webhook 发现非法配置只打日志或 warning，没有返回 error 拒绝请求。
- Mutating webhook 的 `Default()` 不幂等，多次调用会追加重复字段或产生副作用。
- Webhook 中调用外部服务且无短超时，导致 API Server admission 阻塞。
- 只实现 `ValidateCreate`，忽略 update/delete，导致不可变字段或删除保护被绕过。

## 高风险

- 不可变字段在 defaulting 中被修改，而不是在 `ValidateUpdate` 中校验。
- 错误没有使用 `field.Error`，用户看不到具体字段路径。
- 简单字段约束用了 webhook，而不是 CRD OpenAPI/CEL，增加部署和证书复杂度。
- webhook failure policy、timeout、side effects 语义与业务风险不匹配。

## 中风险

- 废弃字段没有通过 warnings 提示迁移路径。
- webhook 证书管理或 manager 注册缺失，代码存在但运行时不生效。
- admission 错误信息暴露内部实现、secret 名称以外的敏感内容。

## 修复方向

- 简单校验放到 CRD validation/CEL；复杂跨字段、不可变或运行时约束才用 webhook。
- `Default()` 只填默认值，不做外部调用，不依赖时间或随机数。
- `ValidateCreate/ValidateUpdate/ValidateDelete` 按操作分别覆盖。
- 返回 `field.Invalid`、`field.Forbidden` 或 `field.Required`，让 `kubectl` 指向具体字段。

## 证书管理审查

Webhook 必须通过 HTTPS 提供服务，API Server 需要信任 webhook 的 CA 证书。证书管理不当是 webhook Operator 在生产中最常见的故障来源。

### 审查要点

- **cert-manager 集成**：`ValidatingWebhookConfiguration` / `MutatingWebhookConfiguration` 上是否有 `cert-manager.io/inject-ca-from` annotation。配合 `Certificate` 资源声明 webhook 服务证书，cert-manager 自动注入 CA bundle 并管理证书轮换。

```yaml
annotations:
  cert-manager.io/inject-ca-from: my-operator-system/my-operator-serving-cert
```

- **controller-runtime certwatcher**：使用内置 `certwatcher` 从文件系统加载证书并热重载时，`main.go` 中是否正确配置了 webhook server 的证书路径。
- **自签名证书**：是否手动管理自签名证书。手动管理容易遗漏续期，证书过期会导致所有涉及 webhook 的 API 请求失败。
- **Kubebuilder 默认方案**：使用 `kubebuilder init --plugins go/v4` 生成的项目默认包含 cert-manager 集成，检查是否被意外移除。
- **证书过期监控**：是否有机制监控证书过期时间，避免证书过期后 webhook 静默失败。

### 严重问题

- Webhook 配置引用的 CA bundle 与实际证书不匹配，导致 API Server 无法连接 webhook。
- 证书管理方案缺失（没有 cert-manager 也没有 certwatcher），依赖手动证书管理且无续期机制。
- `failurePolicy: Ignore` 掩盖证书问题，导致 webhook 静默失效而不报错。
