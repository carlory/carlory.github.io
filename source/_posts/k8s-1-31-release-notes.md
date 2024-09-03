---
title: Kubernetes 1.31 发布日志
comments: false
date: 2024-08-13  14:33:25
tags:
  - sig-release
  - Kubernetes
categories: Kubernetes 发布日志
keywords: 'kubernetes,发布日志,changelog,released,主題,doc,教程'
description: 
top_img: "."
cover: https://kubernetes.io/images/blog/2024-08-13-kubernetes-1.31-release/k8s-1.31.png
abbrlink: k8s-1-31-release-notes
---

10 项已晋升为稳定版，19 项正在进入 Beta 阶段，以及新增 11 项功能。

## 🏠 发布主题和徽标

<img src="https://kubernetes.io/images/blog/2024-08-13-kubernetes-1.31-release/k8s-1.31.png" alt="Kubernetes v1.31 徽标" style="width: 50%;" />

让我们庆祝本次发布，并感谢社区伙伴在本次的里程碑中所作出的努力和贡献。

## 🎉 亮点功能

### [KEP-4639](https://kep.k8s.io/4639) 新增基于 OCI 镜像的只读卷

> 🎬 适用于 AI 的推理场景

需要在 kube-apiserver 和 kubelet 上启用特性门控 ImageVolume 才能正常运行, 并且容器运行时支持该功能 (如 CRI-O ≥ v1.31), 则可以创建如下所示的示例 pod.yaml：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  containers:
    - name: test
      image: registry.k8s.io/e2e-test-images/echoserver:2.3
      volumeMounts:
        - name: volume
          mountPath: /volume
  volumes:
    - name: volume
      image:
        reference: quay.io/crio/artifact:v1
        pullPolicy: IfNotPresent
