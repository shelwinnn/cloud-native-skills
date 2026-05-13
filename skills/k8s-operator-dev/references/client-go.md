# client-go 与 controller-runtime client

controller-runtime client 的读写路径不同：`Get/List` 默认从 informer cache 读，`Create/Update/Patch/Delete` 直接写 API Server。这个模型带来高性能，也带来最终一致性和缓存污染风险。

## Client 选择

- controller-runtime Reconciler 中优先使用 `client.Client`，因为它与 manager cache、scheme、watch 语义一致。
- 访问 Kubernetes 内置资源且需要原生 informer/lister/workqueue 时，使用 typed clientset。
- 访问未注册到 scheme 的 CRD 时，可使用 dynamic client 或 `unstructured.Unstructured`，但要显式处理 GVR、scope 和字段类型。
- 避免在热路径重复创建 `rest.Config`、`Clientset` 或 transport；这些对象应在启动阶段创建并复用。

## 缓存一致性

- cache 读是最终一致的；写入后立即 `Get` 可能读到旧对象。
- 对强一致读取有硬要求时，使用 `mgr.GetAPIReader()` 注入的 uncached reader，但要限制频率。
- 使用原生 lister 返回的对象前，如果要修改，必须 `DeepCopy`；否则可能污染 informer cache。
- 如果开启 `UnsafeDisableDeepCopy`，controller-runtime client 读出的对象也要按 cache 引用处理。

```go
obj, err := r.PodLister.Pods(ns).Get(name)
if err != nil {
    return err
}
copy := obj.DeepCopy()
copy.Labels["managed-by"] = "my-operator"
```

## List 与索引

- 热路径不要无条件跨 namespace 全量 List。
- 能通过 owner、label、field 限定范围时，使用 `InNamespace`、`MatchingLabels`、`MatchingFields`。
- 大规模原生 client-go List 使用 `Limit` 和 `Continue` 分页。
- 复杂关联查询先注册 `FieldIndexer`，再按索引查询。

```go
const ownerKey = ".metadata.controller"

if err := mgr.GetFieldIndexer().IndexField(ctx, &appsv1.Deployment{}, ownerKey, func(obj client.Object) []string {
    owner := metav1.GetControllerOf(obj)
    if owner == nil {
        return nil
    }
    return []string{owner.Name}
}); err != nil {
    return err
}
```

原生 client-go 大规模 List 示例：

```go
opts := metav1.ListOptions{
    LabelSelector: "app=my-app",
    Limit:         500,
}
for {
    list, err := clientset.CoreV1().Pods(ns).List(ctx, opts)
    if err != nil {
        return err
    }
    for i := range list.Items {
        process(&list.Items[i])
    }
    if list.Continue == "" {
        break
    }
    opts.Continue = list.Continue
}
```

## Patch 与 Apply

- 更新自己拥有的少量字段时，优先 `client.MergeFrom(base)`。
- 管理复杂对象且需要 field ownership 时，使用 Server-Side Apply，并设置稳定 `FieldOwner`。
- 使用 `Update` 做读改写时，必须处理 `Conflict`，通常用 `retry.RetryOnConflict` 重新读取最新对象。
- status、scale、finalizers 是不同 subresource，不要用普通 `Update` 顺手修改。

```go
base := obj.DeepCopy()
obj.Labels["app.kubernetes.io/managed-by"] = "my-operator"
if err := r.Patch(ctx, obj, client.MergeFrom(base)); err != nil {
    return err
}
```

## Informer、Lister、Workqueue

- 启动 worker 前等待 `cache.WaitForCacheSync`，否则会基于不完整数据做决策。
- informer handler 只入队 key，不直接调用外部系统或大量写 API Server。
- 删除事件使用 `cache.DeletionHandlingMetaNamespaceKeyFunc`，处理 tombstone。
- 成功处理 key 后调用 `Forget`，失败后调用 `AddRateLimited`，退出时 `ShutDown`。
- 不要为每个对象创建 informer；同类资源应通过 shared informer factory 共享 watch。

workqueue 处理骨架：

