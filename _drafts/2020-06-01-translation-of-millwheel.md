---
layout: post
title:  "[翻译]MillWheel: Fault-Tolerant Stream Processing at Internet Scale"
date:   2020-05-16
---
本文是我在阅读该论文过程中，顺手对Millwheel论文的翻译。

# MillWheel: Fault-Tolerant Stream Processing at Internet Scale

# 摘要
Millwheel是一个在Google内部广泛使用的一个用于构建低延时数据处理应用的计算引擎。
# 1. 介绍

# 2. 需求与动机

# 3. 系统简介

# 4. 核心概念
## 4.1 Computations
## 4.2 Keys
## 4.3 Streams
## 4.4. Persistent State
## 4.5 Low Watermarks
Low Watermark给computation提供了其对于即将到来的数据的一个时间边界。
**定义**: 根据计算的数据流，我们递归地定义Low Watermark。对于给定的一个computation A，设一个关于A的时间戳oldest-work(A), 则oldest-work(A)是一个时间戳，其具体的值是A中最老的还未完成的那条数据的时间戳(最老的那条数据可能是还未到达，或者已经存到了本地，或者正在等待发往下游)。紧接着，我们可以给出A上的low watermark的定义：  
low-watermark(A) = min {oldest-work(A), low-watermark(A)}  
其中C的输出给到A，即C是A的上游（注意，这里应当是C是A的全部上游，即A不能有多个上游）。
如果A没有上游，则low watermark和oldest-work相等。



## 4.6 Timers


# 5. API

# 6. 容错机制

# 7. 系统实现细节

# 8. 性能评测

# 9. 我们所做的一些工作

# 10. 引用
