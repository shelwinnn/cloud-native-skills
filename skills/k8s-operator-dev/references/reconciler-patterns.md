# Reconcile 模式

Reconcile 的职责不是处理某个事件，而是在任何时刻都能从当前状态收敛到期望状态。事件只负责触发，业务逻辑必须基于状态差异，能承受重复执行、进程重启、API 冲突和部分失败。

## 标准主路径

```go
func (r *MyAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var obj appsv1alpha1.MyApp
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    if !obj.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, &obj)
    }

    if !controllerutil.ContainsFinalizer(&obj, myFinalizer) {
        return ctrl.Result{}, r.patchFinalizers(ctx, &obj, func(copy *appsv1alpha1.MyApp) {
            controllerutil.AddFinalizer(copy, myFinalizer)
        })
    }

    if err := r.reconcileDeployment(ctx, &obj); err != nil {
        base := obj.DeepCopy()
        r.setCondition(&obj, "Ready", metav1.ConditionFalse, "DeploymentFailed", err.Error())
        _ = r.patchStatus(ctx, base, &obj)
        return ctrl.Result{}, err
    }

    base := obj.DeepCopy()
    obj.Status.ObservedGeneration = obj.Generation
    r.setCondition(&obj, "Ready", metav1.ConditionTrue, "Reconciled", "所有资源已就绪")
    return ctrl.Result{}, r.patchStatus(ctx, base, &obj)
}
```

## Status 子资源更新

Status 是 controller 对当前状态的观测结果，不能用普通 `Update` 顺手写入。普通 `Update` 会发送整个对象，可能覆盖用户刚刚改过的 `spec`，也会让 status 与 spec 的权限边界混在一起。

### 推荐：Status Patch

```go
func (r *MyAppReconciler) patchStatus(ctx context.Context, base, obj *appsv1alpha1.MyApp) error {
    if reflect.DeepEqual(base.Status, obj.Status) {
        return nil
    }
    return r.Status().Patch(ctx, obj, client.MergeFrom(base))
}
```

调用时必须先保存 base，再修改 status：

```go
base := obj.DeepCopy()
obj.Status.ObservedGeneration = obj.Generation
meta.SetStatusCondition(&obj.Status.Conditions, metav1.Condition{
    Type:               "Ready",
    Status:             metav1.ConditionTrue,
    Reason:             "Reconciled",
    Message:            "所有资源已就绪",
    ObservedGeneration: obj.Generation,
})
return r.patchStatus(ctx, base, &obj)
```

### 什么时候用 Status Update

- 可以在简单 controller 中使用 `Status().Update`，但要接受更高的 conflict 概率。
- 如果 status 更新失败，不要静默忽略；至少要返回错误或记录为可解释的非关键失败。
- 不要每次 Reconcile 都写 status。先比较新旧 status，只在语义变化时写，避免 status 写入反复触发 Reconcile。
- `lastTransitionTime` 只在 condition status 变化时更新；使用 `meta.SetStatusCondition` 可以避免手写错误。

## Conflict 处理

Kubernetes 使用 `resourceVersion` 做乐观锁。任何 `Update` 都可能因为对象已被用户、API server 或其他 controller 修改而 conflict。正确处理方式是重新基于最新对象计算变更，而不是在旧对象上盲目重试。

### 优先使用 Patch

```go
base := obj.DeepCopy()
obj.Labels["app.kubernetes.io/managed-by"] = "my-operator"
if err := r.Patch(ctx, &obj, client.MergeFrom(base)); err != nil {
    return ctrl.Result{}, err
}
```

### 必须 Update 时使用 RetryOnConflict

```go
func (r *MyAppReconciler) updateSpecOwnedFields(ctx context.Context, key client.ObjectKey, mutate func(*appsv1alpha1.MyApp)) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &appsv1alpha1.MyApp{}
        if err := r.Get(ctx, key, latest); err != nil {
            return err
        }
        mutate(latest)
        return r.Update(ctx, latest)
    })
}
```

### Status Conflict

