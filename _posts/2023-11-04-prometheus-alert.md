---
layout: post
---

```
场景：客户现场经常会出现服务资源不足，导致服务器down掉，需要接入预警通知。
环境：
  kubernetes v1.24.2
  kube-prometheus-stack：v0.26.0
```

# Prometheus的概念
  在开始之前首先要清楚prometheus是什么，