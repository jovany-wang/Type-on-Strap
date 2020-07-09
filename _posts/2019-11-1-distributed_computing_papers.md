---
layout: post
title:  "Papers I Have Read"
date:   2019-11-1
---

<p class="intro"><span class="dropcap">In </span>this post，I will summarize the papers about distributed computing that I have read. On the one hand, this can help me to look some thing up when I need, and on the other hand, I will give some my personal views or summaries on them.</p>

### 1. DAGuE: A generic distributed DAG engine for high performance computing
    Conf: IPDPS 
    Date: 2011 
    Author: George Bosilca AND more 
    Original: https://jovany.wang 

#### Abstract
The frenetic development of the current architectures places a strain on the current state-of-the-art programming environments. Harnessing the full potential of such architectures has been a tremendous task for the whole scientific computing community.

We present DAGuE a generic framework for architecture aware scheduling and management of micro-tasks on distributed many-core heterogeneous architectures. Applications we consider can be represented as a Direct Acyclic Graph of tasks with labeled edges designating data dependencies. DAGs are represented in a compact, problem-size independent format that can be queried on-demand to discover data dependencies, in a totally distributed fashion. DAGuE assigns computation threads to the cores, overlaps communications and computations and uses a dynamic, fully-distributed scheduler based on cache awareness, data-locality and task priority. We demonstrate the efficiency of our approach, using several micro-benchmarks to analyze the performance of different components of the framework, and a Linear Algebra factorization as a use case.
