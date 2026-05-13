# 可观测性与运行特性

Operator 的可观测性要服务两个对象：用户通过 CR status 和 Events 理解资源状态，运维通过日志和 metrics 判断 controller 是否健康。

## 日志

- 日志包含 namespace、name、operation、owned resource、外部资源 ID 和错误上下文。
- 不记录 secret、token、证书、连接串或完整敏感请求体。
- 高频路径只记录状态变化或异常，不要每次 Reconcile 都刷 Info。

```go
log := log.FromContext(ctx).WithValues("myapp", req.NamespacedName.String())
log.Info("创建 Deployment", "deployment", deploy.Name)
log.Error(err, "创建 Deployment 失败", "deployment", deploy.Name)
```

## Kubernetes Events

Events 面向用户排障，适合记录生命周期里程碑和可行动失败。不要在每次 Reconcile 都发相同 event。

```go
r.Recorder.Event(&obj, corev1.EventTypeNormal, "Reconciled", "资源已完成同步")
r.Recorder.Eventf(&obj, corev1.EventTypeWarning, "DeploymentFailed", "Deployment 同步失败: %v", err)
```

需要在 Reconciler 中注入 recorder：

```go
type MyAppReconciler struct {
    client.Client
    Scheme   *runtime.Scheme
    Recorder record.EventRecorder
}

r.Recorder = mgr.GetEventRecorderFor("myapp-controller")
```

## Metrics

controller-runtime 默认暴露 reconcile 次数、错误数和耗时。业务指标只记录用户真正会用来判断容量、健康或外部资源状态的信号。

```go
var readyReplicas = prometheus.NewGaugeVec(prometheus.GaugeOpts{
    Name: "myapp_ready_replicas",
    Help: "MyApp 当前 ready 副本数",
}, []string{"namespace", "name"})

func init() {
    metrics.Registry.MustRegister(readyReplicas)
}
```

## Health、Readiness 和 Cache Sync

- liveness 表示进程还能运行。
- readiness 应该反映 manager 已启动、cache 已同步、关键依赖初始化完成。
- 不要把外部业务系统短暂失败直接变成 operator 不 ready；否则会导致重启风暴。

## Leader Election

多副本部署且 Reconcile 有写操作或外部副作用时启用 leader election。即使启用了 leader election，Reconcile 仍必须幂等，因为 leader 切换、重试和重复事件仍会发生。

确认 RBAC 包含：

```go
// +kubebuilder:rbac:groups=coordination.k8s.io,resources=leases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=events,verbs=create;patch
```
