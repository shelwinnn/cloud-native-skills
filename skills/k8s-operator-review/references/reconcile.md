# Reconcile 审查

审查目标：确认 controller 是 level-based controller，而不是事件处理器。它必须能从任意中间状态恢复并收敛。

## 严重问题

- Reconcile 依赖 create/update/delete 事件类型或内存状态判断业务路径；事件可能丢失、合并或在重启后消失。
- 主资源 NotFound 返回错误；删除后的正常情况会变成无意义重试。
- 创建子资源前不检查现有状态，或无 owner reference，导致重复创建或孤儿资源。
- Operator 主动改写用户 `spec`，再用普通 `Update` 提交，可能覆盖用户期望状态。
- 对永久性错误直接 `return err`，导致无限重试和日志/API 压力。
- Reconcile 中 `time.Sleep` 或启动无生命周期 goroutine，阻塞 worker 或泄漏。

## 高风险

- 更新对象时整对象 `Update`，没有处理 conflict，也没有分析字段所有权。
- 无条件 `Requeue: true` 或每次都写 status，形成热循环。
- 外部 API 调用没有 timeout、context、稳定 ID 或幂等语义。
- `MaxConcurrentReconciles > 1` 时 Reconciler 中有请求级共享状态。
- watch 设置不完整，owned resource 变化不会触发 owner 重新入队。

## 中风险

- 缺少 predicate 过滤，status 或 metadata 噪声触发大量 reconcile。
- 日志缺少 namespace/name、操作对象、错误上下文。
- 没有 Kubernetes Event，用户只能查 controller 日志才能理解失败。
- 重试间隔和外部系统轮询没有解释，可能太频繁或太慢。

## Owner Reference 审查

- 专属于某个 CR 的集群内子资源必须设置 controller owner reference（`SetControllerReference`），CR 删除时 GC 自动清理。没有 owner reference 的子资源在 owner 删除后成为孤儿。
- 跨 namespace 不能设置 owner reference（Kubernetes 不支持跨 namespace owner reference）；跨 namespace 子资源必须用 finalizer 手动清理。
- 共享资源（多个 CR 引用同一 ConfigMap、Secret 等）不要设置 controller owner reference，否则第一个 owner 删除会把共享资源一起回收。共享资源用 label 或 annotation 标记归属关系。
- `SetControllerReference` 会检查 owner 和 child 的 scope 是否一致：namespace-scoped CR 不能成为 cluster-scoped 资源的 owner。忽略这个错误会导致静默跳过。
- 只管理集群内子资源时，owner reference + GC 是首选方案，不需要 finalizer。给不需要 finalizer 的场景加 finalizer 是不必要的复杂度。

### 高风险

- 创建子资源时没有调用 `SetControllerReference`，或只设了 `ownerReference` 但没设 `controller: true`，导致 GC 不触发。
- 同一个子资源被多个 CR 设为 controller owner，`SetControllerReference` 会返回错误但代码没有处理。
- 使用 owner reference 但子资源跨 namespace，运行时静默跳过设置。

## 永久错误处理

- 对永久性配置错误（如 spec 中引用的镜像不存在、外部 API 返回 400 Bad Request）不应无限重试。无限重试会刷屏日志并浪费 workqueue 容量。
- 两种处理方式：
  1. 写入 status condition 后返回 `nil`（不重试），用户通过 status 看到失败原因并修改 spec。
  2. 使用 `reconcile.TerminalError(err)` 标记永久错误，controller-runtime 不会重试 terminal error。
- 审查时要确认代码区分了临时错误和永久错误，且永久错误有明确的用户可操作的恢复路径。

```go
// 方式 1：写入 status 后不重试
if isPermanentConfigError(err) {
    base := obj.DeepCopy()
    r.setCondition(obj, "Ready", metav1.ConditionFalse, "InvalidSpec", err.Error())
    _ = r.patchStatus(ctx, base, obj)
    return ctrl.Result{}, nil
}

// 方式 2：Terminal Error（controller-runtime 不重试）
if isPermanentConfigError(err) {
    return ctrl.Result{}, reconcile.TerminalError(err)
}
```

## 必查调用形态

```go
if err := r.Get(ctx, req.NamespacedName, obj); err != nil {
    return ctrl.Result{}, client.IgnoreNotFound(err)
}
```

```go
base := child.DeepCopy()
mutate(child)
if err := r.Patch(ctx, child, client.MergeFrom(base)); err != nil {
    return ctrl.Result{}, err
}
```

如果代码偏离这些形态，不一定就是错，但需要证明它同样满足幂等、冲突安全和字段所有权清晰。
