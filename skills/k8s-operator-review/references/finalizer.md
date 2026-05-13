# Finalizer 审查

审查目标：确认删除流程不会泄漏外部资源，也不会让 CR 永远卡在 `Terminating`。

## 严重问题

- Operator 创建外部资源但没有 finalizer；CR 删除后外部资源可能泄漏。
- finalizer 在外部资源创建之后才添加；如果中间失败，删除时没有机会清理。
- cleanup 不幂等；外部资源已不存在时仍返回错误，导致 finalizer 永远无法移除。
- cleanup 成功后移除了内存中的 finalizer，但没有持久化 `Update/Patch`。
- 删除分支没有优先返回，导致对象删除期间继续创建或更新资源。

## 高风险

- cleanup 错误被记录后吞掉，finalizer 被移除，导致外部资源泄漏。
- 外部清理没有 timeout/context，删除请求可能长期卡住。
- finalizer 名称不是 domain-qualified，容易与其他 controller 冲突。
- finalizer 用于纯 Kubernetes owned resources；这种场景通常 owner reference + GC 足够。

## 中风险

- 删除进度没有写入 status 或 event，用户无法判断卡在哪里。
- 多个 finalizer 被同一个 controller 管理，增加排序和失败处理复杂度。
- 清理逻辑依赖可能已被 GC 删除的子资源，没有容错。

## 修复方向

- 先持久化 finalizer，再创建外部资源。
- cleanup 对 NotFound/AlreadyDeleted 视为成功，对临时错误返回 error 让 workqueue 重试。
- 清理完成后移除 finalizer 并持久化。
- 删除过程中的长期等待用 condition/event 暴露给用户。
