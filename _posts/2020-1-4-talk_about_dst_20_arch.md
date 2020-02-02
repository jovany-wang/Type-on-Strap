---
layout: post
title:  "Distkv2.0架构漫谈"
date:   2020-1-4
---

[Distkv](https://github.com/distkv-project/distkv)到现在也做了快半年了，虽然大家都只是在周末和空闲时间才能参与进来，但是我还是看到了Distkv的进步：从一个什么都不像的东西做成了现在这样初具雏形。当然更让人开心的事，Distkv2.0的架构改造也即将和大家见面，那就让我们来谈谈Distkv2.0架构的一些设计和想法吧。

### 组件架构
![组件架构图]({{"assets/img/for_posts/20190104/dst_component_arch.png" | relative_url }})

首先看一下整个Distkv系统的组件级别的架构，整体架构还是偏像传统意义上分布式系统。

左边下面红色的区域是[drpc](https://github.com/jovany-wang/dousi)(现在改名叫Dousi RPC)，它的作用是来承担着整个Distkv系统的底层通信任务。 接着是整个Distkv的core，Distkv的core主要分为3个server: `meta server`, `store server`和`proxy server`。
- meta server是一组raft集群，它在整个Distkv中所起的作用是作为全局决策，服务注册与发现，Distkv集群管理。当然你可以理解为它是一个zk或者etcd等。
- store server是整个Distkv的存储服务器，所有的数据全部存储与store server中。store server可以有很多分片，然后一个store server的实例可以包含n个分片，这部分在后面再介绍。
- proxy server就很简单了，它是一组无状态的服务器，用于代理client端发来的请求。无状态的proxy server可以很轻松地进行横行的扩展。

接着最上面一层就是各个语言的client SDK，这部分就不详细赘述了。
最右边一层灰色的是目前还没建设而是长期将建设的部分。`metrics`是所有的工业级分布式系统必备的用于系统指标上报统计等。`studio`和`anlyzer`是主要是用于进行集群操作和问题分析排查的一套工具链。`studio`可以类比于其他分布式系统的dashboard。

### 分布式架构
![集群进程架构图]({{"assets/img/for_posts/20190104/distkv_arch_overview.png" | relative_url }})

一个request经过的流程大概是：
- (1) client通过用户自己配置的LBS将request打在某一个proxy server上。
- (2) proxy server根据这个request的key，进行计算该request应该在哪个shard实例上，然后再根据meta server给proxy同步的shard和store server的关系来决定出这个reuqest应该打在哪个store server上。
- (3) proxy server将这个request打到相应的master store server中。
- (4) master将request按照配置的同步策略将request进行同步到slaves中。
- (5) store server对该request进行处理完，然后将response发回到对应的proxy server。
- (6) proxy server将response发回给该client。

### 单机线程模型

### 存储引擎

### 其他能力的升级

#### RPC升级
#### Command Line Tool

### Thanks for