```

数据科学家、MLOps 工程师或 AI 开发人员可以将**大型语言模型权重或机器学习模型权重与模型服务器**一起安装在 Pod 中，以便他们可以有效地提供它们，而无需将它们包含在模型服务器容器映像中。他们可以将这些内容打包到 OCI 对象中，以利用 OCI 分发并确保高效的模型部署。这使他们能够将模型规格/内容与处理它们的可执行文件分开。

不同之处:

- 如果提供了 `:latest` 标签作为引用, 则 `pullPolicy` 将默认为 `Always`, 而在任何其他情况下，如果未设置，它将默认为 `IfNotPresent`
- 卷作为只读 （ro） 和非可执行文件 （noexec） 挂载
- 不支持容器的子路径挂载 （`spec.containers[*].volumeMounts.subpath`）
- 字段 `spec.securityContext.fsGroupChangePolicy` 对此卷类型没有影响
- 若启用了特性门控 ImageVolume 和  AlwaysPullImages 准入插件, 那么 image 卷的拉取策略将受 AlwaysPullImages 准入插件的影响

更多关于只读卷的信息, 请参考 [Kubernetes 文档](https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/#image)

### [KEP-4622](https://kep.k8s.io/4622) 向 kubelet 添加一个配置选项

> 💡 突破了 Kubelet 旧约束, 但若修改默认配置, 需要集群管理员持续关注, 是否造成严重影响

`--max-allowable-numa-nodes` 是一个新的 kubelet 配置选项，用于配置 TopologyManager 中的 `maxAllowableNUMANodes` 的值。默认情况下，`maxAllowableNUMANodes` 的值为 8。这个值是在 4 年前作为权宜之计添加的，用于缓解在尝试枚举可能的 NUMA 关联并生成其提示时发生的状态爆炸。

通过使此设置可配置，我们使用户能够在适当时增加此限制。管理员现在可以允许具有 8 个以上 NUMA 节点的节点在启用 TopologyManager 的情况下运行。当然, 它会减慢节点上的 Pod 准入/启动时间, 这是因为 kubelet 的 TopoolgyManager 在决定 cpu 和设备可以以对齐方式分配到哪里时需要考虑更多组合。该功能在节点级别, 减速仅影响配置了该功能的节点，不会对集群产生任何影响。


## 📉 破坏性变更

### KEP-3063/KEP-4381 DRA 和结构化参数

> 🍵 莫慌, 还在 Alpha 阶段, 默认不启用, 仅影响手动启用该功能的集群和用户

DRA 相关的 API 定义在 v1alpha3 对 API 进行了彻底的改造。因此不再支持从 v1alpha2 过渡到 v1alpha3，所以 1.29 和 1.30 中该 API 的往返测试数据将被删除。

一些主要变化如下：

- 删除 CRD 支持。供应商配置参数仍支持作为树内类型的原始扩展字段。
- 移除了对即时分配的支持。它对于结构化参数毫无意义, 对于经典 DRA 的论证也很薄弱。
- 移除 `claim.status.sharable`字段, 当多个 Pod 引用它们时，可以共享所有声明。但共享程度不得超过“reserverFor”数组的大小限制。
- 删除了 “设备模型” 的概念以及仅适用于此类模型的选择机制。
- `ResourceClaim` 不再是一个关联一个 `ResourceClaimClass` (已移除)，而是 `ResourceClaim` 内部的各个设备请求现在引用一个 `DeviceClass`。
- 支持 “匹配属性”。
- 支持 “管理员访问权限”。
- 支持 “分区设备”。
- 每个请求支持多个设备（“全部” 和固定数量）。
- 支持容器从 `Claim` 的某个请求中选择设备，而不是选择 `Claim` 公开的所有设备。
- 支持具有结构化参数的网络连接设备。
- 重命名 ResourceSlice 为 ResourcePool, 引入了池的概念。

社区提供了一个[示例 DRA 驱动程序](https://github.com/kubernetes-sigs/dra-example-driver)，它旨在演示如何构造 DRA 资源驱动程序并将其包装在 helm 图表中的最佳实践。开发人员可以对其进行分叉和修改，以开始编写自己的驱动程序。

关于 DRA 的更多信息，请参考 [Kubernetes 文档](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)

## 🗑️ 不顶用了

### 弃用 (Deprecation)

- [KEP-4569](https://kep.k8s.io/4569) 对 cgroup v1 的支持移至维护模式, 弃用路径与当初的 Dockershim 基本相似
- [KEP-4004](https://kep.k8s.io/4004) 弃用 v1.Node 的 `status.nodeInfo.kubeProxyVersion` 字段
- [Issue #125689](https://github.com/kubernetes/kubernetes/issues/125689) GODEBUG 和对 SHA-1 证书的支持将 在 2025 年上半年发布的 go 1.24 版本中完全消失, 如果你依赖 SHA-1 证书，请开始放弃使用它们
- kube-scheduler 中非 CSI 卷限制插件的弃用 (AzureDiskLimits、CinderLimits、EBSLimits 及 GCEPDLimits), 如果已在调度器配置中显式使用已弃用的插件, 请用 NodeVolumeLimits 插件替换它们

### 移除 (Removal)

- 删除 CephFS、Ceph RBD 卷插件
- 删除 kubelet `--keep-terminated-pod-volumes` 命令行标志
- 删除云驱动集成的树内支持的最后剩余部分。 这并不意味着你无法与某云驱动集成，只是你现在必须使用推荐的外部集成方法。一些集成组件是 Kubernetes 项目的一部分，其余集成组件则是第三方软件

## 🏥 升级须知

### [KEP-2589]() Portworx 卷/持久卷支持 CSI 迁移

> 😴 若你的集群没有使用 Portworx 类型的数据卷或持久卷, 可跳过本节内容

如果您的集群中使用了 Portworx 树内卷, 那么在升级到 Kubernetes 1.31 版本前, 您需要注意以下事项：

- 可以通过将 `CSIMigrationPortworx` 功能门控设置为 `false` 来禁用 Portworx 卷的 CSI 迁移。
- 在测试集群上安装 [Portworx CSI 驱动程序](https://docs.portworx.com/portworx-enterprise/operations/operate-kubernetes/storage-operations/csi), 并测试已有的 Portworx 卷是否可以正常工作。Kubernetes e2e 测试不包含此部分, 但通过了本地测试和 Portworx 的支持团队确认。
  
为了确保您的集群在升级后不会受到影响, 请在升级之前自行测试您的集群。



## 🐶 稳如老狗 (GA)

> GA 全称 General Availability，即正式发布。Kubernetes 的进阶路线通常是 Alpha、Beta、Stable (即 GA)、Deprecation/Removal, 这四个阶段。

- [KEP-3762](https://kep.k8s.io/3762) 持久卷的最新阶段转换时间功能, 记录 PersistentVolume 最后一次转换到不同阶段的时间戳
- [KEP-2305](https://kep.k8s.io/2305) 监控指标动态基数强制限制功能, 限制标签允许值的数量来缓解基数爆炸, 和缓解内存问题
- [KEP-3836](https://kep.k8s.io/3836) 提高 Kube-proxy 服务的入口连接的可靠性
- [KEP-4009](https://kep.k8s.io/4009) 设备插件的 API 添加对 CDI 设备的支持, 允许设备插件提供方使用 CDI 定义容器化环境所需的修改
- [KEP-0024](https://kep.k8s.io/24) AppArmor 支持
- [KEP-3017](https://kep.k8s.io/3017) PDB 添加对不健康的 Pod 驱逐策略的支持, 通过用户提供的配置信息, 来防止被保护的应用程序受到不可用和服务中断的影响
- [KEP-3329](https://kep.k8s.io/3329) 扩展了 Job 的字段，以配置用于处理 Pod 故障的作业策略。它允许用户确定由基础结构错误引起的某些 pod 故障，并在不增加对 backoffLimit 的计数器的情况下重试它们。此外，它还允许用户确定由软件错误引起的某些 pod 故障，并提前终止关联的作业
- [KEP-3715](https://kep.k8s.io/3715) 允许改变 Indexed 类 Job 的 spec.completions, 前提是更新后的值等于 spec.parallelism
- [KEP-3335](https://kep.k8s.io/3335) 控制 StatefulSet 启动副本序号, 允许在不中断底层应用程序的情况下迁移 StatefulSet
- [KEP-2185](https://kep.k8s.io/2185) ReplicaSet 控制器的缩减 Pod 受害者选择算法的随机方法，以减轻故障域之间的 ReplicaSet 不平衡

## 🔥 推广试用 (Beta)

> Beta 阶段的功能是指那些已经经过 Alpha 阶段的功能, 且在 Beta 阶段中添加了更多的测试和验证, 通常情况下是默认启用的。

- [KEP-2589](https://kep.k8s.io/2589) Portworx 卷/持久卷支持 CSI 迁移
- [KEP-2644](https://kep.k8s.io/2644) 始终遵守 PersistentVolume 回收策略, 防止 PersistentVolume 删除后, 存储系统中的数据卷依然存在的情况
- [KEP-3633](https://kep.k8s.io/3633) 将 MatchLabelKeys 和 MismatchLabelKeys 引入 PodAffinity 和 PodAntiAffinity, 能够在现有 LabelSelector 之上精细控制 Pod 预期共存 (PodAffinity) 或不共存 (PodAntiAffinity) 的范围
- [KEP-3866](https://kep.k8s.io/3866) Kube-proxy 基于 nftables（Linux Kernel 社区对 iptables 的替代）实现了新的后端，旨在解决 iptables 现存的性能问题
- [KEP-4358](https://kep.k8s.io/4358) 自定义资源字段选择器, 允许为自定义资源类型设置预定义的字段选择配置, 客户端可以使用前面预设的字段选择器来过滤资源
- [KEP-4420](https://kep.k8s.io/4420) 当生成的名称与现有资源名称冲突时，Kube-apiserver 会自动重试使用 generateName 的创建请求，最多重试 7 次
- [KEP-4444](https://kep.k8s.io/4444) 服务流量分配, 在 Service 中实现了一个新字段，作为底层实现在制定路由决策时要考虑的偏好或提示
- [KEP-4006](https://kep.k8s.io/4006) 将 Kubernetes 客户端的双向流协议从 SPDY/3.1 转换为 WebSockets
- [KEP-4193](https://kep.k8s.io/4193) 改进绑定的服务帐户令牌, 自动将 Pod 关联的节点的 name 和 uid （通过 spec.nodeName ）嵌入到生成的令牌中, 并允许用户获取专门与 Node 对象生命周期相关的令牌
- [KEP-4033](https://kep.k8s.io/4033) 增加了容器运行时指示 kubelet 使用哪个 cgroup 驱动程序的能力, 无需在 kubelet 配置中指定 cgroup 驱动程序, 并消除 kubelet 和运行时之间 cgroup 驱动程序配置不一致的可能性
- [KEP-4622](https://kep.k8s.io/4622) 向 kubelet 添加一个配置选项，允许在 TopologyManager 中配置 maxAllowableNUMANodes 的值
- [KEP-1880](https://kep.k8s.io/1880) 管理 Kubernetes Service 的 IP 范围 (因依赖 Beta API, 默认关闭)
- [KEP-4568](https://kep.k8s.io/4568) 启用弹性监视缓存初始化以避免控制平面过载
- [KEP-4292](https://kep.k8s.io/4292) 在 kubectl debug 命令中的预定义配置文件之上添加了新的自定义配置文件功能, 使kubectl debug pod/node 或临时容器规范可配置
- [KEP-3751](https://kep.k8s.io/3751) 允许用户在配置/绑定持久卷后动态修改卷属性, 例如 IOP 和吞吐量, 可变卷属性名及值规范来自存储系统, 不同存储系统间可能不通用 (因依赖 Beta API, 默认关闭)
- [KEP-4265](https://kep.k8s.io/4265) 允许用户选择退出 Linux 容器的 CRI 屏蔽 /proc (因依赖默认关闭的 Beta 功能 UserNamespacesSupport, 导致默认关闭)
- [KEP-3857](https://kep.k8s.io/3857) 递归的只读挂载
- [KEP-3998](https://kep.k8s.io/3998) Job 添加自定义的 Job 成功策略的支持, 一旦 Job 满足成功策略, 其余的 Pod 将被终止
- [KEP-1029](https://kep.k8s.io/1029) 使用文件系统配额监控本地临时存储的利用率

## 🚀 手动尝鲜 (Alpha)

- [KEP-2340](https://kep.k8s.io/2340) 监视缓存提供一致的数据视图, 提高 Kubernetes 对于 Get 和 List 请求的可扩展性和性能. 尝试升至 Beta 后, 因性能问题被回退至 Alpha 阶段
- [KEP-4601](https://kep.k8s.io/4601) 授权属性将扩展为包括列表、监视和删除集合中的字段选择器和标签选择器。这将允许授权者在做出授权决定时使用这些选择器
- [KEP-4633](https://kep.k8s.io/4633) 仅允许对配置的端点进行匿名身份验证
- [KEP-3063](https://kep.k8s.io/3063) 使用控制平面控制器进行动态资源分配 (DRA)
- [KEP-4176](https://kep.k8s.io/4176) 一种新的静态策略，优先分配同一插槽上不同 CPU 的核心
- [KEP-4381](https://kep.k8s.io/4381) 借助结构化参数，kube-scheduler 和 Cluster Autoscaler 可以自行处理和模拟声明分配，而无需依赖第三方驱动程序
- [KEP-4639](https://kep.k8s.io/4639) 新增基于 OCI 镜像的只读卷
- [KEP-3619](https://kep.k8s.io/3619) 细粒度的补充组控制
- [KEP-4192](https://kep.k8s.io/4192) 移动存储版本迁移器到主库
- [KEP-4680](https://kep.k8s.io/4680) 将资源运行状况状态添加到设备插件和 DRA 的 Pod 状态
- [KEP-4368](https://kep.k8s.io/4368) 支持 Job 的 ManagedBy 字段

## 致谢

1. [Kubernetes 1.31 官方 - 发布团队](https://github.com/kubernetes/sig-release/blob/master/releases/release-1.31)
1. [Kubernetes 1.31 官方 - 增强跟踪](https://bit.ly/k8s131-enhancements)
1. [Kubernetes 1.31 官方 - 主题讨论](https://github.com/kubernetes/sig-release/discussions/2526)
1. [Kubernetes 1.31 官方 - 变更日志](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.31.md)
