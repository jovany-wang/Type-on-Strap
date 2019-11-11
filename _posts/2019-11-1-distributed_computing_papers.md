---
layout: single
title:  "我看的一些论文"
date:   2019-11-1
---

<p class="intro"><span class="dropcap">这</span>篇文章，我对自己看过的分布式计算系统的一些论文进行一个简单的总结。一方面有便于自己在需要的时候查阅，另一方面也总结一点自己对每片论文的一些看法和归纳吧。</p>

### 1. DAGuE: A generic distributed DAG engine for high performance computing
    会议: IPDPS
    时间: 2011
    作者: George Bosilca AND more

#### Abstract
The frenetic development of the current architectures places a strain on the current state-of-the-art programming environments. Harnessing the full potential of such architectures has been a tremendous task for the whole scientific computing community.
We present DAGuE a generic framework for architecture aware scheduling and management of micro-tasks on distributed many-core heterogeneous architectures. Applications we consider can be represented as a Direct Acyclic Graph of tasks with labeled edges designating data dependencies. DAGs are represented in a compact, problem-size independent format that can be queried on-demand to discover data dependencies, in a totally distributed fashion. DAGuE assigns computation threads to the cores, overlaps communications and computations and uses a dynamic, fully-distributed scheduler based on cache awareness, data-locality and task priority. We demonstrate the efficiency of our approach, using several micro-benchmarks to analyze the performance of different components of the framework, and a Linear Algebra factorization as a use case.
