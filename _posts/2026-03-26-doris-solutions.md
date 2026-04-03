---
title: Doris 对一些数据库经典问题的解决方案
date: 2026-03-26 20:04:00 +08:00
permalink: /blogs/10029
tags: [database]
categories: [笔记]
---

学而时习之，不亦说乎

## Doris 是怎样实现主键模型的？

```
...
...
rowset-v3
rowset-v2 delete_bitmap-v3
rowset-v1 delete_bitmap-v2 delete_bitmap-v3
```

1. 写的时候，在老的版本上做一个标记，标记一个 rowset 里面那些行已经被删除了。
   这个标记叫 `delete bitmap`，它里面记录了的是行号（rowid）。
   - 性能优化tip：怎样根据 key 找到对应的行号？---> 引入了一个主键索引（primary key index），它的结构是类似 RocksDB Partitioned Index。
     相当于记录一个 key range 的位置（相比于记录每个 key 的位置会更省内存空间）。
   - 性能优化tip：主键索引上面还套了一个 bloom filter，来加速查询。
     - bloom filter 可以用于范围查询么？---> 不行。
2. 通常 Doris 不会读历史版本。如果读写几乎同时的话，读也会根据版本来判断要使用哪些 delete bitmap。
   - ps：schema change 和 compaction 也会产生读请求
   - 性能优化tip：一个 rowset 上有多个 delete bitmap，这样读的时候 delete bitmap 要合并。这个合并的结果会放到 cache。

这里的性能优化点基本都是空间换时间的策略。这个时候就会有内存和缓存问题。从测试角度看，这里就会有一些值得注意的风险点。

3. 并发写怎么处理？rowset 顺序怎么保证？
   - 首先写入是有事务的，事务提交的时候确认版本号，事务提交会走到 FE 上。（不过它还有个 publish 机制，只有 publish 了才会被读到。）
   - 先提交的事务版本就比较小，CDC 场景并发更新时就需要额外的机制来保证顺序。也就是 sequence 列，sequence 列的值由客户端控制。
4. update 和写并发的时候，官方说行为不确定。

5. 为啥存算分离模式下，主键会有一个表锁呢？而存算一体模式下没有类似的东西？
   - 这个锁的存在是因为在事务提交阶段，需要更新所有 tablet 上的 delete bitmap。并且每个导入计算 delete bitmap 的时候，需要依赖现有的所有 delete bitmap。
     让 AI 分析一下代码试试，看看能不能更加具体一点：核心问题还是 commit 阶段需要计算 delete bitmap。AI 的分析结论
     - delete bitmap update lock：负责 MoW delete bitmap 的最终正确性
     - commitLock：负责把同表 MoW commit 先串一下，减少远端锁竞争、异步任务交错和重试风暴
   - 这个问题和 2PC 也有点关系，单独开一节记录一下。

## Doris compaction 和 RocksDB compaction 有哪些异同？

基本目的是相同的：都是 append 写，磁盘友好；后台合并来避免读性能太差。

我觉得要区别它们的 compaction 策略，一个关键问题是看在哪一层，重复数据被彻底消灭？
Doris 是在 base compaction 把重复数据彻底消灭。一次 base compaction 可以说几乎是读取所有数据来进行的。
RocksDB(leveled compaction) 除了 L0，其它每一层内部的数据都是不重叠的。

让 AI 给我分析一下异同，还是 AI 专业一点。它从适用场景、触发机制、合并粒度、资源控制来分析。

综合一下 AI 的理解，Compaction 的目标是尽可能“及时”的“触发”，“选择”一部分数据，并稳定性的“执行”。
这样就和 AI 分析的维度对应上了。

## Doris 存算一体模式下的多副本是怎样保证一致性的？

- Doris 只需要两副本写成功，事务就可以提交了。
  - 那一个副本缺版本了，又有查询怎么办？ --- AI 说有个 tablet scheduler 会受到心跳，然后处理它，接着修复它。和我理解对得上。

## Doris 存算一体和存算分离的 2PC

Doris 存算一体导入流程是：prepare -> commit -> publish。它这里的 commit 相当于 2PC 的 prewrite/precommit。
不过这里有个点是 commit 这个操作是指 1 个 tablet 写入成功。而 1 个 tablet 写入成功也需要 2 副本写入成功才行。

存算分离的导入流程为啥没有 publish 阶段了？存算一体的 publish 阶段本质是为了协调每个 BE 的元数据达到一致的状态。
而在存算分离模式下，它的元数据都是从 meta-service 同步。

从 2PC 的角度看“为啥没有 publish 阶段”？因为存算分离没有 2PC 的说法了，它把事务交给 meta-service 这样一个中心节点来处理。
所有元数据的操作都需要提交到 meta-service。使用元数据的时候，都需要去 sync 一下（当然这里有 cache 层）。

再想一个问题，2PC 的 commit 阶段是如何保证“最终一致”的？基本思路，假设某一个节点在过程中挂了，它恢复之后，
去问 coordinator，最终是应该 commit 还是 rollback。
