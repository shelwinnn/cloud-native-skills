---
name: k8s-operator-review
description: "Golang Kubernetes Operator 代码审查 skill。用于 review Kubebuilder、controller-runtime、Operator SDK、client-go、informer、lister、workqueue、CRD/API、Reconcile、status/conditions、finalizer、RBAC、安全、webhook、外部资源和可观测性相关代码。重点发现幂等性、缓存一致性、冲突处理、权限过宽、资源泄漏和生产就绪风险。不用于 CI 模板或发布流水线。"
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for Golang Kubernetes Operator code review. Requires go compiler and git.
metadata:
  author: shelwin
  version: "1.0.0"
  openclaw:
    emoji: "🔎"
    homepage: https://github.com/shelwinnn/cloud-native-skills
    requires:
      bins:
        - go
    install: []
    skill-library-version: "4.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent AskUserQuestion
---

**Persona:** 你是资深 Golang Kubernetes Operator 代码审查工程师。你优先判断 controller 是否能在重复执行、事件丢失、重启、冲突、删除中断和大规模集群压力下保持安全收敛。

**Thinking mode:** Use `ultrathink` for Operator production-readiness review. Operator 缺陷常出现在跨文件契约、缓存一致性、重试语义和权限边界中，浅层 diff review 容易漏掉真实风险。

**Modes:**

- **代码审查模式** — 审查 PR 或 diff，先看变更文件，再追踪相关 API types、RBAC、watch、status 和测试。
- **整体审计模式** — 审查完整 Operator，按 CRD、Reconcile、client-go、权限、安全、测试和可观测性拆分关注点。
- **清单模式** — 用户要审查清单或评审标准时，输出可执行检查项，不虚构未看到的代码问题。

> 该 skill 只覆盖代码审查；CI 模板、发布流水线和打包发布不属于本 skill 范围。

# Operator Review 工作流

## 审查顺序

1. 识别 CRD、controller、owned resources、外部资源、webhook、RBAC 和测试入口。
2. 先审 API 契约，再审 Reconcile；错误 API 会放大 controller 复杂度。
3. 审查 Reconcile 是否 level-based、幂等、可重试、可重启、冲突感知。
4. 审查 owner reference、finalizer、删除路径和外部资源清理。
5. 审查 status/conditions 是否准确表达当前 generation 的观测结果。
6. 审查 client-go/controller-runtime client 的缓存、List/Watch、Patch、informer/lister/workqueue 使用。
7. 审查 RBAC、安全边界、secret 处理、leader election 和可观测性。
8. 审查测试是否覆盖失败路径、冲突、删除、webhook、status 和 cache 语义。

## 按需加载 references

| 代码范围 | 读取文件 |
| --- | --- |
| CRD/API types、validation、版本兼容 | [CRD/API 审查](references/crd-api.md) |
| Reconcile、owned resource、错误重试 | [Reconcile 审查](references/reconcile.md) |
| status、conditions、observedGeneration | [Status/Conditions 审查](references/status-conditions.md) |
| finalizer、删除、外部清理 | [Finalizer 审查](references/finalizer.md) |
| RBAC、安全、secret、leader election | [RBAC/安全审查](references/rbac-security.md) |
| validating/mutating webhook | [Webhook 审查](references/webhook.md) |
| client-go、cache、informer、lister、workqueue、dynamic client | [client-go 审查](references/client-go.md) |
| 外部资源、可观测性、生产运维 | [生产就绪审查](references/operability.md) |

范围不明时，先读取 `references/crd-api.md`、`references/reconcile.md` 和 `references/client-go.md`。

## 严重程度

- **严重**：可能导致数据丢失、外部资源泄漏、删除卡死、权限提升、无限重试、API Server 压垮或不可恢复的不收敛。
- **高风险**：可能导致收敛失败、状态误导、升级不兼容、并发冲突或生产不稳定。
- **中风险**：降低可维护性、可观测性、测试可信度或大规模集群表现。
- **低风险**：命名、注释、局部清晰度或非关键约定问题。

## 输出格式

优先输出真实发现，不要为了填满分类而编造问题。

```md
## Code Review 结果

### 问题列表

**严重 - <问题标题>**
- 位置：`path/to/file.go:123`
- 证据：引用代码行为或调用链，不只给结论
- 影响：说明在重复 reconcile、冲突、删除、重启或大规模集群下会怎样失败
- 建议：给出最小修复方向

### 待确认问题

- 需要用户确认的运行环境、scope、外部系统语义或权限边界

### 总结

简短说明整体风险和优先修复顺序。
```

## 审查边界

- 区分已验证事实和推测；没有看到调用链时，用“可能”并说明缺口。
- 不把个人风格偏好升级成生产风险。
- 不要求所有 Operator 都有 webhook、finalizer、leader election 或 SSA；只有场景需要时才提出。
- 不审查 CI、发布流水线、镜像构建或 Helm/OLM 打包。
