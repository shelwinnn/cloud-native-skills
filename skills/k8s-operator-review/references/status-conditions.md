# Status/Conditions 审查

审查目标：确认 status 是用户和自动化系统可信的观测结果，不制造热循环，不误报旧 generation 的成功。

## 严重问题

- 使用普通 `Update` 更新 status；这可能覆盖 `spec`，也绕过 status subresource 语义。
- CRD 没有 status subresource，但 controller 依赖 status 写入。
- status 写入错误被静默吞掉，导致用户看到过期状态。
- 每次 Reconcile 都改 status 时间戳或 message，导致 status update 反复触发 reconcile。

## 高风险

- 缺少 `observedGeneration`，用户无法判断状态是否对应最新 `spec`。
- 只有单个 `Phase` 字符串，没有 Conditions，无法表达细粒度失败原因。
- `lastTransitionTime` 在 condition status 未变化时也更新，破坏状态历史。
- 成功 condition 在部分 owned resource 或外部资源未就绪时提前置 True。

## 中风险

- Condition `Type`、`Reason` 命名不稳定或不可机器识别。
- `Message` 只有原始错误或 “failed”，没有告诉用户该检查什么。
- Status 存储可从 owned resource 推导的大块副本，例如完整 PodStatus，容易过期和膨胀。

## 修复方向

- 使用 `Status().Patch` 或 `Status().Update`，不要普通 `Update`。
- 使用 `meta.SetStatusCondition`，避免清空重建整个 condition 列表。
- status 反映当前 spec 后设置 `ObservedGeneration = obj.Generation`。
- 错误 condition 的 `Reason` 稳定，`Message` 面向用户排障。
