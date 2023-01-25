---
layout: post
---

# Github actions + flux CD 构建CICD自动化流水线
![](images/k8s.png)
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
