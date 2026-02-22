# 零碎片成本打造私有 GitLab Runner CI

这篇是「云原生 CI/CD」系列的首发，主题是“零碎片成本”的私有 GitLab 快速起步：选择 GitLab Runner 构建首个云原生流水线，在内网环境中完成从仓库、CI、镜像到发布的闭环，后续再补充 GitHub Actions 的多云对照。

## 1. 为什么选择 GitLab 作为云原生私有仓库

私有 GitLab 不仅提供代码仓库，还自带 CI/CD、容器注册表、Issue、Security Dashboard 等一体化能力，适合想把构建、测试、镜像和发布全部收入囊中的团队。与 GitHub 相比，GitLab 的自托管版对企业网络更友好，对接内网私有镜像库也更容易。加上 GitLab Runner 与 GitLab Server 的原生集成，可以用最少的 glue 就实现流水线自动化。

## 2. GitLab Runner 与 Jenkins 的对比

| 维度 | GitLab Runner | Jenkins |
|------|---------------|---------|
| 所属生态 | GitLab 原生，Pipeline 写在 `.gitlab-ci.yml` 中 | 独立项目，通过插件与 SCM 连接 |
| CI 表达 | Pipeline/Job+Stage 一次性描述 | Freestyle + Pipeline + Groovy DSL，多种模式混用 |
| 扩容能力 | runner 可基于 Kubernetes 自动扩展 Pod | 需要自己管理 executor 节点或通过主机池 |
| 插件/集成 | GitLab 提供的 Docker/Helm/安全扫描即开即用 | 插件生态丰富，但版本兼容风险高 |
| 授权模型 | GitLab 用户即拥有 Pipeline 权限 | 需要额外配置 Jenkins 权限模型 |
| 维护成本 | 只需维护 GitLab Server + Runner | 需要维护 Jenkins Master、Plugin、Worker |
| 安全隔离 | Kubernetes Executor 运行在 Job Namespace，和宿主隔离 | 脚本通常在主节点执行，需严格限制 |
| 可观测性 | GitLab CI 自带 pipeline 可视化 | 需依赖 Blue Ocean 或插件 |

> 小结：对私有云原生团队而言，GitLab Runner 的上手门槛更低、可扩展性更好；而 Jenkins 仍适合跨多个仓库、已有大量插件依赖的老平台。我们的第一篇 CI 就从 GitLab Runner 起步，未来再看是否引入 GitHub Actions 做多云对照。

## 3. GitLab + Higress 架构部署概述

为了让 GitLab 在 Kubernetes 中可靠地暴露出对外界的访问，需要一个现代化的 Ingress 控制器。因为官方的 Ingress NGINX 已宣布退役，所以推荐使用阿里云开源的 **Higress**，它对 Ingress API 100% 兼容、与 Gateway API 演进兼容，也已经在同目录的
`ingress nginx已废，我们应该选择这个控制器` 里详细介绍了 Higress 的安装与准备步骤，务必先完成那篇的部署，再在下面的 GitLab 示例中引用 Higress 的 ingress class。

部署这套私有 GitLab 的关键构件包括：

1. Kubernetes 集群（可用 3 个 master + 若干 worker）
2. 外部访问域名，例如 `gitlab.example.com`，需要指向 Higress LoadBalancer
3. 存储类（StorageClass）以满足 PostgreSQL、Redis、Object Storage 的要求
4. Higress Ingress Controller（参考前文）提供 `ingressClass=higress`
5. TLS 证书（可以借助 cert-manager + ACME）

## 4. GitLab Server 部署步骤

1. **创建命名空间与添加 Helm 源**

```bash
kubectl create namespace gitlab
helm repo add gitlab https://charts.gitlab.io
helm repo update
```

2. **准备 `production-values.yaml`**

```yaml
global:
  edition: ce
  hosts:
    domain: gitlab.example.com
    externalIP: ""
  ingress:
    configureCertmanager: false
    class: higress
    tls: false

certmanager:
  install: false

gitlab:
  webservice:
    ingress:
      annotations:
        kubernetes.io/ingress.class: higress
        higress.io/backend-protocol: HTTP
```

- `class: higress` 配置告知 GitLab 生成的 Ingress 资源由 Higress 接管。
- 如果已经准备了 TLS Secret，可以关闭 `configureCertmanager` 并手动引用 `certmanager.email` + `tls.secretName`。

3. **部署 GitLab**

```bash
helm upgrade --install gitlab gitlab/gitlab \
  --namespace gitlab \
  -f production-values.yaml
```

4. **等待所有 Pod 就绪**

```bash
kubectl -n gitlab get pods
```

5. **访问与初始化**

- 打开 `https://gitlab.example.com`，根据 Helm 输出设置初始密码
- 在 GitLab UI 中准备项目、组、注册 Runner 所需的 token

## 5. GitLab Runner 部署步骤

1. **创建命名空间**

```bash
kubectl create namespace gitlab-runner
```

2. **准备 `runner-values.yaml`**（将 `runnerRegistrationToken` 中的值从 GitLab 的 `Settings → CI/CD → Runners` 拷贝）

```yaml
gitlabUrl: https://gitlab.example.com/
runnerRegistrationToken: "<替换成你的 token>"
rbac:
  create: true
runners:
  privileged: true
  config: |
    [[runners]]
      executor = "kubernetes"
      [runners.kubernetes]
        namespace = "gitlab-runner"
        image = "alpine:latest"
        poll_timeout = 600
  cache:
    cacheType: "s3"
    s3Bucket: "gitlab-runner-cache"
    s3ServerAddress: "minio.gitlab.svc.cluster.local:9000"
```

3. **安装 Runner**

```bash
helm upgrade --install gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab-runner \
  -f runner-values.yaml
```

4. **确认 Runner 状态**

```bash
kubectl -n gitlab-runner get pods
```

在 GitLab 的 CI/CD Runner 面板里，你会看到新的 Runner 被注册并处于 `online` 状态。Runner 会自动从 `gitlabUrl` 拉取 `.gitlab-ci.yml`，在 Runner 命名空间内用 Kubernetes Executor 运行 Job，Job 运行的 Pod 可以被 Higress 控制的 Ingress 暴露，若需要调试 Job，可以通过 GitLab 提供的终端查看。

## 6. 后续计划与总结

- 这篇是「云原生 CI/CD」合集的第一张：「私有 GitLab + GitLab Runner」。
- 下一篇将聚焦 GitHub Actions，在不同平台之间比较触发方式与 Runner 运行环境。
- 如果你已有 Higress 部署，直接参考本篇的 Helm values 填写即可；还没部署的请先读 `_posts/ingress nginx已废，我们应该选择这个控制器.md`，完成 Higress 的部署与域名配置。

欢迎在公众号留言告诉我你们的私有 GitLab 架构，我会在后续的文章中补充常见的备份、监控与 GitOps 集成方案。
