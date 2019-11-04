---
layout: post
title:  "我眼中的Dst"
date:   2019-10-26
---

<p class="intro"><span class="dropcap">大</span>概是今年年初的时候，我们的项目要用到redis，不过要求redis有2个功能:高可用和table的概念。</p>


其中高可用是一个非常强的需求，而对table的需求没那么强，但是没有table类型会使得我们在使用redis的时候要模拟出这样一个概念，也是很头痛的事情。所以，我就有了做一个**高可用的，支持table类型的kv内存数据库**的想法。给它取了个名字叫[Dst](https://github.com/dst-project/dst),取Distributed Store with Table中3个单词的首字母。

对于Dst的目标，我的设定是以下3点：
1. 类redis接口，让用户能直接上手。
2. 能容忍牺牲一点性能来换取高可用方案。
3. 支持table概念。

### 类redis接口
redis的接口真的太自然了，所以Dst采用类似于redis的接口似乎是顺其自然的事情。比如，我们要想往Dst数据库中写一个字符串类型的value，可以这样做:
```bash
> str.put "k1" "v1"
> "ok"
> str.get "k1"
> "v1"
```
是不是太像redis啦？不过在命令的关键词上有一点区别，我们的命令是由2个词组成，第一个词是要操作的类型`str`，接着是要在这个类型上操作的命令`put`，是不是很像你写代码？当然，我还想同时实现这样一个的优雅的接口，让你命令行操作起来像极了写代码:
```bash
> str.put("k1", "v1")
> "ok"
> str.get("k1")
> "v1"
```
当然，对于其他的类型操作也是类似的：
```bash
> dict.put "dict_1" "k1" "v1" "k2" "v2"
> "ok"
dict.get "dict_1"
> {"k1" : "v1" , "k2" : "v2"}
dict.get "dict_1" "k2"
> "v2"
```

### 高可用
高可用其实是一个老大难的问题，追求极致性能前提下，再要想做到非常高的可用性是太难了。因此我是想着可以稍微放弃一点点性能来满足我们的高可用的要求，具体的方案就是我可以容忍一条log被正确同步到其他机器上之后再进行下一步。其实这就是我们高可用的本质，但是要做好这件事情也是很麻烦的。它不像单机版程序，有一个kv store的instance就足够。在这种设计下，一个kv store的instance仅仅是一个分布式系统中的状态机而已，而状态机恰恰又是整个系统中最轻量的一环。

起初我设想的分布式方案是每一个store instance作为一个raft组的raft实例，使用raft进行各个store instance间的数据同步。不过我认为这种方案的实现复杂的程度较高，因为你的raft组需要用户协同同步数据，这个在我看来是没必要的。对于性能敏感的store server，我更倾向于store server直接同步log(直接是指不采用raft或者其他一致性协议)，这样的复杂程度和难度肯定比raft低，出错率以及性能应该也会好一些(这个结论仅仅是我我自己意淫的，没有做过任何实验验证和调查，也欢迎有人做实验，或者做调研来打脸)。如果store server直接同步log而不依赖raft，那么势必需要一个中心节点(meta server)来做coordination，用于出现故障后如何进行恢复，出现不一致行为后如何裁判等一系列仲裁和协调工作。meta server可以直接使用zookeeper，不过我的想法并没有使用zookeeper，而是自己写一个使用raft协议的meta server。

想必你也很奇怪，前面的store server我把raft去掉，为什么meta server又把它加回来？其实一点也不矛盾：
(1) 首先meta server也是需要高可用的，否则整个Dst集群还是无法保证高可用。
(2) store server需要同步的数据是大量的，我不希望他走raft协议，而meta server同步的数据是很少的，他只同步metadata,例如集群信息等，所以我可以接受使用raft同步这些meta server数据。

### 支持table类型
刚开始很多人认为这样一个类型放到一个Dst中是不合适的，因为它的用法实在是太像一个关系型数据库了。它允许你定义并创建一个table:
```bash
mytables.sc
table task_table {
  [p]task_id: string;
  [i]driver_id: string;
  task_name: string;
  return_num: int;
  arguments: [string];
}

table driver_table {
 [p]driver_id: string;
 driver_name: string;
 actor_num: int;
};

> table.create task_table driver_table from mytables.sc
> "ok"
```
此时在Dst数据库中,便创建出一名为`task_table`的table对象了。接着我们可以像普通关系型数据库那样进行一些简单的类SQL操作了。
```bash
> task_table.add "00001", "22222", "my_task", 3, ["1", "2"]
> "ok"
> task_table.query (*) when driver_id == "22222"
> task_id      driver_id     task_name   num_return      arguments
> "00001"      "22222"       "my_task"       3           ["1", "2"]
```
现在至少你可以看到，在Dst上的table类型如何操作，也很简洁，优雅。Dst的其他类型的k-v本质还是基于内存的，因此我们的这样的table类型也是基于内存，这是Dst和关系型数据库的最大区别，尽管Dst会有snapshot之类的持久化，但Dst在你需要query的时候，是真真切切的在内存中查的，而其他关系型数据库的query至少有概率地需要从磁盘加载块数据。
不过对于“table类型数据量大，内存是否够用的”，我是可以理解这种疑问的，只不过我所期望的Dst应当是真的是内存数据库，因此它不适用于你把它当做一个关系型持久化数据库去用，而是作为我开篇时候提到的遇到这样的问题的一种解决方案罢了:即它是来解决我们需要在内存kv上使用一些table类型的高效操作,因此他应该和你存kv的数据量没有本质的差距。至于table数据量如果真的大了,Dst该怎么办？我也没有想清楚。

所以，Dst在我的设计里，它应该是一个内存数据库，尽管可能会去持久化日志;其次它还应当是一个分布式的高可用存储，以便我们轻松地为中小企业提供便捷可靠的内存数据库方案；此外它支持了table的类型让我们某些特定情况下，可以不动用那么heavy的关系型数据库来做一些关系存储的事情。你认为呢？

