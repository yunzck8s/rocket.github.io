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

在开始之前首先要清楚prometheus是什么。Prometheus 是一个开源的系统监控和告警工具包，最初由 SoundCloud 开发。它具有以下特点：

- **多维数据模型**：通过指标名称和键值对标识时间序列
- **灵活的查询语言**：PromQL 允许对收集的数据进行切片和切块
- **不依赖分布式存储**：单个服务器节点是自治的
- **HTTP 拉取模型**：通过 HTTP 协议收集时间序列数据
- **支持推送网关**：用于批处理作业
- **通过服务发现或静态配置发现目标**
- **多种图形和仪表板支持模式**

## 1. 安装 kube-prometheus-stack

使用 Helm 安装 kube-prometheus-stack，这是一个包含 Prometheus、AlertManager、Grafana 等组件的完整监控解决方案。

### 1.1 添加 Helm 仓库

```bash
# 添加 prometheus-community helm 仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 1.2 创建命名空间

```bash
kubectl create namespace monitoring
```

### 1.3 安装 kube-prometheus-stack

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.ruleSelectorNilUsesHelmValues=false
```

## 2. 配置 AlertManager

### 2.1 创建 AlertManager 配置

创建 `alertmanager-config.yaml` 文件：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-kube-prometheus-stack-alertmanager
  namespace: monitoring
stringData:
  alertmanager.yml: |
    global:
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'your-email@gmail.com'
      smtp_auth_username: 'your-email@gmail.com'
      smtp_auth_password: 'your-app-password'
    
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'web.hook'
      routes:
      - match:
          alertname: Watchdog
        receiver: 'null'
      - match:
          severity: critical
        receiver: 'critical-alerts'
      - match:
          severity: warning
        receiver: 'warning-alerts'
    
    receivers:
    - name: 'null'
    
    - name: 'web.hook'
      email_configs:
      - to: 'admin@company.com'
        subject: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Labels: {{ range .Labels.SortedPairs }}{{ .Name }}={{ .Value }} {{ end }}
          {{ end }}
    
    - name: 'critical-alerts'
      email_configs:
      - to: 'critical-alerts@company.com'
        subject: '🚨 [CRITICAL] {{ .GroupLabels.alertname }}'
        body: |
          🚨 Critical Alert Triggered!
          
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Severity: {{ .Labels.severity }}
          Instance: {{ .Labels.instance }}
          Time: {{ .StartsAt.Format "2006-01-02 15:04:05" }}
          {{ end }}
      
      webhook_configs:
      - url: 'http://webhook-service:8080/alerts'
        send_resolved: true
    
    - name: 'warning-alerts'
      email_configs:
      - to: 'warning-alerts@company.com'
        subject: '⚠️ [WARNING] {{ .GroupLabels.alertname }}'
        body: |
          ⚠️ Warning Alert
          
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Instance: {{ .Labels.instance }}
          {{ end }}
```

### 2.2 应用配置

```bash
kubectl apply -f alertmanager-config.yaml
```

## 3. 创建自定义告警规则

### 3.1 创建告警规则文件

创建 `custom-alerts.yaml`：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-alerts
  namespace: monitoring
  labels:
    app: kube-prometheus-stack
    release: kube-prometheus-stack
spec:
  groups:
  - name: kubernetes-resources
    rules:
    
    # Pod CPU 使用率过高
    - alert: PodHighCPUUsage
      expr: |
        (
          sum by (namespace, pod) (
            rate(container_cpu_usage_seconds_total{container!="POD",container!=""}[5m])
          ) * 100
        ) > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} CPU usage is high"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} CPU usage is {{ $value }}% for more than 5 minutes."
    
    # Pod 内存使用率过高
    - alert: PodHighMemoryUsage
      expr: |
        (
          sum by (namespace, pod) (
            container_memory_working_set_bytes{container!="POD",container!=""}
          ) / sum by (namespace, pod) (
            container_spec_memory_limit_bytes{container!="POD",container!=""} > 0
          ) * 100
        ) > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} memory usage is high"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} memory usage is {{ $value }}% for more than 5 minutes."
    
    # Pod 重启频繁
    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 0
      for: 0m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has restarted {{ $value }} times in the last 15 minutes."
    
    # 节点磁盘空间不足
    - alert: NodeDiskSpaceFillingUp
      expr: |
        (
          node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"} * 100 < 10
          and
          predict_linear(node_filesystem_avail_bytes{fstype!="tmpfs"}[6h], 24*60*60) < 0
          and
          node_filesystem_readonly{fstype!="tmpfs"} == 0
        )
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "Node {{ $labels.instance }} disk is filling up"
        description: "Filesystem on {{ $labels.device }} at {{ $labels.instance }} has only {{ printf \"%.2f\" $value }}% available space left and is filling up."
    
    # 节点内存使用率过高
    - alert: NodeHighMemoryUsage
      expr: |
        (
          (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 90
        )
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ $labels.instance }} memory usage is high"
        description: "Node {{ $labels.instance }} memory usage is {{ printf \"%.2f\" $value }}% for more than 5 minutes."
    
    # 服务不可用
    - alert: ServiceDown
      expr: up == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Service {{ $labels.job }} is down"
        description: "Service {{ $labels.job }} on {{ $labels.instance }} has been down for more than 5 minutes."
```

