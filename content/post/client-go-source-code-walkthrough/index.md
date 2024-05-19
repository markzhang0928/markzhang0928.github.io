---
title: Client-go源码走读
summary: 学习笔记类这段时间梳理过很多, 没什么新意，简单记录一下。
date: 2024-05-19

authors:
  - admin

tags:
  - Informer List-Watch
  - K8S WorkerQueue
  - Kubernetes

---


## Informer机制

{{< figure src="informer.png" caption="informer整体架构" theme="light" >}}

Kubernetes使用Informer代替Controller去访问API Server，Controller的所有操作都和Informer进行交互，而Informer并不会每次都去访问API Server。 
- Informer使用ListAndWatch的机制，在Informer首次启动时，会调用LIST API获取所有最新版本的资源对象，然后再通过WATCH API来监听这些对象的变化，
并将事件信息维护在一个只读的缓存队列中提升查询的效率，同时降低API Server的负载。 
- 除了ListAndWatch，Informer还可以注册相应的事件，之后如果监听到的事件变化就会调用对应的EventHandler，实现回调。


Informer主要包含以下组件:
1) Controller：Informer的实施载体，可以创建reflector及控制processLoop。processLoop将DeltaFIFO队列中的数据pop出，首先调用Indexer进行缓存并建立索引，然后分发给processor进行处理。
2) Reflector：Informer并没有直接访问k8s-api-server，而是通过一个叫Reflector的对象进行api-server的访问。Reflector通过ListAndWatch监控指定的 kubernetes 资源，当资源发生变化的时候， 
   例如发生了Added资源添加等事件，会将其资源对象存放在本地缓存DeltaFIFO中。
3) DeltaFIFO：是一个先进先出的缓存队列，用来存储Watch API返回的各种事件，如Added、Updated、Deleted。
4) Indexer：Indexer使用一个线程安全的数据存储来存储对象和它们的键值。需要注意的是，Indexer中的数据与etcd中的数据是完全一致的，这样client-go需要数据时，
   无须每次都从api-server获取，从而减少了请求过多造成对api-server的压力。一句话总结：Indexer是用于存储+快速查找资源。
5) Processor：记录了所有的回调函数（即ResourceEventHandler）的实例，并负责触发回调函数。

Informer负责与k8s apiserver 进行watch操作，watch的资源，可以是k8s内置资源对象，也可以是CRD。

Informer 是一个带有本地缓存以及索引机制的核心工具包，当请求为查询操作的时候，会优先从本地缓存内去查找数据，而创建、更新、删除这类操作，
则会根据事件通知写入到队列DeltaFIFO中，同时对应的事件处理后，更新本地缓存，使本地缓存与ETCD的数据保持一致。

从上图就能看出来，Informer是有多个组件构成的，咱们就先简单了解一下都是什么组件，减轻了APIServer的数据交互压力。

**Reflector** ： 使用List-Watch来保证本地缓存数据的，准确性、顺序性和一致性的，List对应资源的全量列表数据，Watch负责变化部分的数据，
当watch的资源发生变化时，触发变更的事件，比如Added， updated， delete事件，并将资源对象的变化事件存放到本地队列DeltaFIFO中。

**DeltaFIFO** ：是一个增量队列，记录了资源变化的过程，Reflector就相当于队列的生产者。这个组件可以拆分两部分来理解，FIFO是一个队列，
拥有基本方法，例如ADD，UPDATE，DELETE，LIST，POP，CLOSE等，delta是一个资源对象存储，保存存储对象的消费类型，比如 added， updated， deleted等

**Indexer** ：用来存储资源对象并自带索引功能的本地存储，Reflector从deltaFIFO中将消费出来的资源对象存储到Indexer，indexer与etcd中的数据保持完全一致，
从而client-go可以本地读取，减少k8s apiserver的数据交互压力。

## List-Watch机制
List-Watch机制是Kubernetes中的异步消息通知机制，通过它能有效的确保了消息的实时性、顺序性和可靠性。
ListAndWatch的主要逻辑分为三大块：

