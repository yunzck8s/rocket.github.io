# 生产级 K8s 存储方案:Rook-Ceph 完整部署实战指南

## 前言

在 Kubernetes 集群中,存储一直是一个关键问题。传统的存储方案往往需要额外的硬件支持和复杂的配置,而 Rook-Ceph 作为云原生存储编排器,能够将 Ceph 分布式存储系统无缝集成到 Kubernetes 中,为容器化应用提供块存储、对象存储和文件系统存储。

本文将详细介绍如何在 Kubernetes 集群中部署 Rook-Ceph,帮助你构建生产级的分布式存储方案。

## 一、环境准备

在开始部署 Rook-Ceph 之前,我们需要确保环境满足以下要求:

### 1.1 Kubernetes 版本要求

Rook 支持的 Kubernetes 版本范围:**v1.29 至 v1.34**

检查当前集群版本:

```bash
kubectl version --short
```

### 1.2 CPU 架构支持

Rook-Ceph 支持以下 CPU 架构:
- **amd64 / x86_64**
- **arm64**

### 1.3 存储设备要求

配置 Ceph 存储集群,至少需要以下存储类型之一:

✅ **裸设备**(无分区或格式化的文件系统)
✅ **裸分区**(无格式化的文件系统)
✅ **LVM 逻辑卷**(无格式化的文件系统)
✅ **块模式的持久卷**(PV)

**检查设备是否可用:**

```bash
lsblk -f
```

输出示例:

```
NAME                  FSTYPE      LABEL UUID                                   MOUNTPOINT
vda
└─vda1                LVM2_member       >eSO50t-GkUV-YKTH-WsGq-hNJY-eKNf-3i07IB
  ├─ubuntu--vg-root   ext4              c2366f76-6e21-4f10-a8f3-6776212e2fe4   /
  └─ubuntu--vg-swap_1 swap              9492a3dc-ad75-47cd-9596-678e8cf17ff9   [SWAP]
vdb
```

**判断标准:**
- 如果 `FSTYPE` 字段为空,该设备可用于 Rook
- 如果 `FSTYPE` 字段不为空,说明设备上已有文件系统,不可用
- 上述示例中,`vdb` 可用于 Rook,而 `vda` 及其分区已有文件系统,不可用

### 1.4 LVM 软件包

在以下场景中,Ceph OSD 需要依赖 LVM:

- ✅ 启用加密(cluster CR 中 `encryptedDevice: "true"`)
- ✅ 指定了元数据设备
- ✅ `osdsPerDevice` 大于 1

**不需要 LVM 的场景:**

- ❌ OSD 在裸设备或分区上创建
- ❌ OSD 使用 `storageClassDeviceSets` 在 PVC 上创建

**安装 LVM:**

CentOS/RHEL:

```bash
sudo yum install -y lvm2
```

Ubuntu/Debian:

```bash
sudo apt-get install -y lvm2
```

**重要提示:** 
即使 Rook 能成功创建 Ceph OSD,如果节点重启但未安装 LVM,OSD Pod 将无法启动。因此请务必在所有存储节点上安装 LVM。

### 1.5 内核模块要求

#### 1.5.1 RBD 模块

Ceph 需要内核支持 RBD 模块。测试方法:

```bash
modprobe rbd
```

如果提示 "module not found",需要:
- 重新编译内核以包含 RBD 模块
- 安装更新的内核版本
- 或选择其他 Linux 发行版

**注意:** GKE 的 Container-Optimised OS (COS) 不包含 RBD 模块。

**内核版本优化:**

Rook 默认的 RBD 配置仅指定 `layering` 特性,以兼容旧内核。如果你的节点运行 **Linux 5.4 或更高版本**,可以在 StorageClass 中启用额外特性:

```yaml
imageFeatures: layering,fast-diff,object-map,deep-flatten,exclusive-lock
```

推荐启用 `fast-diff` 和 `object-map` 特性以提升性能。

#### 1.5.2 CephFS 要求

如果需要从 CephFS 共享文件系统创建 RWX(ReadWriteMany)卷:

