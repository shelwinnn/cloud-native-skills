# RBAC/安全审查

审查目标：确认 Operator 权限最小化，secret 不泄漏，多副本运行安全，RBAC marker 与实际 API 调用一致。

## 严重问题

- ServiceAccount 绑定 `cluster-admin`，或 RBAC marker 使用 `groups=*,resources=*,verbs=*`。
- 对 Secrets 拥有不必要的 `list/watch/update/delete` 权限，且代码会记录 secret 内容。
- Operator Pod 以特权模式运行、允许权限提升，或不必要地挂载宿主机路径。
- Namespace-scoped Operator 使用 cluster-wide cache/RBAC，没有明确理由。

## 高风险

- 缺少 `<resource>/status` 或 `<resource>/finalizers` 权限，导致 status/finalizer 路径运行时失败。
- 多副本部署但没有 leader election，且 Reconcile 有外部副作用。
- leader election 打开但缺少 `leases` 权限。
- RBAC marker 与代码实际调用不一致，例如代码 `List/Watch` 资源但只给 `get`。
- 错误日志、event 或 status message 泄露 token、证书、密码、连接串。

## 中风险

- Role/ClusterRole 选择与 controller scope 不一致。
- 没有配置基础 Pod security context，例如 `runAsNonRoot`、`allowPrivilegeEscalation: false`、`readOnlyRootFilesystem`。
- 外部 API 凭证不支持轮换，或读取后缓存在不可控全局变量里。

## 权限对照

- `Get` 需要 `get`。
- `List` 需要 `list`。
- informer/watch 需要 `list;watch`。
- `Create` 需要 `create`。
- `Update` 需要 `update`。
- `Patch` 需要 `patch`。
- `Delete` 需要 `delete`.
- `Status().Update/Patch` 需要 `<resource>/status` 的 `update/patch`。
- finalizer 更新需要 `<resource>/finalizers` 的 `update`。
