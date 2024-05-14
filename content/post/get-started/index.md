---
title: KCD 2024 SHANGHAI - 05 
summary: 下文是我在线观看这场行业大会议题的简单介绍与梳理，主要包括Kubernetes调度与资源管理以及LLM相关的议题。
date: 2024-04-25

authors:
  - admin

tags:
  - Godel Scheduler
  - Bytedance
  - Kubernetes
---

## 05 - Godel Scheduler：字节开源内部超大规模在离线统一调度器 - 任玉泉，字节跳动
演讲者为来自字节跳动的任玉泉，他向我们介绍了字节跳动内部基于 Kubernetes 的大规模在离线统一调度系统Godel Scheduler的研发背景、
详细设计、内部实践以及后续开源的一系列规划。

## Part 01: 研发背景
业务需要：解决离线两套系统混部导致的资源成本和效率问题。
- 成本：
    - 资源成本：低利用率、高适配成本。
    - 开发成本：重复工作多、适配成本高。
- 效率：
  - 资源流转效率低
  - 功能迭代效率低
- 运维：
  - 压力大
- 开源产品不满足需求

## Part 02: 详细设计
### Godel Scheduler整体架构：利用乐观并发、两层框架、统一调度。

{{< figure src="godel_scheduler.png" caption="godel_scheduler整体架构" theme="light" >}}

主要特点：
* 乐观并发：把最耗时的操作抽出来，并发执行，提升吞吐天花板。
* 两层框架：增强扩展性（“批”调度能力）的同时，进一步提升性能。
* 统一调度：在线任务，离线任务，一个调度器。
* 基于K8S系统：完全兼容K8S生态。

{{% callout note %}}
1. k8s scheduler调度器调度性能的上限在1k pods/s。在性能方面，原生 Kubernetes 默认调度器在 5000 个节点的集群中只能实现每秒 10个Pod 左右的调度吞吐量。
2. pod跟Node匹配在调度器中较为复杂。把这部分单独拆出来，单独成为组件scheduler，可以水平扩展。pod调度到节点和 抢占框架，Scheduler Framework有多个阶段，每个阶段都是pluggabled。
   调度多组件横向扩展，会存在并发冲突的问题, 冲突的解决在binder这里收敛。
3. dispatcher和 binder之所以是单组件，设计之初承接量是比较小的，可以保证吞吐相对符合要求。dispatcher需要为pod选择合适的scheduler进行调度决策。
4. scheduler分两层调度，unit scheduling和 pod scheduling。  <mark> 每个scheduler都有自己的全局view </mark>
   比如：离线业务 pod group gang语义的pod、ML训练任务job级别的语义、比如：训练池里的pod都在一个网段/交换机/机型。
5. 把Pod的状态分成三部分，dispatcher获取到pending状态的pods, 
{{% /callout %}}

### Dispatcher：应用排队、分发、节点分区。

Dispatcher的作用是为不同的pod找到合适scheduler进行调度。
{{< figure src="dispatcher.png" caption="详细设计dispatcher" theme="light" >}}

它主要由几个部分构成：Sorting Policy Manager, Dispatching Policy Manager, Node Shuffler, Scheduler Maintainer 和Reconciler。其中，
* Sort Policy Manager: 主要负责对应用进行排队，现在实现了如下排队策略， 如：FIFO，DRF/FairShare（没上生产环境）。后面会添加更多排队策略，如：priority value based等
* Dispatching Policy Manager: 主要负责分发应用到不同的Scheduler实例，现阶段是默认策略：LoadBalance（根据scheduler的负载，去平衡负责）， 后面会增强该功能，做成插件化配置模式；比如：gang pod group需要保证里面的pod都分配到一个scheduler。
* Node Shuffler：主要负责基于Scheduler实例个数，对集群节点进行Partition分组，每个节点在一个Partition里面，每个Scheduler实例对应一个Partition，
  Scheduler调度的时候会优先选择自己Partition节点，没有合适的情况下，才会去其他Partition节点。
  如果Node增删或者Scheduler个数变化，会基于实际情况重新分配节点；Partition规则现在是基于Scheduler个数平均分配，后面会增强，Partition策略可配置;
* Scheduler Maintainer：主要负责对Scheduler实例状态进行维护，包括Scheduler实例健康状况，负载情况，Partition节点数等；

{{% callout  note %}}
1. SortedQueue目前是一个FIFO队列，后续会抽象成Sorting PolicyManager。使用一个goroutine循环pop queue中的数据，分发到不同的scheduler。
2. Store缓存了调度器信息， 后续会在主流程上游添加sorting policy和dispatching policy，抽象成对应的Manager。
3. 目前scheduler maintainer和 store中都保存scheduler信息，后续会去掉store这部分，直接使用scheduler maintainer中的信息。
{{% /callout %}} 
 
### Scheduler：具体调度和抢占决策

Scheduler 主要负责为应用做出具体的 调度和抢占决策，但是不真正执行（执行者是 binder）。
{{< figure src="scheduler.png" caption="详细设计scheduler" theme="light" >}}

由两级框架组成： Unit scheduling framework 和 Pod scheduling framework。整个调度过程主要分为 
3 大部分：Node Organizing， Unit Scheduling 和 Unit Preempting。

- Node Organizing： 过滤节点减少后续流程计算量，以及为节点进行排序。主要有两类插件：
  - Locating plugins： 基于应用，过滤掉不符合要求节点，比如： Local PV，DaemonSet Pods，Resource Reservation， Rescheduling 等，共同点是： 可以基于应用信息，过滤掉大部分节点，减少后面流程计算量，提升调度吞吐；
  - Node Grouping plugins： 为通过 Locating plugins 的节点进行分组，比如： 节点节点剩余资源量进行分组，或者基于 Job level affinity 里面的拓扑信息进行节点分组等，为的是能更快调度上或者获得更好的调度质量。

