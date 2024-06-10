---
layout: post
title:  "论文翻译:Virtual Consensus in Delos"
date:   2021-02-22
---

Mahesh Balakrishnan, Jason Flinn, Chen Shen, Mihir Dharamshi, Ahmed Jafri, Xiao Shi Santosh Ghosh, Hazem Hassan, Aaryaman Sagar, Rhed Shi, Jingming Liu, Filip Gruszczynski Xianan Zhang, Huy Hoang, Ahmed Yossef, Francois Richard, Yee Jiun Song Facebook, Inc.

# 概要
基于一致性算法的复制系统(复制系统指那些类似于主备，多主复制等)是复杂的，臃肿的，以及一旦部署则难以升级维护的。因此，已经部署过的老系统无法从具有创新性的研究中获益，新的一致性协议也很少被投入真正的生产环境中。我们提出一种通过虚拟化共享日志API来虚拟化一致性的方案，允许用户服务在不停机的情况下更改一致性协议。虚拟化机制将一致性逻辑分为2部分，一部分是VirtualLogs，一个通用，可复用和可配置的抽象层；一部分是可插拔的排序协议，称为Loglets。