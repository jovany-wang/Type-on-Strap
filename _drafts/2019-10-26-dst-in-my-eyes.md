---
layout: post
title:  "The Dst in My Eyes"
date:   2019-10-26
---

<p class="intro"><span class="dropcap">At</span> the beginning of this year(2019), our project in company needed to use Redis as a backend storage, but we required 2 functionalities of it: strong consistency and table concept.</p>

For the duration of it, we have a very strong requirement on strong consistency, while not having so strong requirement on table concept. But we have to write the table-like concept in our buiness if there is not a table concept in Redis. That's a troublesome thing to me. So the idea **to write a light weight storage with strong consistency and table concept** came out naturally. I had givan a name for it called [Dst](https://github.com/dst-project/dst), which is combined with the first letters of the words `Distributed Store with Table`.

The targets of Dst in my mind are:
1. Redis-like APIs: avoid cost to understand.
2. Trade performance off for greater consistency.
3. Support simple table concept.


### Redis-like APIs
It is too natural to use the Redis API, so it is reasonable that Dst uses the Redis-like APIs. If you want to write a value typed as string to Dst store, you can use the following commands:
```bash
> str.put "k1" "v1"
> "ok"
> str.get "k1"
> "v1"
```
Does those look like Redis commands? But there is a little difference between both of them. Dst command is consists of 2 words, the 1st one is the word `str` to indicate the type of the object, followed by the word `put` to indicate the operator that you'd like to invoke on the object. It looks very like the OO(Object-Oriented) code that you write. Actually, I want to provide a graceful APIs to make your commands look like code in command line.
```bash
> str.put("k1", "v1")
> "ok"
> str.get("k1")
> "v1"
```
Without doubt, the same is true for other type operators:
```bash
> dict.put "dict_1" "k1" "v1" "k2" "v2"
> "ok"
dict.get "dict_1"
> {"k1" : "v1" , "k2" : "v2"}
dict.get "dict_1" "k2"
> "v2"
```

### Strong Consistency
It's difficult to archive the goal of both making a great strong consistency and having great performance, which is limited by both of the CAP theory and code implementation complexity. So I was considering that wasting a bit little of performance to meet our strong consistency requirements at that time. I could tolerate performing the next actions after a log being synchronized to the most of the cluster nodes. That was the actul plan on strong consistency! But it's also not easy to archive this goal. It is not likely a single node program with one kv store instance. With this distributed design, the kv store instance is just a FSM in the distributed system, and the FSM is one of the lightest things.

I before imaged the design that every store instance is as a RAFT instance in a RAFT group, and so that the data synchronizing between store instances is using RAFT synchronization. But I thought there would be unnecessary cost by RAFT synchronization. That's not very perfect to me. Regarding the performance of store server, I prefered using `AppendDataLogs` handed by ourselves to using RAFT. And then would use a RAFT group as the coordinators server group to do master-slection, requesting recovrying from failures and managing all the store servers. We called the group of coordinators as meta servers, or meta server group. And the meta servers would use RAFT to gurantee the consistency of meta servers. That wouldn't effect the performance because meata servers is not used frequently.

Maybe you are so curious about the reason for removing the RAFT and then adding the RAFT back. But it's not contradictory at all:
(1) We should gurantee the consistency between meta servers.
(2) We needed to synchronize a large of data between store servers, so synchronizing so large data between store servers by RAFT is not allowed here while synchronizing data between meta servers making a lot of senses.
 
### Supporting table concept
Most guys throught it's not reasonable to give a table concept for Dst because the usage of it looks very like a relationship database. It allows you define and create a table:
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
OK, there is a table object named `task_table` lies in the Dst system. And we can use the table with the sql-like operators.
```bash
> task_table.add "00001", "22222", "my_task", 3, ["1", "2"]
> "ok"
> task_table.query (*) when driver_id == "22222"
> task_id      driver_id     task_name   num_return      arguments
> "00001"      "22222"       "my_task"       3           ["1", "2"]
```

Now as you can see above, it's graceful to do operations on Dst(or Dst table). The original essence of the other types of Dst is a k-v pair. So the table concept is also stored in memory, and this is the biggest difference between Dst and othe relationship datbases even if there some opertions around disk like snapshot and checkpoint. But the operations of your queries are both executed in memory.

There are some words from others like `Is the memory enought if table is too large?`. I could see it, but the target of the Dst is just an in-memory k-v storage instead of a real realtionship database. And it aims to address the question: How to use the Redis(or other in memory kvs) with table concept?
As for the question `Is the memory enought if table is too large?`, I still don't know as well.

That is all about Dst, and how do you think of it?
