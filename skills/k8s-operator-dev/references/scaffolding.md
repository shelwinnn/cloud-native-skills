# Kubebuilder 脚手架

脚手架只负责生成基础结构，业务语义仍然要按 API 契约和 Reconcile 收敛模型补齐。不要把生成代码当成生产完成态。

## 初始化项目

```bash
kubebuilder init \
  --domain example.com \
  --repo github.com/org/my-operator \
  --plugins go/v4
```

## 创建 API 和 Controller

```bash
kubebuilder create api \
  --group apps \
  --version v1alpha1 \
  --kind MyApp \
  --resource \
  --controller
```

## 创建 Webhook

```bash
kubebuilder create webhook \
  --group apps \
  --version v1alpha1 \
  --kind MyApp \
  --defaulting \
  --validation
```

## 典型目录

```text
api/v1alpha1/
  myapp_types.go
  myapp_webhook.go
  groupversion_info.go
internal/controller/
  myapp_controller.go
  myapp_controller_test.go
config/
  crd/
  rbac/
  webhook/
cmd/main.go
```

## 生成和检查

```bash
# 运行 controller-gen object，为所有 API 类型生成 DeepCopy、DeepCopyInto 等方法
make generate

# 运行 controller-gen crd:crdVersions=v1,rbac:roleName=manager-role,webhook，
# 从 Go marker 生成 CRD YAML、ClusterRole 和 WebhookConfiguration
make manifests

# 运行全部测试（包括 envtest）
go test ./...
```

检查生成结果时重点看：

- CRD 是否包含 status subresource、validation、printcolumn、scale subresource。
- RBAC 是否包含主资源、`/status`、`/finalizers`、owned resources、events、leases。
- Webhook marker 是否覆盖 create/update/delete 需要的操作。
- controller 是否注册到 manager，API types 是否 AddToScheme。

每次修改 API types、RBAC marker 或 webhook marker 后，都要重新运行 `make generate && make manifests` 并检查 diff。