- Unit Scheduling：基于应用请求，对通过 Node Organizing plugins 的节点进行匹配筛选和打分，
类似 k8s Scheduler framework，Unit scheduling 阶段也有两类插件：
  - Filtering plugins：基于应用请求，过滤掉不符合要求的节点；
  - Scoring plugins：对上面筛选出来的节点进行打分，选出最合适的节点；

- Unit Preempting：上面阶段无法调度，则会进入**抢占阶段**，尝试为待调度应用去抢占正在运行的应用实例。该阶段也有两类插件：
  - Victim Searching： 遍历集群节点，尝试搜索 victims （被抢占应用），看是否能找到节点和 victims；
  - Candidates Sorting：如果上面步骤找到了合适的节点和 victims，则会为这些 victims 进行排序（节点粒度），选出最合适的节点和 victims；

{{% callout note %}} 
1. 存在Node Organizing的原因是，加快node的匹配过程。诸如: 
1) daemonset pod通过node affinity去进行节点的过滤，但仍然需要遍历所有Node节点。
2) 将vk场景的node匹配操作前置，加快node的匹配过程。
2. 通常情况会把一个集群划作一个Node Group，还有一些其他的划分像 同一个网段下的Node也划分一个Node Group。
{{% /callout %}}
### Binder：冲突检查、抢占操作、应用绑定

{{< figure src="binder.png" caption="详细设计binder" theme="light" >}}

Binder 主要负责乐观冲突检查，执行具体的抢占操作（删除 victims），进行应用绑定前的准备工作，比如动态创建存储卷等，以及最终执行绑定操作。
Binder 主要有 ConflictResolver, PreemptionExecutor 和 UnitBinder 三部分组成。

-  ConflictResolver: 主要负责并发冲突检查，一旦发现冲突，立即打回，重新调度；Conflict resolver 有两大
类： Cross node conflict resolver 以及 Single node conflict resolver
  - Cross node conflict resolver: 负责检查跨节点冲突，比如： 某个拓扑域调度限制是否仍然能满足等， 
    由于该节点跨节点，Binder 必须串行执行；
  - Single node conflict resolver: 单节点内冲突检查，比如： 节点资源是否仍然足够等，该节点检查的逻
    辑限制在节点内部，所以不同节点的检查可以并发执行（unit 内 pods 调度到不同节点）；
- PreemptionExecutorperator: 如果没有冲突，同时应用需要抢占，则执行抢占操作，删除 victims，等待最终调度；
  - UnitBinder: 主要负责绑定前准备工作，比如：创建 volume 等，以及执行真正的绑定操作。
    现在的版本，Binder 里面还耦合了一个 PodGroup controller 实现，负责维护 PodGroup 的状态以及生命周期，
    后面会从 Binder 里面移除，独立成一个 Controller。

{{% callout note %}}
1. cross Node的冲突检查，需要串行执行。
2. 每一步的操作失败后，会打回，重新调度。
{{% /callout %}}

## Part 03: 内部实践
{{< figure src="services.png" caption="支撑的业务" theme="light" >}}

* 支撑的业务：微服务、有状态服务、大数据、机器学习等。
* 单集群规模：2w节点、100w Pod。
* 性能：单Scheduler实例高峰调度吞吐2k+ pods/s，多(3)实例4k+ pods/s。（实例数越多，冲突会更多，吞吐可能会下降。）
* 在离线资源流转达到从周级别到分钟级别的优化。
* 资源利用率
  * CPU利用率微服务常态60%+, 推广周40%左右(需要微拓扑绑定NUMA,无法shareCPU)。
  * 7W张GPU卡，利用率95%以上。

## Part 04: 未来工作
已经开源：持续投入(非KPI项目)、合作共建、丰富功能和完善文档。

时间表：
- 2024 Q1：实时数据接入、文档完善。
- 2024 Q2：资源预留。
- 2024 Q3：重调度、丰富Unit对象。
- 2024 Q4：多队列资源管理、DRF、FairShare等。

## 总结
* Godel Scheduler通过两级框架实现乐观并发和统一调度
* Dispatcher负责排队、分发和节点分区
* Scheduler执行具体调度和抢占决策
* Binder处理冲突检查、抢占操作和应用绑定
* 内部实践中展示高性能调度吞吐量
* 未来工作包括开源、社区维护和功能丰富化
* 发展方向涵盖资源预留、重调度和队列管理

## 相关仓库：
[kubewharf/godel-scheduler](https://github.com/kubewharf/godel-scheduler)

## 相关文章：
[Gödel Scheduler open-sourced: a unified scheduler for online and offline workloads | CNCF](https://www.cncf.io/blog/2024/04/02/godel-scheduler-open-sourced-a-unified-scheduler-for-online-and-offline-workloads/)


## 相关论文：
* Wu Xiang, Yakun Li, Yuquan Ren, Fan Jiang, Chaohui Xin, Varun Gupta, Chao Xiang, Xinyi Song, Meng Liu, Bing Li, Kaiyang Shao, Chen Xu, Wei Shao, Yuqi Fu, Wilson Wang, Cong Xu, Wei Xu, Caixue Lin, Rui Shi, and Yuming Liang. 2023. Gödel: Unified Large-Scale Resource Management and Scheduling at ByteDance. In Proceedings of the 2023 ACM Symposium on Cloud Computing (SoCC ‘23). Association for Computing Machinery, New York, NY, USA, 308–323. https://doi.org/10.1145/3620678.3624663