```go
func (r *MyAppReconciler) updateStatusWithRetry(ctx context.Context, key client.ObjectKey, mutate func(*appsv1alpha1.MyApp)) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        latest := &appsv1alpha1.MyApp{}
        if err := r.Get(ctx, key, latest); err != nil {
            return err
        }
        base := latest.DeepCopy()
        mutate(latest)
        if reflect.DeepEqual(base.Status, latest.Status) {
            return nil
        }
        return r.Status().Patch(ctx, latest, client.MergeFrom(base))
    })
}
```

### Finalizer Conflict

Finalizer 是 metadata 的一部分，多个 controller 可能同时修改。添加或移除 finalizer 时也要用 patch 或 `RetryOnConflict`。

```go
func (r *MyAppReconciler) patchFinalizers(ctx context.Context, obj *appsv1alpha1.MyApp, mutate func(*appsv1alpha1.MyApp)) error {
    base := obj.DeepCopy()
    mutate(obj)
    if reflect.DeepEqual(base.Finalizers, obj.Finalizers) {
        return nil
    }
    return r.Patch(ctx, obj, client.MergeFrom(base))
}
```

## 子资源管理

使用 `controllerutil.CreateOrUpdate` 时，mutation 函数只设置自己拥有的字段。共享对象或多 controller 协作对象更适合 Server-Side Apply。

```go
_, err := controllerutil.CreateOrUpdate(ctx, r.Client, deploy, func() error {
    if err := controllerutil.SetControllerReference(owner, deploy, r.Scheme); err != nil {
        return err
    }
    deploy.Labels = desiredLabels(owner)
    deploy.Spec = desiredDeploymentSpec(owner)
    return nil
})
```

Server-Side Apply 适合复杂 owned object，但要使用稳定 field owner，并谨慎使用 `ForceOwnership`，因为它会抢占其他 actor 的字段所有权。

```go
return r.Patch(ctx, deploy, client.Apply,
    client.FieldOwner("myapp-controller"),
    client.ForceOwnership,
)
```

## Owner Reference 与垃圾回收

Kubernetes 垃圾回收（GC）通过 owner reference 自动删除子资源。正确使用 owner reference 可以避免大多数 finalizer 场景。

- 专属于某个 CR 的子资源使用 `controllerutil.SetControllerReference` 设置 controller owner reference，CR 删除时 GC 自动清理子资源。
- 跨 namespace 不能设置 owner reference（Kubernetes 不支持跨 namespace 的 owner reference），这类场景必须用 finalizer 手动清理。
- 共享资源（多个 CR 同时引用）不要设置 controller owner reference，否则第一个 owner 删除时会把共享资源一起回收。共享资源用 label 或 annotation 标记归属关系。
- `controllerutil.SetControllerReference` 会检查 owner 和 child 的 scope 是否一致：namespace-scoped CR 不能成为 cluster-scoped 资源的 owner。
- 只管理集群内子资源时，owner reference + GC 是首选方案，不需要 finalizer。

```go
deploy := &appsv1.Deployment{
    ObjectMeta: metav1.ObjectMeta{
        Name:      obj.Name,
        Namespace: obj.Namespace,
    },
}
if err := controllerutil.SetControllerReference(obj, deploy, r.Scheme); err != nil {
    return err
}
```

### 何时不用 Owner Reference

- 子资源跨 namespace（如 CR 在 namespace A，需要在 namespace B 创建资源）。
- 子资源被多个 CR 共享（如 ConfigMap 包含多个 CR 的公共配置）。
- 子资源的生命周期独立于 CR（CR 删除后子资源应继续存在）。

这些场景用 finalizer + label 管理生命周期。

## 外部资源管理

管理 Kubernetes 集群外部的资源（云存储、外部数据库、DNS 记录等）需要额外注意：

- 外部资源必须使用稳定标识符（从 CR 名称或 spec 派生），不要每次 Reconcile 生成随机名称或 ID，否则重试时会创建重复资源。
- 外部副作用发生前必须先持久化 finalizer；否则进程崩溃会导致外部资源泄漏且无法追踪。
- 区分临时错误和永久错误：网络超时是临时的，404 Not Found 在清理场景下是成功的，400 Bad Request 可能是永久配置错误。
- Credentials 通过 Secret 引用，读取后不缓存到进程内存，不在日志或 status 中暴露。
- 重启后必须能恢复状态：外部资源 ID 记录在 status 中，或可从 CR spec 稳定推导。

