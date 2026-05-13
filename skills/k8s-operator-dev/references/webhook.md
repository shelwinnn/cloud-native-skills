# Webhook 开发

Webhook 位于 API Server admission 路径上，必须快速、幂等、可解释。简单字段约束优先使用 CRD OpenAPI validation 或 CEL；webhook 留给复杂默认值、不可变字段、跨字段校验和少量运行时检查。

## Defaulting

Defaulting 只能补默认值，不应调用外部系统或产生副作用。多次调用结果必须一致。

```go
func (r *MyApp) Default() {
    if r.Spec.Replicas == nil {
        r.Spec.Replicas = ptr.To[int32](1)
    }
    if r.Labels == nil {
        r.Labels = map[string]string{}
    }
    if _, ok := r.Labels["app.kubernetes.io/name"]; !ok {
        r.Labels["app.kubernetes.io/name"] = r.Name
    }
}
```

## Validation

`ValidateCreate`、`ValidateUpdate`、`ValidateDelete` 的语义不同。不可变字段通常在 `ValidateUpdate` 中比较 old/new。

```go
func (r *MyApp) ValidateUpdate(old runtime.Object) (admission.Warnings, error) {
    oldObj := old.(*MyApp)
    if r.Spec.StorageClass != oldObj.Spec.StorageClass {
        return nil, field.Forbidden(
            field.NewPath("spec").Child("storageClass"),
            "storageClass 创建后不可修改",
        )
    }
    return r.validate()
}
```

使用 `field.Error` 能让 `kubectl` 展示具体字段：

```go
func (r *MyApp) validate() (admission.Warnings, error) {
    var allErrs field.ErrorList
    if r.Spec.Image == "" {
        allErrs = append(allErrs, field.Required(field.NewPath("spec").Child("image"), "必须设置镜像"))
    }
    if len(allErrs) > 0 {
        return nil, allErrs.ToAggregate()
    }
    return nil, nil
}
```

## Warnings 与 Errors

- `warnings` 用于提示废弃字段、推荐迁移路径或非阻断风险。
- `error` 用于非法输入、不可变字段变更、明确无法接受的配置。
- 外部服务不可用时，只有在业务明确要求强一致校验时才阻断请求；否则更适合作为 warning 或交给 Reconcile 更新 status。

## 注册

```go
func (r *MyApp) SetupWebhookWithManager(mgr ctrl.Manager) error {
    return ctrl.NewWebhookManagedBy(mgr).
        For(r).
        Complete()
}
```

确认 `main.go` 调用了 `SetupWebhookWithManager`，并且 webhook marker 与 `config/webhook` 生成结果一致。

## 什么时候不用 Webhook

- 单字段范围、枚举、长度、格式：用 OpenAPI validation。
- 简单跨字段关系：优先用 CEL。
- 需要长期轮询或访问外部系统的校验：放到 Reconcile，结果写 status。

## 证书管理

Webhook 必须通过 HTTPS 提供服务，API Server 需要信任 webhook 的 CA 证书。生产环境有三种证书管理方案：

### cert-manager（推荐）

在 `ValidatingWebhookConfiguration` / `MutatingWebhookConfiguration` 上添加 annotation，cert-manager 自动注入 CA bundle 并管理证书轮换：

```yaml
annotations:
  cert-manager.io/inject-ca-from: my-operator-system/my-operator-serving-cert
```

配合 `Certificate` 资源声明 webhook 服务证书。

### controller-runtime certwatcher

controller-runtime 内置 `certwatcher`，从文件系统加载证书并热重载。适合有外部证书分发机制的环境。在 `main.go` 中配置 webhook server 的证书路径。

### Kubebuilder 内置方案

使用 `kubebuilder init --plugins go/v4` 时，生成的项目默认包含 cert-manager 集成。不要手动管理自签名证书——证书过期会导致所有涉及 webhook 的 API 请求失败。
