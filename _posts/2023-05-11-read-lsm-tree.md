---
title: 读《The Log-Structured Merge-Tree》
date: 2023-05-11 20:05:00 +08:00
permalink: /blogs/10018
tags: [database, storage]
categories: [阅读]
---

## 先唠嗑

前几天，和 lonng 聊到 “我对 xx 很感兴趣，但一直没真正上手” 的话题，
lonng 说学东西没有奇技淫巧，主要就是要专注，一次做好一件事。后来又说到一个话题，
“科班和非科班”，聊了聊，感觉非科班确实不能作为不沉下心来的接口。中间好像还聊到，
机会是留给有准备的人的，而不是给一个机会让你去准备。嗯，都是通俗易懂的道理。

今天偶然又看 manateelazycat 写了一篇关注的文章。嗯，他们说的都挺有道理的。
lonng 还说很多人都无法直视自己的缺点。嗯，我觉得也挺对的。

我以前觉得“分布式”很牛，但工作来、工作去，也不知道它牛在哪。去年体验了一年，
对这东西更加怀疑了，真的很 interesting 么？回想到专注+奇技淫巧的话题，
自己好好学习可能就是最好的‘捷径’。

## 论文从这里开始

读这论文以及相关资料给我印象比较深的感受是

1. 论文里面描述的 LSM-tree 和我平常所听说的大不一样，memtable,sstable,compaction,
   这些概念在这个论文里面根本不存在，但确实又能找到它们一点影子。
   后来看了《LSM-based storage techniques: a survey》这篇论文后，
   对这个问题有了更好的认识。原来大家嘴里的 LSM-tree 都是特指当代的一些具体实现。
2. 学习了一些看存储产品的常见角度
   1. 有哪些常见的角度来评价一种磁盘数据结构呢？最佳工况是什么（负载）；性价比（性能）；
      并发控制（concurrency control）; 数据恢复（recovery）；可调性（tunability）。
      （这里不得不感叹一下，很多人都会抱怨 TiDB 性能不好、或玄学调参，
        但可能不会把这个作为一个评价数据库的角度，tunability 是个不错的发明。）
   2. 这篇论文建了一种数学模型来分析了磁盘数据结构的性价比（优势）。
      然后用这个模型来对比 LSM-tree 和 B-tree（或其他数据结构），有理有据。
   3. 性能分析的角度：I/O 成本、数据温度（Data Temperature），multiple-page block，
      这几个分析角度值得记住。不过，我也没有看这个分析的细节，以后有需要再来吧。
   4. 多看论文，可能以后就不会好奇：这人竟然还能从这个角度想问题！
3. 大部分东西的产生都是需求导向的，论文在一开始就介绍了 1996 年那个时候需求的变化，
   从而促使了他们研究了 LSM-tree 这样一种数据结构。有点哲学意味。

## LSM-tree 是什么？

LSM-tree 全名 Log-Structured Merge-Tree，一般直接用英文名称呼它。
它是一种以磁盘为基础的数据结构，旨在为较长周期内有高频写入的文件提供低成本的索引。
这两篇论文读下来，我理解，Log-Structured 的含义就在于它是顺序写的，
就像写日志那样，而 Merge 的含义在于这种数据结构里面自带了一个 merge 过程，
这个过程是必不可少的。论文里面原文是这样定义的：
> The Log-Structured Merge-tree (LSM-tree) is a disk-based data structure
designed to provide low-cost indexing for a file experiencing a high rate of
record inserts (and deletes) over an extended period.

## 为什么要有 LSM-tree？它有什么优势？

论文在一开始就说了为什么要有这种数据结构，因为有一类负载在现实场景中越来越多。
总的来说，LSM-tree 适合写多读少（就是读要有，但也不能特别多的场景）。
> The need to answer queries about a vast number of past activity logs implies
that indexed log access will become more and more important.

论文作者以 B-tree 作为基准来进行对比，B-tree 索引会增加 50% 的 IO 成本。
论文里面花了一个章节来论述 LSM-tree 的性价比，这个论文比较费脑，我只是匆匆扫了几遍。
但我发现它的分析角度还是比较值得我这个新手学习。

它先给出了一个性价比的分析方法，也就是 3.1 章节 The Disk Model。这里面分析了 I/O 成本，
数据温度（Data Temperature），也说了 Multi-page block I/O 的优势。
（他们把评价标准都想的这么清楚了，那设计出来的数据结构/算法肯定也就不会差了嘛。）
然后它把这个方法套在 LSM-tree 和 B-tree 上。它在第二章的时候还提到了
The Five Minute Rule，这个 rule 阐述了什么样的数据应该被缓存，也挺好玩的。
分析过程看起来还是有理有据的，它算是对性价比建模了。也就是这个过程比较费脑，我先跳过了。

## LSM-tree 的基本工作原理，LSM-tree 一些有趣的细节？

LSM-tree 有三个特点。论文里面的第二章也介绍了这种数据结构是怎样进行增删读写的，
还介绍了它的 rolling-merge 流程。根据 a-survey 这篇论文的说法，
它的 rolling-merge 和当下大家所说 level-merge 是比较相似的。