```go
func (r *MyAppReconciler) reconcileExternal(ctx context.Context, obj *appsv1alpha1.MyApp) error {
    // 使用稳定 ID（从 CR 派生，不是随机值）
    externalID := fmt.Sprintf("myapp-%s-%s", obj.Namespace, obj.Name)

    err := r.ExternalClient.Create(ctx, externalID, obj.Spec.Config)
    if err != nil {
        if isAlreadyExists(err) {
            return nil // 幂等：已存在视为成功
        }
        if isPermanentError(err) {
            // 永久错误写入 status，不无限重试
            base := obj.DeepCopy()
            r.setCondition(obj, "Ready", metav1.ConditionFalse, "InvalidConfig", err.Error())
            _ = r.patchStatus(ctx, base, obj)
            return nil
        }
        return fmt.Errorf("creating external resource %s: %w", externalID, err)
    }
    return nil
}
```

## 错误和重试

- 临时错误返回 `error`，交给 workqueue rate limiter 做指数退避。
- 永久性配置错误不要无限重试；写入 status condition 后返回成功，或使用 controller-runtime 支持的 terminal error。
- `RequeueAfter` 只用于外部系统轮询、证书续期、定期检查等不会由 watch 触发的场景。
- 不要在 Reconcile 中 `time.Sleep`，它会占住 worker 并破坏吞吐。

```go
if err := external.Create(ctx, obj.Spec.Name); err != nil {
    if isInvalidSpec(err) {
        base := obj.DeepCopy()
        r.setCondition(obj, "Ready", metav1.ConditionFalse, "InvalidSpec", err.Error())
        return ctrl.Result{}, r.patchStatus(ctx, base, obj)
    }
    return ctrl.Result{}, fmt.Errorf("creating external resource: %w", err)
}
```

### Terminal Error

controller-runtime 提供了 `reconcile.TerminalError`，专门标记永久性错误。Terminal error 不会被 workqueue 重试，比手动写入 status 再返回成功更语义化。

```go
import "sigs.k8s.io/controller-runtime/pkg/reconcile"

if isPermanentConfigError(err) {
    return ctrl.Result{}, reconcile.TerminalError(
        fmt.Errorf("invalid spec: image %q does not exist", obj.Spec.Image),
    )
}
```

## Finalizer

- 在创建外部资源之前添加并持久化 finalizer。
- 删除分支必须尽早处理，避免对象删除时继续创建资源。
- 清理逻辑必须幂等；外部资源已不存在应视为成功。
- 清理成功后移除 finalizer 并持久化。

```go
func (r *MyAppReconciler) handleDeletion(ctx context.Context, obj *appsv1alpha1.MyApp) (ctrl.Result, error) {
    if !controllerutil.ContainsFinalizer(obj, myFinalizer) {
        return ctrl.Result{}, nil
    }
    if err := r.cleanupExternal(ctx, obj); err != nil {
        return ctrl.Result{}, fmt.Errorf("cleanup external resource: %w", err)
    }
    return ctrl.Result{}, r.patchFinalizers(ctx, obj, func(copy *appsv1alpha1.MyApp) {
        controllerutil.RemoveFinalizer(copy, myFinalizer)
    })
}
```

## Watch 与 predicate

- 对主资源通常使用 `GenerationChangedPredicate`，避免 status 写入反复触发主路径。
- 对 owned resources 使用 `Owns`，让子资源变化能回到 owner。
- 对非 owner 资源使用 map function 把资源映射回相关 CR。
- event handler 或 map function 不做重业务逻辑，只计算需要入队的 key。

```go
return ctrl.NewControllerManagedBy(mgr).
    For(&appsv1alpha1.MyApp{}, builder.WithPredicates(predicate.GenerationChangedPredicate{})).
    Owns(&appsv1.Deployment{}).
    Watches(&corev1.ConfigMap{}, handler.EnqueueRequestsFromMapFunc(r.mapConfigMapToApps)).
    Complete(r)
```
