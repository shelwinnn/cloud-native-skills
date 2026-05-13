# 生产就绪审查

审查目标：确认 Operator 在生产环境中可排障、可恢复、可扩展，并且管理外部系统时不会造成重复副作用。

## 外部资源管理

### 严重问题

- 外部资源创建没有稳定 ID 或幂等键，重复 Reconcile 会创建重复资源。
- 外部资源删除没有 finalizer 保护，CR 删除后资源泄漏。
- 外部 API 调用没有 context/timeout，Reconcile worker 可能长期阻塞。
- leader 切换或 controller 重启后，外部操作无法从 status 或实际状态恢复。

### 高风险

- 外部系统永久错误和临时错误没有区分，导致无限重试或错误吞掉。
- 外部资源状态只存在内存中，重启后丢失。
- 批量同步启动时对所有对象并发调用外部 API，没有限流。

## 可观测性

### 高风险

- 日志缺少 namespace/name、operation、resourceVersion、外部资源 ID 或错误上下文。
- 错误只存在日志中，status condition 和 Kubernetes Event 没有反映用户可见状态。
- 高频路径每次 Reconcile 都发 Event，污染事件流。
- 没有暴露或检查 workqueue depth、reconcile latency、error count、external API latency 等指标。

### 中风险

- Event reason 不稳定，自动化系统难以识别。
- 日志中包含 secret、token、证书、连接串或完整请求体。
- health/readiness 只表示进程存活，不反映 cache sync 或依赖初始化失败。

## 测试与升级

- envtest 覆盖 status subresource、webhook、CRD validation、finalizer 和 owner reference。
- fake client 测试不应被当作 API server 语义证明。
- 测试覆盖重复 Reconcile、部分失败后重试、删除中断后恢复、Conflict、Forbidden 和外部 NotFound。
- CRD 升级要检查字段兼容性、conversion、默认值变化和旧对象读取。
- Fuzz testing：Reconcile 对非预期输入（异常 spec 值、边界 condition、极端 label/annotation）不应 panic、无限循环或产生非预期副作用。Go 1.18+ 原生 fuzz testing 适合用 fake client 验证。

### Fuzz 审查要点

- Reconcile 入口是否有 fuzz 覆盖，使用 `testing.F` 添加种子输入后验证不 panic。
- Fuzz 种子是否覆盖空值、零值、极端值（负数 replicas、超长字符串、特殊字符）。
- Fuzz 测试使用 fake client 运行，不依赖 envtest。重点不是验证正确性，而是确认 Reconcile 不会崩溃。

### kstatus 与 GitOps 兼容性

如果 Operator 可能与 ArgoCD、FluxCD 等 GitOps 工具集成，审查 status 设计是否兼容 kstatus 规范：

- 是否提供 `Ready` 或 `Available` condition，kstatus 用它判断资源健康状态。
- `ObservedGeneration` 是否正确设置为 `Generation`，kstatus 依赖它判断 controller 是否已处理最新 spec。
- Condition 状态是否符合 kstatus 语义：`True` 表示就绪，`False` 表示异常，`Unknown` 表示判断中。
- 不兼容 kstatus 会导致 GitOps 工具无法自动判断部署健康，需要用户手动配置 health check。