1. 通过 defer 和 batch 来低成本维护实时索引
> The LSM-tree uses an algorithm that defers and batches index changes,
cascading the changes from a memory-based component through one or more
disk components in an efficient manner reminiscent of merge sort.

2. 减少磁盘臂移动来减少 IO 成本
> The algorithm has greatly reduced disk arm movements compared to a traditional
access methods such as B-trees, and will improve costperformance in domains
where disk arm costs for inserts with traditional access methods overwhelm
storage media costs.

3. 适合写多读少（在介绍部分又强调了一次）
> However, indexed finds requiring immediate response will lose I/O efficiency in
some cases, so the LSM-tree is most useful in applications where index inserts are
more common than finds that retrieve the entries.

## 读《The Log-Structured Merge-Tree》

我读这一篇论文的主要目的主要是看看 memtable, sstable 这些概念是从哪里引入的。
因为我看 LSM-tree 的论文上根本没有提到这些概念。这篇论文的第二章 LSM-tree Basics
是我阅读的主要重点。它分两节，一节将 LSM-tree**s** 的历史，一节讲现在的 LSM-trees。

### LSM-trees 历史

Update 方式可以分为两种：in-place 和 out-of-place。
> In general, an index structure can choose one of two strategies
to handle updates, that is, in-place updates and out-of-place updates.

out-of-place 中的一种典型是 LSM-tree。它的好处是利用顺序 I/Os 来处理写入，
它对于恢复（recovery）也是更有利的。但是这种方式的读性能被牺牲了。
并且，它需要一种额外的数据组织过程来提升空间和读取效率。
> This design improves write performance since it can exploit
sequential I/Os to handle writes. It can also simplify the recovery
process by not overwriting old data. However, the major problem of this design
is that read performance is sacrificed since a record may be stored in
any of multiple locations. Furthermore, these structures generally require
a separate data reorganization process to improve storage and
query efficiency continuously.

这种顺序，非原地更新的思路很早就有了。看完这句，我似乎明白了 log-structured
的含义，因为 log 也是顺序往后写的。
> Later, in the 1980s, the Postgres project [65] pioneered the idea of
log-structured database storage.

在 LSM-tree 之前，这种 log-structured 的存储主要有几个问题。
最重要的就是读性能问题，因为相关的日志条目被打散了。另外一个是空间浪费。
尽管有各种各样的数据重组织过程，但没有一个很 principled 代价模型来分析这里面的
trade-offs，这让调优变得非常困难。

LSM-tree 算法里面自带一个 merge 的过程，这种设计在现在通常被称为 level-merge。
> However, as we shall see later, the
originally proposed rolling merge process is not used by to-
day’s LSM-based storage systems due to its implementation
complexity.

后来又有人设计了新的 merge 策略，比如 tiering-merge policy，有更好的写入性能。

### 当代的 LSM-trees

当代的实现和论文里面写的 rolling merge 过程有些不同，简化了并发控制以及数据恢复。
> However, today’s LSM-tree implementations commonly exploit the immutability
of disk components to simplify concurrency control and recovery.

当下的 LSM-tree 通常用 skip-list 或 B+-tree 来实现内存结构，
使用 B+-tree 或 SSTables 来实现磁盘结构。

有两种 merge 策略当下比较常见: leveling merge 和 tiering merge。
> Two types of merge policies are typically used in practice.

### 当下 LSM-tree 一些典型的优化手段

**Bloom Filter**
支持两种操作，插入，以及测试一个key在不在（会假阳，不会假阴）。
它可以大幅提升点差性能。当下的常见调优配置中，假阳概率通常 1% 左右。

**Partitioning**
应该可以理解为每一层是由多个 SSTable 组成，而不是一个。

**Concurrency Control and Recovery**
有加锁和多版本这两种模式。多版本模式在 LSM-tree 上工作良好。
并发 flush 和 merge 的实现通常和 LSM-tree 具体实现相关性较大。
由于这些操作会修改 LSM-tree 元信息，比如 SSTable 列表，
所以这些操作需要非常小心的同步。引用技术策略可以较好的防止一个正在使用的组件被删除。

为了简化恢复过程，现存系统通常采用 no-steal 缓冲区管理策略。也就是，
只有当所有的写事务都终止的时候，一个内存组件（memtable）才会被 flush。
所以恢复时只需要做 redo，而不需要 undo。对于分区的 LSM-tree，
恢复时还依赖“存储了 LSM-tree 结构变更的” metadata log。

### 其它一些有趣的内容

这篇论文后面还把 LSM-tree 的优化手段进行了分类。读放大，merge 策略，
硬件适配，特殊负载，自动调优（调节阀太多，人很难调），二级索引。

未来的研究方向里面说到，目前还没有人给这些 DB 做一个“全面的性能评估”，
这里还提到一个词 tunability，TiDB 其实也有这个问题，参数太多。
> It is not clear how the improvements would compare against a well-
tuned baseline LSM-tree for a given workload.
> Moreover, many of the improvement proposals have primarily evalu-
ated their impact on query performance, with space utiliza-
tion often being neglected.

## 后续

下面去学学 B+-Tree 吧。
