---
layout: post
title: "Kubernetes集群监控告警：Prometheus AlertManager实战"
date: 2023-11-04
tags: [devops, kubernetes, prometheus, alertmanager, monitoring]
permalink: /2023/11/04/prometheus-alertmanager-kubernetes-monitoring.html
---

```
场景：客户现场经常会出现服务资源不足，导致服务器down掉，需要接入预警通知。
环境：
  kubernetes v1.24.2
  kube-prometheus-stack：v0.26.0
```

# Prometheus的概念
  在开始之前首先要清楚prometheus是什么，