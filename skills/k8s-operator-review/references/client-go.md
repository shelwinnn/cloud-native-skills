# client-go 审查

审查目标：确认直接使用 `client-go` 或混用 controller-runtime client 时，代码正确处理缓存一致性、分页、watch、informer、workqueue、冲突、限流和 dynamic client 风险。

## Client 类型和生命周期

### 严重问题

- 热路径重复创建 `rest.Config`、`Clientset`、`DynamicClient` 或 HTTP transport，绕过连接复用和限流。
- 直接拼 URL 调 Kubernetes API，绕过 discovery、serializer、auth 和错误语义。
- 忽略 `kubernetes.NewForConfig`、`dynamic.NewForConfig` 等初始化错误。

### 高风险

- 未设置可识别 `UserAgent`，生产排查 API Server 请求来源困难。
- 不理解 controller-runtime client 读 cache、写 API Server 的差异，在写后立即依赖 cache 读到新值。
- 混用 lister 返回对象和写 client，但修改前没有 `DeepCopy`，可能污染 informer cache。

## API 错误语义

- 使用 `apierrors.IsNotFound`、`IsAlreadyExists`、`IsConflict`、`IsForbidden`、`IsUnauthorized` 判断错误。
- 不要字符串匹配错误文本。
- 不要无条件吞错；临时错误应返回给 workqueue/rate limiter。
- Forbidden/Unauthorized 通常是配置或权限问题，应该清楚暴露，而不是无限静默重试。

## Create/Update/Patch/Conflict

### 严重问题

- `Update` 整对象覆盖用户或其他 controller 管理的字段。
- status、scale、finalizers 等 subresource 被普通 Update 顺手修改。
- Conflict 后继续使用过期对象重试，没有重新 Get 或 Patch。

### 修复方向

- 更新少量字段用 merge patch。
- 管理复杂 owned object 用 Server-Side Apply，并设置稳定 field owner。
- 必须读改写时，用 `retry.RetryOnConflict` 包裹最新对象读取和更新。

## List/Watch/Informer

### 严重问题

- Reconcile 热路径中跨 namespace 全量 List 高基数资源。
- 大规模 List 不分页，不设置 selector，忽略错误。
- 自定义 Watch 不处理 `410 Gone` 或 resourceVersion 过旧。
- worker 在 informer cache sync 前启动。

### 高风险

- event handler 直接执行业务逻辑、外部调用或大量 API 写操作，而不是只入队 key。
- 未处理 tombstone/`DeletedFinalStateUnknown`。
- 为每个对象创建 informer，导致 watch 连接和 goroutine 爆炸。
- resync 被误用为可靠定时任务。

### 修复方向

- 读取路径优先使用 informer/lister 或 controller-runtime cache。
- 大集合 List 使用 `Limit`/`Continue`，并加 label/field selector。
- handler 使用 `cache.MetaNamespaceKeyFunc` 或 `DeletionHandlingMetaNamespaceKeyFunc` 后入队。
- 启动 worker 前等待 `cache.WaitForCacheSync`。

## Workqueue 和 Backpressure

- 使用 rate-limiting queue；失败 `AddRateLimited`，成功 `Forget`，退出 `ShutDown`。
- 不用手写 `for + sleep` 重试，它会阻塞 worker 且没有统一 backoff。
- 限制最大重试次数或把永久错误写入 status，避免单个坏对象无限刷屏。
- `rest.Config.QPS/Burst`、worker 数量和外部系统限流要协调，避免错误风暴。

## Dynamic Client、RESTMapper、Scheme

### 严重问题

- `GroupVersionResource` 写错 plural 或 scope，却没有错误处理。
- 对 unstructured 字段做链式类型断言，字段缺失或 JSON number 类型变化会 panic。
- 忘记注册 scheme，导致 decode、owner reference、event recorder 或测试行为异常。

### 高风险

- CRD 安装/升级后没有处理 RESTMapper/discovery cache 过期。
- 把 GVK 和 GVR 混用。
- 对 CRD 使用 strategic merge patch；CRD 不支持这种 patch 语义。

## Cache 优化与内存控制

controller-runtime 默认缓存所有 watched GVK 的完整对象。在大规模集群中（数万 Pod、数百 Namespace），内存占用会显著增长。审查时关注以下优化手段是否被合理使用：

- `cache.TransformStripManagedFields`：剥离 `managedFields`，通常占对象体积的 30-50%。在不需要 SSA field ownership 信息时推荐启用。

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    NewCache: cache.BuilderWithOptions(cache.Options{
        DefaultTransform: cache.TransformStripManagedFields,
    }),
})
```

- `ByObject` 按类型和 namespace 限制 cache 范围：只缓存 Operator 实际管理的 namespace 或资源类型，减少内存和 watch 连接开销。

```go
cache.Options{
    ByObject: map[client.Object]cache.ByObject{
        &corev1.ConfigMap{}: {
            Namespaces: map[string]cache.Config{
                "my-operator-ns": {},
            },
        },
    },
}
```

- Metadata-only informer：只缓存 `metav1.PartialObjectMetadata`，不需要对象 spec/status 时大幅降低内存。适用于只关心 label/annotation/ownerReference 的场景。
- 偶尔读取用 `APIReader`（uncached client），不走 cache。适合低频管理操作（如读 Secret），避免为一次性读取维护 informer。
- 使用 `SharedInformerFactory` 共享 informer，同一 GVK 不要创建多个 informer 实例。

### 审查要点

- 在管理数百以上 CR 或跨多个 namespace 的 Operator 中，是否配置了 cache scoping 或 strip managedFields。
- 是否存在只为一次性读取而 watch 整个 GVK 的情况，可以改为 `APIReader`。
- 多个 controller 或多个 informer factory 是否对同一 GVK 创建了重复 informer。

## API Server 压力与 Backpressure

大规模集群中 Operator 的 List/Watch 和写操作可能对 API Server 造成显著压力。审查时关注：

- **启动风暴**：controller 启动时全量 List 所有 watched 资源。如果 Operator watch 高基数资源（如 Pod、EndpointSlice），多个副本同时启动会形成 API Server 压力峰值。配合 leader election 可以避免多副本同时启动。
- **informer 范围**：未配置 namespace selector 或 label selector 的 informer 会 watch 集群全部对象。审查 watch 配置是否覆盖了不必要的资源。
- **Reconcile 频率**：高频 status 写入或无意义的 Requeue 触发大量写请求。status 只在语义变化时更新，`RequeueAfter` 间隔要合理。
- **metrics 检查**：是否有 `workqueue_depth`、`reconcile_duration_seconds`、`client_api_latency` 等指标暴露，用于发现 API Server 压力问题。
- `rest.Config.QPS/Burst` 设置是否与集群规模匹配。默认 QPS=20/Burst=30 在高并发 Operator 中可能不够。

## client-go 最低质量门槛

- client 在启动阶段创建并复用，错误全部处理。
- 读路径理解 cache 一致性，修改 cache/lister 对象前 DeepCopy。
- List/Watch 对大规模集群有 selector、分页或 informer 方案。
- 写路径使用 Patch/Apply 或安全读改写，并处理 Conflict。
- workqueue 有 backoff、shutdown 和生命周期管理。
- 测试覆盖 NotFound、AlreadyExists、Conflict、Forbidden、cache sync、tombstone、分页和重试中的关键路径。
- 大规模 Operator 有 cache 优化和 API Server 压力控制措施。