- **推荐最低内核版本:** 4.17
- **低于 4.17 的影响:** PVC 大小限制将不会被强制执行,存储配额仅在新内核上生效

检查内核版本:

```bash
uname -r
```

### 1.6 特定发行版配置

#### NixOS 特殊配置

NixOS 的内核模块位于非标准路径:
- `/run/current-system/kernel-modules/lib/modules/`
- 符号链接在 `/nix`

Rook 容器需要读取这些位置才能加载模块,需要将它们作为卷挂载到 CephFS 和 RBD 插件 Pod 中。

**Helm 安装时:** 在 `values.yaml` 中取消注释:
- `csi.csiCephFSPluginVolume`
- `csi.csiCephFSPluginVolumeMount`
- `csi.csiRBDPluginVolume`
- `csi.csiRBDPluginVolumeMount`

**非 Helm 安装时:** 在 `operator.yaml` 的 `rook-ceph-operator-config` ConfigMap 中添加:
- `CSI_CEPHFS_PLUGIN_VOLUME`
- `CSI_CEPHFS_PLUGIN_VOLUME_MOUNT`
- `CSI_RBD_PLUGIN_VOLUME`
- `CSI_RBD_PLUGIN_VOLUME_MOUNT`

**containerd 配置:**

如果使用 containerd,从服务配置中移除 `LimitNOFILE` 以避免 Ceph 命令缓慢或 Mon 脱离仲裁:

```nix
systemd.services.containerd.serviceConfig = {
  LimitNOFILE = lib.mkForce null;
};
```

### 1.7 环境检查清单

在继续部署之前,请确认:

- [ ] Kubernetes 版本在 v1.29-v1.34 范围内
- [ ] CPU 架构为 amd64 或 arm64
- [ ] 至少有一个可用的裸设备/分区/LVM卷
- [ ] 所有存储节点已安装 LVM(如需要)
- [ ] 内核支持 RBD 模块
- [ ] 内核版本 ≥ 4.17(如使用 CephFS)
- [ ] 特殊发行版已完成相应配置

## 二、部署 Rook Operator

### 2.1 添加 Helm 仓库

使用 Helm 部署 Rook Operator 是最简单的方式:

```bash
helm repo add rook-release https://charts.rook.io/release
helm repo update
```

### 2.2 安装 Rook Operator

```bash
helm install --create-namespace --namespace rook-ceph rook-ceph rook-release/rook-ceph
```

### 2.3 验证部署

检查 Operator Pod 状态:

```bash
kubectl get pod -n rook-ceph
```

期望输出:

```bash
NAME                                          READY   STATUS    RESTARTS   AGE
ceph-csi-controller-manager-78d4fd465-5nxxp   1/1     Running   0          54s
rook-ceph-operator-5d58767c48-922xp           1/1     Running   0          54s
```

当两个 Pod 都处于 `Running` 状态时,说明 Rook Operator 部署成功。
## 三、部署 Ceph 集群

### 3.1 安装 Ceph 集群

使用 Helm 安装 Ceph 集群:

```bash
helm install --create-namespace --namespace rook-ceph rook-ceph-cluster \
   --set operatorNamespace=rook-ceph rook-release/rook-ceph-cluster
```

### 3.2 验证集群部署

