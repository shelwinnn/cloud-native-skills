# RBAC 与安全

Operator 的权限模型直接影响集群安全。每个 RBAC marker 都是对 API Server 权限的声明，过宽的权限会扩大故障爆炸半径，过窄会导致运行时 Forbidden。

## 最小权限原则

- 禁止使用通配符权限（`groups=*`、`resources=*`、`verbs=*`）。
- 精确声明每个 verb：`r.Get` 对应 `get`，`r.List` 对应 `list`，`r.Watch`（informer）对应 `watch`，`r.Create`/`r.Update`/`r.Patch`/`r.Delete` 分别对应同名 verb。
- 对 Secret、TokenReview、SubjectAccessReview、admission webhook 配置等敏感资源格外谨慎。

```go
// +kubebuilder:rbac:groups=apps.example.com,resources=myapps,verbs=get;list;watch;create;update;patch;delete
// +kubebac:rbac:groups=apps.example.com,resources=myapps/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps.example.com,resources=myapps/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=secrets,verbs=get;list;watch
```

## 三组权限缺一不可

主资源、status 子资源、finalizers 子资源的 RBAC 必须同时声明。缺少 `/status` 会导致 `Status().Update/Patch` 静默失败或报 Forbidden；缺少 `/finalizers` 会导致 finalizer 添加/移除失败。

```go
// 主资源
// +kubebuilder:rbac:groups=apps.example.com,resources=myapps,verbs=get;list;watch;create;update;patch;delete
// status 子资源
// +kubebuilder:rbac:groups=apps.example.com,resources=myapps/status,verbs=get;update;patch
// finalizers 子资源
// +kubebuilder:rbac:groups=apps.example.com,resources=myapps/finalizers,verbs=update
```

## Leader Election 额外权限

多副本部署启用 leader election 时，需要 leases 和 events 权限：

```go
// +kubebuilder:rbac:groups=coordination.k8s.io,resources=leases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=events,verbs=create;patch
```

## ClusterRole vs Role

- namespace-scoped Operator 优先使用 Role + RoleBinding，通过 `DefaultNamespaces` 限制 cache 和权限范围。
- cluster-scoped Operator 才使用 ClusterRole + ClusterRoleBinding，且必须有明确理由。

```go
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Cache: cache.Options{
        DefaultNamespaces: map[string]cache.Config{
            "target-namespace": {},
        },
    },
})
```

## RBAC 与代码一致性

RBAC marker 声明的权限必须与代码实际使用的 API 调用一一对应。定期对照：

| 代码中的调用 | 需要的 verb |
| --- | --- |
| `r.Get` | `get` |
| `r.List` | `list` |
| informer / watch | `watch` |
| `r.Create` | `create` |
| `r.Update` | `update` |
| `r.Patch` | `patch` |
| `r.Delete` | `delete` |
| `r.Status().Update/Patch` | `/status` 子资源的 `update;patch` |

## 凭证处理

- 禁止硬编码密码、Token 或 API Key 到代码或注释中。
- 从 Secret 读取凭证后，不要将其写入日志、Event message 或 status 字段。
- 凭证只在需要时读取，不要缓存到进程内存长期持有。

```go
secret := &corev1.Secret{}
if err := r.Get(ctx, types.NamespacedName{Name: obj.Spec.SecretRef, Namespace: obj.Namespace}, secret); err != nil {
    return err
}
password := secret.Data["password"]
// 使用 password，但不写入日志或 status
```

## Pod 安全上下文

Operator 的 Deployment 应配置最小安全上下文：

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: manager
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
```

## 权限审计

定期使用以下命令审查 Operator 实际权限，与 RBAC marker 对比：

```bash
kubectl auth can-i --list --as=system:serviceaccount:my-ns:my-operator-sa
```