### 3.2 应用告警规则

```bash
kubectl apply -f custom-alerts.yaml
```

## 4. 配置钉钉/企业微信通知

### 4.1 创建 Webhook 服务

创建一个简单的 webhook 服务来处理钉钉通知：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alert-webhook
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alert-webhook
  template:
    metadata:
      labels:
        app: alert-webhook
    spec:
      containers:
      - name: webhook
        image: timonwong/prometheus-webhook-dingtalk:v2.1.0
        ports:
        - containerPort: 8060
        env:
        - name: DINGTALK_WEBHOOK
          value: "https://oapi.dingtalk.com/robot/send?access_token=YOUR_TOKEN"
        args:
        - --web.listen-address=:8060
        - --config.file=/etc/prometheus-webhook-dingtalk/config.yml
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus-webhook-dingtalk
      volumes:
      - name: config
        configMap:
          name: webhook-config
---
apiVersion: v1
kind: Service
metadata:
  name: alert-webhook
  namespace: monitoring
spec:
  selector:
    app: alert-webhook
  ports:
  - port: 8060
    targetPort: 8060
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: webhook-config
  namespace: monitoring
data:
  config.yml: |
    templates:
      - /etc/prometheus-webhook-dingtalk/template.tmpl
    targets:
      webhook:
        url: https://oapi.dingtalk.com/robot/send?access_token=YOUR_TOKEN
        secret: YOUR_SECRET
  template.tmpl: |
    {{ define "dingtalk.default.message" }}
    {{- if gt (len .Alerts.Firing) 0 -}}
    {{- range $index, $alert := .Alerts.Firing }}
    =========== 监控报警 ===========
    告警名称: {{ $alert.Labels.alertname }}
    告警级别: {{ $alert.Labels.severity }}
    告警机器: {{ $alert.Labels.instance }}
    告警详情: {{ $alert.Annotations.summary }}
    告警时间: {{ $alert.StartsAt.Format "2006-01-02 15:04:05" }}
    =============================
    {{- end }}
    {{- end }}
    
    {{- if gt (len .Alerts.Resolved) 0 -}}
    {{- range $index, $alert := .Alerts.Resolved }}
    =========== 告警恢复 ===========
    告警名称: {{ $alert.Labels.alertname }}
    告警机器: {{ $alert.Labels.instance }}
    告警详情: {{ $alert.Annotations.summary }}
    恢复时间: {{ $alert.EndsAt.Format "2006-01-02 15:04:05" }}
    =============================
    {{- end }}
    {{- end }}
    {{ end }}
```

## 5. 验证和测试

### 5.1 检查组件状态

```bash
# 检查 Prometheus 状态
kubectl get pods -n monitoring | grep prometheus

# 检查 AlertManager 状态
kubectl get pods -n monitoring | grep alertmanager

# 查看告警规则
kubectl get prometheusrules -n monitoring
```

### 5.2 访问 Web 界面

```bash
# 端口转发访问 Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# 端口转发访问 AlertManager
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093

# 端口转发访问 Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

### 5.3 测试告警

创建一个高 CPU 使用率的测试 Pod：

```bash
kubectl run cpu-stress --image=progrium/stress -- stress --cpu 2 --timeout 300s
```

## 6. 故障排除

### 6.1 常见问题

1. **告警规则不生效**：检查 PrometheusRule 的 labels 是否正确
2. **邮件发送失败**：验证 SMTP 配置和网络连接
3. **Webhook 调用失败**：检查 webhook 服务状态和网络策略

### 6.2 调试命令

```bash
# 查看 Prometheus 配置
kubectl get prometheus -n monitoring -o yaml

# 查看 AlertManager 日志
kubectl logs -n monitoring alertmanager-kube-prometheus-stack-alertmanager-0

# 查看告警状态
curl http://localhost:9093/api/v1/alerts

# 测试告警规则
curl -X POST http://localhost:9093/api/v1/alerts
```

## 总结

通过本文的配置，我们实现了一个完整的 Kubernetes 集群监控告警系统：

1. **📊 监控覆盖**：CPU、内存、磁盘、网络等关键指标
2. **🚨 多级告警**：warning 和 critical 不同级别的告警
3. **📧 多渠道通知**：邮件、钉钉、企业微信等
4. **🔧 自定义规则**：根据业务需求定制告警规则
5. **📈 可视化展示**：通过 Grafana 展示监控数据

这套监控系统能够帮助运维团队：
- ⚡ **快速响应**：及时发现和处理问题
- 🎯 **精准定位**：详细的告警信息帮助快速定位问题
- 📊 **趋势分析**：通过历史数据分析系统性能趋势
- 🛡️ **预防故障**：提前预警避免系统宕机

建议根据实际业务场景调整告警阈值和通知策略，确保告警的有效性和及时性。