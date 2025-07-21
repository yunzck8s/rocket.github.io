---
layout: post
title: "Github Actions + Flux CD 构建CICD自动化流水线"
date: 2023-01-13
tags: [devops, kubernetes, fluxcd, cicd, github-actions]
---

# Github actions + flux CD 构建CICD自动化流水线
<img src="./images/k8s.png" align='center' />
## 1.构建kubernetes集群（本次使用集群为GKE）

`GKE是GCP上的容器化平台，可以帮助我们托管kubernetes集群，不需要过多的去考虑维护集群的问题，包括sc之类的都是已经提供。`
`GCP link`: https://console.cloud.google.com

## 2.创建GitHub 仓库

`Flux CD` 是 `gitOps` 产品，所以操作都是围绕着git 来进行。
这个仓库不需要手动创建，flux CD 会自动的帮助你创建，在此之前需要提供你的github username以及 `Personal access tokens`。

`Personal access tokens` 的位置于GitHub的个人的 `settings/Developer settings` 下，创建完成注意妥善保管。

```shell
## install flux command
# mac
brew install fluxcd/tap/flux

# linux
curl -s https://fluxcd.io/install.sh | sudo bash

#windows
choco install flux
##

# To configure your shell to load flux bash completions add to your profile
. <(flux completion bash)

# Check your kubernetes cluster
# The version of kubernetes cluster must be v1.20 or above
flux check --pre


## install fluxcd
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>


flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./ \
  --personal

# Clone the git repository
git clone https://github.com/$GITHUB_USER/fleet-infra
```

## 3. 配置应用程序部署

创建应用程序的 Kubernetes 清单文件，Flux CD 将监控这些文件的变化并自动部署到集群中。

### 3.1 创建应用程序配置

在 `fleet-infra` 仓库中创建应用程序目录结构：

```bash
cd fleet-infra
mkdir -p apps/production
```

### 3.2 创建应用程序部署文件

创建 `apps/production/my-app.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: default
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

## 4. 配置 GitHub Actions

在应用程序源码仓库中创建 `.github/workflows/ci-cd.yml`：

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v3
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
    
    - name: Update deployment manifest
      if: github.ref == 'refs/heads/main'
      run: |
        # 克隆 fleet-infra 仓库
        git clone https://x-access-token:${{ secrets.FLEET_REPO_TOKEN }}@github.com/${{ github.repository_owner }}/fleet-infra.git
        cd fleet-infra
        
        # 更新镜像标签
        sed -i "s|image: .*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}|g" apps/production/my-app.yaml
        
        # 提交更改
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
        git add .
        git commit -m "Update image to ${{ github.sha }}"
        git push
```

## 5. 验证部署

### 5.1 检查 Flux CD 状态

```bash
# 检查 Flux CD 组件状态
flux get all

# 查看同步状态
flux get sources git
flux get kustomizations
```

### 5.2 监控应用程序部署

```bash
# 查看应用程序 Pod 状态
kubectl get pods -l app=my-app

# 查看部署历史
kubectl rollout history deployment/my-app

# 查看 Flux CD 日志
kubectl logs -n flux-system deployment/source-controller
kubectl logs -n flux-system deployment/kustomize-controller
```

## 6. 故障排除

### 6.1 常见问题

1. **权限问题**：确保 GitHub Token 具有足够的权限
2. **网络问题**：检查集群是否能访问 GitHub
3. **镜像拉取失败**：验证镜像仓库权限和网络连接

### 6.2 调试命令

```bash
# 查看 Flux CD 事件
kubectl get events -n flux-system

# 强制同步
flux reconcile source git flux-system
flux reconcile kustomization flux-system

# 暂停/恢复同步
flux suspend kustomization flux-system
flux resume kustomization flux-system
```

## 总结

通过 GitHub Actions 和 Flux CD 的结合，我们实现了一个完整的 GitOps 工作流：

1. **代码提交** → GitHub Actions 构建镜像
2. **镜像推送** → 更新部署清单
3. **清单变更** → Flux CD 自动同步到集群
4. **持续监控** → 确保集群状态与 Git 仓库一致

这种方式提供了：
- 🔄 **自动化部署**：减少手动操作
- 📝 **版本控制**：所有变更都有记录
- 🔒 **安全性**：通过 Git 权限控制部署
- 🎯 **一致性**：确保环境配置的一致性

下一步可以考虑添加更多功能，如多环境部署、回滚策略、监控告警等。