```go
func (c *Controller) processNextItem(ctx context.Context) bool {
    key, shutdown := c.queue.Get()
    if shutdown {
        return false
    }
    defer c.queue.Done(key)

    if err := c.sync(ctx, key); err != nil {
        c.queue.AddRateLimited(key)
        return true
    }

    c.queue.Forget(key)
    return true
}
```

informer handler 只入队，不做业务：

```go
informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        if key, err := cache.MetaNamespaceKeyFunc(obj); err == nil {
            queue.Add(key)
        }
    },
    DeleteFunc: func(obj interface{}) {
        if key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj); err == nil {
            queue.Add(key)
        }
    },
})
```

## Dynamic Client 风险

- `GroupVersionResource` 要用 plural resource，并区分 namespaced 与 cluster-scoped。
- 读取 unstructured 字段使用 `unstructured.Nested*`，不要链式类型断言。
- CRD 安装或升级后，RESTMapper/discovery 可能过期；代码要能处理 `NoMatchError` 并刷新映射。
- Patch 时确认 patch 类型适用于目标资源；strategic merge patch 不适用于 CRD。

安全读取 unstructured 字段：

```go
replicas, found, err := unstructured.NestedInt64(obj.Object, "spec", "replicas")
if err != nil {
    return fmt.Errorf("read spec.replicas: %w", err)
}
if !found {
    replicas = 1
}
```

## Cache 优化与内存控制

默认 cache 会存储所有 watch 资源的完整对象，大集群下内存开销显著。以下策略可以控制 cache 内存占用。

### 剥离 managedFields

`metadata.managedFields` 可能占对象内存的 30-50%，大多数 controller 不需要这个字段。在 Manager 初始化时全局剥离：

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Cache: cache.Options{
        DefaultTransform: cache.TransformStripManagedFields(),
    },
})
```

### 按类型和 namespace 限制 cache 范围

不需要全集群 watch 的资源类型，通过 `ByObject` 限定 cache 的 namespace 和 label selector：

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Cache: cache.Options{
        ByObject: map[client.Object]cache.ObjectCacheConfig{
            &corev1.ConfigMap{}: {
                Namespaces: map[string]cache.Config{
                    "my-operator-ns": {},
                },
                LabelSelector: labels.SelectorFromSet(labels.Set{"managed-by": "my-operator"}),
            },
        },
    },
})
```

### Metadata-only informer

如果 controller 只需要对象的 metadata（name、labels、annotations、ownerReferences），不缓存 spec/status 可以大幅减少内存：

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Cache: cache.Options{
        ByObject: map[client.Object]cache.ObjectCacheConfig{
            &corev1.Pod{}: {Metadata: true},
        },
    },
})
// 查询时使用 PartialObjectMetadata
podMeta := &metav1.PartialObjectMetadata{}
podMeta.SetGroupVersionKind(corev1.SchemeGroupVersion.WithKind("Pod"))
r.List(ctx, podMeta)
```

### 偶尔读取的资源用 APIReader

如果某种资源只在特定场景偶尔读取（如启动时读一次 ConfigMap），不要让 cache 为它建立 informer。使用 `mgr.GetAPIReader()` 直接请求 API Server：

```go
r.APIReader.Get(ctx, key, &corev1.ConfigMap{})
```

### 共享 Informer

使用原生 client-go 时，同类资源必须通过 `SharedInformerFactory` 共享 watch 连接，不要各自创建 informer：

```go
factory := informers.NewSharedInformerFactory(clientset, time.Minute)
fooInformer := factory.MyGroup().V1().Foos().Informer()
factory.Start(stopCh)
```

## QPS、Burst 与生命周期

- `rest.Config.QPS/Burst` 要与 worker 数量、外部系统限流和 API Server 容量匹配。
- 不要每个 goroutine 创建自己的 client；这会绕过统一限流并制造连接风暴。
- Watch、informer、worker 必须跟随 context 或 stop channel 退出。
- 自定义 watch 要处理 resourceVersion 过旧；能用 informer 时不要手写 watch 循环。
