# Operator 测试

Operator 测试要覆盖 Kubernetes API 语义，而不是只验证 Go 函数返回值。fake client 很快，但不能替代 API server 行为。

## 测试分层

| 层级 | 适合验证 | 不适合验证 |
| --- | --- | --- |
| 纯单元测试 | spec 到 desired object 的转换、condition helper、错误分类 | cache、status subresource、webhook、CRD schema |
| fake client | 简单 Reconcile 分支、对象创建参数 | resourceVersion、真实 patch 语义、admission、managedFields |
| envtest | CRD validation、status subresource、webhook、controller 收敛 | 多节点调度、真实 kubelet 行为 |

## envtest 覆盖重点

- 创建 CR 后 owned resources 最终出现，并带正确 owner reference。
- 修改 `spec` 后，status 的 `observedGeneration` 最终更新。
- status 写入走 `/status` 子资源，不会改写 `spec`。
- 删除 CR 后 finalizer 清理完成，finalizer 被移除。
- webhook 拒绝非法输入，并给出字段级错误。
- Reconcile 重复执行不会重复创建资源或持续写 status。
- Conflict、Forbidden、AlreadyExists、外部 NotFound/timeout 等失败路径按语义处理。

```go
Eventually(func(g Gomega) {
    current := &appsv1alpha1.MyApp{}
    g.Expect(k8sClient.Get(ctx, client.ObjectKeyFromObject(myApp), current)).To(Succeed())
    cond := meta.FindStatusCondition(current.Status.Conditions, "Ready")
    g.Expect(cond).NotTo(BeNil())
    g.Expect(cond.ObservedGeneration).To(Equal(current.Generation))
}, timeout, interval).Should(Succeed())
```

多个断言要放在同一个 `Eventually(func(g Gomega) {...})` 中，让读取和断言一起重试：

```go
Eventually(func(g Gomega) {
    latest := &appsv1alpha1.MyApp{}
    g.Expect(k8sClient.Get(ctx, client.ObjectKeyFromObject(myApp), latest)).To(Succeed())
    g.Expect(latest.Status.ObservedGeneration).To(Equal(latest.Generation))
    g.Expect(meta.IsStatusConditionTrue(latest.Status.Conditions, "Ready")).To(BeTrue())
}, timeout, interval).Should(Succeed())
```

## fake client 使用边界

使用 fake client 时显式注册 scheme 和 status subresource：

```go
c := fake.NewClientBuilder().
    WithScheme(scheme).
    WithObjects(myApp).
    WithStatusSubresource(&appsv1alpha1.MyApp{}).
    Build()
```

不要用 fake client 证明以下行为，因为它与真实 API server 可能不一致：

- admission webhook 是否生效。
- CRD schema 是否拒绝非法字段。
- resourceVersion conflict 是否真实出现。
- informer cache 延迟和 watch 行为。
- Server-Side Apply field ownership。

## 必测失败路径

- 主资源 NotFound 时返回成功。
- owned resource NotFound 时会创建。
- AlreadyExists、Conflict、Forbidden、Invalid、外部系统 404/409/timeout 的处理。
- finalizer 清理过程中失败后会重试，外部资源已不存在时能继续完成删除。
- status patch 失败不会被静默吞掉，除非代码明确解释为什么可忽略。

## Webhook envtest

Webhook 需要 envtest 安装 webhook 配置和证书路径；不要用 fake client 证明 admission 行为。

```go
testEnv = &envtest.Environment{
    CRDDirectoryPaths: []string{filepath.Join("..", "..", "config", "crd", "bases")},
    WebhookInstallOptions: envtest.WebhookInstallOptions{
        Paths: []string{filepath.Join("..", "..", "config", "webhook")},
    },
}
```

测试 defaulting 时要重新读取对象，因为默认值由 API Server admission 写入：

```go
Expect(k8sClient.Create(ctx, myApp)).To(Succeed())
latest := &appsv1alpha1.MyApp{}
Expect(k8sClient.Get(ctx, client.ObjectKeyFromObject(myApp), latest)).To(Succeed())
Expect(*latest.Spec.Replicas).To(Equal(int32(1)))
```

## Conflict 测试思路

- 对 helper 函数注入返回 `apierrors.NewConflict` 的 fake client wrapper，验证会重新读取或返回可重试错误。
- 对使用 `Patch` 的路径，验证只修改目标字段，不覆盖已有 label、annotation、finalizer。
- 对 status 路径，验证 status 更新不改变 spec，并且无变化时不重复写。

## Fuzz Testing

Go 1.18+ 原生支持 fuzz testing，适合验证 Reconcile 对非预期输入的处理。Fuzz 测试用 fake client 运行，不依赖 envtest。

```go
func FuzzReconcile(f *testing.F) {
    // 种子输入
    f.Add("nginx:latest", int32(1))
    f.Add("", int32(0))
    f.Add("invalid image!@#", int32(-1))

    f.Fuzz(func(t *testing.T, image string, replicas int32) {
        // 跳过明显无效的输入（由 webhook/CEL 拦截）
        if image == "" || replicas < 0 || replicas > 100 {
            return
        }

        myApp := &appsv1alpha1.MyApp{
            ObjectMeta: metav1.ObjectMeta{Name: "fuzz", Namespace: "default"},
            Spec:       appsv1alpha1.MyAppSpec{Image: image, Replicas: &replicas},
        }

        c := fake.NewClientBuilder().
            WithScheme(scheme).
            WithObjects(myApp).
            WithStatusSubresource(&appsv1alpha1.MyApp{}).
            Build()

        r := &MyAppReconciler{Client: c, Scheme: scheme}
        _, err := r.Reconcile(context.Background(), reconcile.Request{
            NamespacedName: types.NamespacedName{Name: "fuzz", Namespace: "default"},
        })
        if err != nil {
            t.Errorf("reconcile returned unexpected error: %v", err)
        }
    })
}
```

Fuzz 测试的重点不是验证正确性，而是确认 Reconcile 不会 panic、不会无限循环、不会产生非预期副作用。