当看到以下内容,代表部署成功:
```
[root@master1 ~]# kubectl get pod -n rook-ceph 
NAME                                                       READY   STATUS      RESTARTS   AGE
ceph-csi-controller-manager-78d4fd465-w26xc                1/1     Running     0          5m4s
rook-ceph-crashcollector-master1-6d6d5f6b8d-rtjb9          1/1     Running     0          4m3s
rook-ceph-crashcollector-master2-c59cdf9d7-ghblr           1/1     Running     0          4m41s
rook-ceph-crashcollector-master3-7fb65dd88b-cb8np          1/1     Running     0          4m45s
rook-ceph-crashcollector-worker1-5dbfcdbd4b-nvl8h          1/1     Running     0          9m33s
rook-ceph-crashcollector-worker2-8cd877fc-jlzth            1/1     Running     0          3m42s
rook-ceph-crashcollector-worker3-6c9fc7bb66-2stvk          0/1     Pending     0          4m9s
rook-ceph-crashcollector-worker3-86579cc58d-lr87m          0/1     Pending     0          4m9s
rook-ceph-exporter-master1-5757c4c44c-9nxz6                1/1     Running     0          4m3s
rook-ceph-exporter-master2-6575779956-4tmdn                1/1     Running     0          4m41s
rook-ceph-exporter-master3-5897b65b8f-s2mvg                1/1     Running     0          4m45s
rook-ceph-exporter-worker1-5f466c5b57-n895h                1/1     Running     0          9m31s
rook-ceph-exporter-worker2-78655f7674-mchq8                1/1     Running     0          16s
rook-ceph-exporter-worker3-b95d85d75-f4lvk                 0/1     Pending     0          4m7s
rook-ceph-mds-ceph-filesystem-a-645c85c95c-l5v9b           2/2     Running     0          4m45s
rook-ceph-mds-ceph-filesystem-b-7cf7bbb7dc-7fhzf           2/2     Running     0          4m41s
rook-ceph-mgr-a-b6c8fcd7b-nmbzk                            3/3     Running     0          9m33s
rook-ceph-mgr-b-56bf9b455-sg9hm                            3/3     Running     0          10m
rook-ceph-mon-a-68f987b8c4-cbxms                           2/2     Running     0          10m
rook-ceph-mon-b-66f855dd8b-2ktj9                           2/2     Running     0          10m
rook-ceph-mon-c-5876cd7cd-l489f                            2/2     Running     0          10m
rook-ceph-operator-5d58767c48-llzgg                        1/1     Running     0          9m32s
rook-ceph-osd-0-76dfcc8fc7-4qqzq                           2/2     Running     0          39s
rook-ceph-osd-1-5cc44bc857-zz8dw                           1/2     Running     0          16s
rook-ceph-osd-2-5cd47f7bbd-gx69k                           2/2     Running     0          4m9s
rook-ceph-osd-prepare-master1-4bsxw                        0/1     Completed   0          44s
rook-ceph-osd-prepare-master2-jsxcq                        0/1     Completed   0          41s
rook-ceph-osd-prepare-master3-jdg9c                        0/1     Completed   0          56s
rook-ceph-osd-prepare-worker1-clk8b                        0/1     Completed   0          53s
rook-ceph-osd-prepare-worker2-4kqms                        0/1     Completed   0          50s
rook-ceph-osd-prepare-worker3-tp6zq                        0/1     Completed   0          47s
rook-ceph-rgw-ceph-objectstore-a-76589c456b-lfwtv          2/2     Running     0          4m3s
rook-ceph.cephfs.csi.ceph.com-ctrlplugin-5456dbbd6-5bmm6   5/5     Running     0          12m
rook-ceph.cephfs.csi.ceph.com-ctrlplugin-5456dbbd6-p6jrt   5/5     Running     0          12m
rook-ceph.cephfs.csi.ceph.com-nodeplugin-2gmwg             2/2     Running     0          7m11s
rook-ceph.cephfs.csi.ceph.com-nodeplugin-2x2fh             0/2     Pending     0          4m8s
rook-ceph.cephfs.csi.ceph.com-nodeplugin-7xpjn             2/2     Running     0          12m
rook-ceph.cephfs.csi.ceph.com-nodeplugin-gc8rv             2/2     Running     0          7m6s
rook-ceph.cephfs.csi.ceph.com-nodeplugin-qhfpc             2/2     Running     0          7m
rook-ceph.cephfs.csi.ceph.com-nodeplugin-rwv5d             2/2     Running     0          12m
rook-ceph.rbd.csi.ceph.com-ctrlplugin-58d6cb98cb-5qxhr     5/5     Running     0          12m
rook-ceph.rbd.csi.ceph.com-ctrlplugin-58d6cb98cb-gtq7z     5/5     Running     0          12m
rook-ceph.rbd.csi.ceph.com-nodeplugin-28h8f                2/2     Running     0          12m
rook-ceph.rbd.csi.ceph.com-nodeplugin-6qkk2                2/2     Running     0          50s
rook-ceph.rbd.csi.ceph.com-nodeplugin-8bm7t                2/2     Running     0          7m6s
rook-ceph.rbd.csi.ceph.com-nodeplugin-96jvd                2/2     Running     0          7m11s
rook-ceph.rbd.csi.ceph.com-nodeplugin-blh4v                2/2     Running     0          7m
rook-ceph.rbd.csi.ceph.com-nodeplugin-r2qsc                2/2     Running     0          47s

```
* 主要确认 rook-ceph-osd-0 这类容器启动成功就代表部署完成