A. List操作（只执行一次）：
```
（1）设置ListOptions，将ResourceVersion设置为“0”；
（2）调用r.listerWatcher.List方法，执行list操作，即获取全量的资源对象；
（3）根据list回来的资源对象，获取最新的resourceVersion；
（4）资源转换，将list操作获取回来的结果转换为[]runtime.Object结构；
（5）调用r.syncWith，根据list回来转换后的结果去替换store里的items；
（6）调用r.setLastSyncResourceVersion，为Reflector更新已被处理的最新资源对象的resourceVersion值；
```
B. Resync操作（异步循环执行）:
```
（1）判断是否需要执行Resync操作，即重新同步；
（2）需要则调用r.store.Resync操作后端store做处理；
```

C. Watch操作（循环执行）：
```
（1）stopCh处理，判断是否需要退出循环；
（2）设置ListOptions，设置resourceVersion为最新的resourceVersion，即从list回来的最新resourceVersion开始执行watch操作；
（3）调用r.listerWatcher.Watch，开始监听操作；
（4）watch监听操作的错误返回处理；
（5）调用r.watchHandler，处理watch操作返回来的结果，操作后端store，新增、更新或删除items；
```

List-Watch，顾名思义，它是分为两部分的。 

- List负责调用资源对应的K8S APIServer的RestFul API获取全局数据列表，并同步到本地缓存中。
- Watch负责监听资源的变化，并调用相应事件的处理函数进行处理，同时更新本地缓存，使本地缓存与Etcd中数据，保持一致。
- List是基于HTTP中的短链接实现，Watch则是基于HTTP长链接实现，Watch使用长链接的方式，极大的减轻了Kubernetes APIServer的访问压力。

### K8S util workQueue和一般的队列实现有什么区别？
功能更加丰富，支持有序、去重、并发安全等特性
* 有序：按照添加顺序处理元素（item）
* 去重：相同元素在同一时间不会被重复处理，例如一个元素在处理前被添加了多次，它只会被处理一次。
* 并发性：多生产者和多消费者
* 标记机制：支持标记功能，标记一个元素是否被处理，也允许元素在处理时重新排队。
* 通知机制：ShutDown方法通过信号量通知队列不再接收新的元素，并通知metric goroutine退出。
* 延迟：支持延迟队列，延迟一段时间后再将元素存入队列。
* 限速：支持限速队列，元素存入队列时进行速率限制。限制一个元素被重新排队（Reenqueued）的次数。

### 为什么需要Resync机制?
- Resync 机制的引入，定时将 Indexer 缓存事件重新同步到 Delta FIFO 队列中，在处理 SharedInformer 事件回调时，让处理失败的事件得到重新处理。
- 并且通过入队前判断 FIFO 队列中是否已经有了更新版本的 event，来决定是否丢弃 Indexer 缓存不进行 Resync 入队。
- 在处理 Delta FIFO 队列中的 Resync 的事件数据时，触发 onUpdate 回调来让事件重新处理。


### 为什么不从etcd重新拉取？
* 问：因为watch是长连接，如果网络发生波动连接断了呢，本地缓存因为网络断了那段事件肯定就丢失了一部分事件导致etcd和本地数据不一致，这种情况下如果网络稳定下来或者在网络波动间隙，定时从etcd拉取的方案还能继续保持本地缓存和etcd一致，为什么不这样做呢？
* 答：informer能保证通过list+watch不会丢失事件，如果网络抖动重新恢复后，watch会带着之前的resourceVersion号重连，resourceVersion是单调递增的，apiserver收到该请求后会将所有大于该resourceVersion的变更同步过来。　
* 另外好像网络长期中断的话会导致informer重新初始化也就会重新list。

### 总结：
- Reflector的职责很清晰，要做的事情就是保持DeltaFIFO中的items持续更新，具体实现是通过ListerWatcher提供的list-watch能力来列选指定类型的资源，产生一系列Sync事件，通过ResourceVersion来开启监听过程，
- 监听到新的事件后，会和前面提到的Sync事件一样，通过DeltaFIFO提供的方法构造相应的DeltaType到DeltaFIFO中。
- 监听到更新事件时，也不是直接修改DeltaFIFO中已经存在的元素，而是添加一个新的DeltaType到队列。DeltaFIFO中添加到新的DeltaType时也会有一定的去重机制。
- 监听到新事件时，会拿着对象的新ResourceVersion重新开始一轮新的监听过程。
- watch调用也有超时机制。