---
title: KCD 2024 SHANGHAI - 01
summary: 两位演讲者分别是来自字节跳动的曹贺与来自 Intel 的任强，他们分别介绍了字节跳动开源的 Katalyst 以及其插件化的资源管理机制、CRI 扩展框架 NRI 以及 Katalyst 基于 NRI 的优化。
date: 2024-05-19

authors:
  - admin

tags:
  - Katalyst-Core
  - Bytedance
  - Kubernetes
---

## 01 - 基于NRI实现精细化且可拔插的容器资源管理 - 曹贺, 任强

### Katalyst 简介
#### Kubernetes资源规划的挑战

* 在线业务的资源利用率呈现潮汐现象，夜间的资源利用率又非常低
* 用户倾向过度申请资源以保证服务的稳定性，造成资源浪费

{{< figure src="resource_provision.png" caption="resource_provision" theme="light" >}}


#### 在离线混部

在线业务和离线作业对资源的使用模式天然是互补的:
* 在线业务重CPU和RPC时延
* 离线作业重内存和吞吐

{{< figure src="resource_provision_2.png" caption="resource_provision_2" theme="light" >}}


#### 资源管理系统 Katalyst
旨在为运行Kubernetes之上的工作负载提供更加强劲的资源管理能力

{{< figure src="katalyst.png" caption="katalyst" theme="light" >}}


### Katalyst 插件化的资源管理机制

#### 精细化的资源管理策略

{{< figure src="qos_class.png" caption="qos_class" theme="light" >}}

* 4种扩展的QoS级别：
  * 表达了服务对资源质量的要求
  * 以CPU为主导的维度来命名
* 更多QoS Enhancement
  * NUMA 绑定
  * NUMA 独占
  * 流量分级
  * ...

#### QRM框架: 使kubelet的资源管理策略可扩展

{{< figure src="qrm.png" caption="" theme="light" >}}

* QoS Resource Manager
  * 提供插件化的资源管理能力
  * 与Topology Manager集成以实现NUMA亲和
  * 在容器运行的过程中，动态调整其资源分配
* QoS Resource Plugin
  * 定制对容器的资源分配策略

{{% callout note %}}
参考了原生Device Manager的特性， Katalyst 侵入式地修改了Kubelet，实现了 QoS Resource Manger（QRM） 对 Kubelet 中自带的 CPU Manager 与 Memory Manager 进行了替换，
以实现更细粒度的动态资源管理，并通过与 Topology Manager 集成实现了 NUMA 亲和的资源管理。
{{% /callout %}}

#### ORM框架: 将QRM框架从kubelet中解耦

{{< figure src="orm.png" caption="" theme="light" >}}

ORM: [Out-of-Band Resource Manager](https://github.com/kubewharf/katalyst-core/issues/430)

* 两种模式
  * NRI 模式
  * 旁路异步更新模式(Bypass 模式)
* 支持NUMA亲和
  * 带外的Topology Manager
  * 带外的PodResourcesServer：
    - 由于原生的PodResourceAPI不可用，Reporter Manger模块从带外的PodResourcesServer获取真实的资源信息，上传到KCNR供调度使用。

{{% callout note %}}
1) Katalyst Agent 以NRI插件的方式在节点上作为独立的进程运行，在其中实现了 Out-of-Band Resource Manager，复用了先前 QRM 框架中的部分插件。
2) Containerd 调用 NRI 插件，并通过 NRI 与容器运行时通信，实现动态的容器资源管理的策略，与先前的实现机制相比具有更好的可扩展性与可插拔性。
3) 对于不支持NRI的容器运行时，Katalyst Agent 提供了旁路 异步更新模式，对于containerd低版本，在启动容器后异步注入cgroup配置，实现资源管理策略。
{{% /callout %}}

### NRI机制简介

#### Kubernetes QoS资源管理目前社区的一些实现方式

{{< figure src="flow_chart.png" caption="" theme="light" >}}

#### NRI节点细粒度资源管理利器
* NRI是一个CRI运行时插件扩展管理的通用框架，可以适用于Containerd，CRI-O等容器运行时
* NRI为扩展插件提供了跟踪容器状态，并对其配置进行有限修改的基本机制。(资源配置、环境变量、annotation)
* NRI插件是运行时无关的，可以适用于Containerd，CRI-O等容器运行时

{{< figure src="nri.png" caption="" theme="light" >}}

{{% callout note %}}
- a. 方案一 CRI proxy (Intel)：hook相应的容器/Pod事件，对原生组件有侵入性修改。
- b. 方案二 Node Agent：异步更新，时效性不高。
- c. 方案三 katalyst (Kubelet增强机制)：调用plugin agent，时效性高。但对上游kubelet侵入修改。
{{% /callout %}}

#### NRI插件运行机制

- 以下两种运行方式:
  * NRI插件作为一个独立的进程，通过NRI Socket与Runtime通信，可以部署为DaemonSet
  * NRI插件作为预编译的二进制文件，放置在指定路径下由Containerd发起调用(类似CNI，一次性加载) 。

- Hook事件
  - Pod Event: RunPodSandbox, StopPodSandbox, RemovePodSandbox
  - Container Events: CreateContainer, StartContainer, StopContainer, UpdateContainer, RemoveContainer, PostCreateContainer, PostStartContainer, PostUpdateContainer

- 主动发起
  - stub.UpdateContainer()

### NRI在Katalyst中的应用

#### NRI Enhanced ORM in KataLyst
* Katalyst Agent作为一个NRI插件，从Containerd获取Pod/Container事件
  * RunPodSandbox，CreateContainer，RemovePodSandbox
* 旁路模式与NRI并存，适配低版本的Containerd环境
* 复用QRM插件机制，降低迁移成本
* Reconcile机制通过NRI UpdateContainer() 更新容器资源

{{< figure src="nri_orm.png" caption="" theme="light" >}}


### 社区建设

* Github仓库：
  [kubewharf/katalyst-core](https://github.com/kubewharf/katalyst-core)
  [containerd/nri](https://github.com/containerd/nri)
  [containers/nri-plugins](https://github.com/containers/nri-plugins)

* 相关 KubeCon 演讲：
* [Advancing Memory Management in Kubernetes: Next Steps with Memory QoS - Dixita Narang, Google & Antti Kervinen, Intel, KubeCon NA 2023](https://kccncna2023.sched.com/event/1R2nL)
* [使用可插拔和可定制的智能运行时提升工作负载的QoS | Enhance Workload QoS with Pluggable and Customizable Smarter Runtimes - Rougang Han, Alibaba & Kang Zhang, Intel](https://kccncosschn2023.sched.com/event/1PTH5)
* [NRI: Extending Containerd And CRI-O With Common Plugins - Krisztian Litkey, Intel & Mike Brown, IBM, KubeCon NA 2022](https://kccncna2022.sched.com/event/182JT)
* [Maximizing Workload’s Performance With Smarter Runtimes - Krisztian Litkey & Alexander Kanevskiy, Intel, KubeCon EU 2021](https://kccnceu2021.sched.com/event/iE1Y)