## 四、验证存储集群

### 4.1 检查 Ceph 集群状态

查看 Ceph 集群健康状态:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

健康的集群应该显示 `HEALTH_OK`。

### 4.2 查看存储类

```bash
kubectl get storageclass
```

应该能看到 Rook-Ceph 提供的 StorageClass,例如:
- `ceph-block` - 块存储(RWO)
- `ceph-filesystem` - 文件系统存储(RWX)
- `ceph-bucket` - 对象存储

### 4.3 创建测试 PVC

创建一个测试 PVC 验证存储功能:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-block
  resources:
    requests:
      storage: 1Gi
```

应用并检查:

```bash
kubectl apply -f test-pvc.yaml
kubectl get pvc test-pvc
```

如果状态为 `Bound`,说明存储集群工作正常。

## 五、结语

通过本文的实战指南,我们成功在 Kubernetes 集群中部署了 Rook-Ceph 分布式存储方案。Rook-Ceph 为 Kubernetes 集群带来了:

✅ **统一的存储解决方案** - 支持块存储(RBD)、对象存储(RGW)和文件系统存储(CephFS)
✅ **自动化运维能力** - 通过 Operator 模式实现自动化部署、扩容和故障恢复
✅ **企业级可靠性** - 基于 Ceph 的分布式架构,提供数据多副本和高可用保障
✅ **云原生集成** - 无缝对接 Kubernetes 存储生态,支持 CSI 标准

### 关键要点回顾

在实际部署过程中,请特别注意:

1. **版本兼容性** - 确保 Kubernetes 版本在 v1.29-v1.34 范围内
2. **存储设备准备** - 使用未格式化的裸设备或分区,避免数据丢失
3. **LVM 依赖** - 根据使用场景判断是否需要安装 lvm2 包
4. **内核支持** - 验证 RBD 模块加载,CephFS 需要内核 ≥ 4.17
5. **OSD 部署验证** - 确认 rook-ceph-osd Pod 成功运行是集群健康的关键指标
6. **生产环境建议** - 至少 3 个存储节点,每个节点配置独立的 OSD 磁盘

### 后续优化建议

完成基础部署后,你可以考虑:

- **性能调优** - 调整 OSD 数量、副本数和 PG 数量
- **监控集成** - 启用 Ceph Dashboard 和 Prometheus 监控
- **备份策略** - 实施定期快照和异地备份
- **高可用配置** - 设置 Mon 和 MGR 的多副本部署
- **容量管理** - 配置存储配额和用量告警
- **安全加固** - 启用存储加密和网络隔离

### 参考资源

- Rook 官方文档: https://rook.io/docs/rook/latest-release/
- Rook GitHub 仓库: https://github.com/rook/rook
- Ceph 官方文档: https://docs.ceph.com/
- Kubernetes CSI 文档: https://kubernetes-csi.github.io/docs/
- Rook Helm Charts: https://rook.io/docs/rook/latest-release/Helm-Charts/helm-charts/

---

如果本文对你有帮助,欢迎关注、点赞和分享!在后续文章中,我将继续分享:

- 📊 **VictoriaMetrics 监控系统部署** - 高性能时序数据库解决方案

有任何问题或建议,欢迎在评论区交流讨论!