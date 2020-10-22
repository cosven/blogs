---
title: 以复现 bug 为导向的 RocksDB 学习笔记
date: 2020-09-25 10:00:00 +08:00
permalink: /blogs/10007
tags: [rocksdb, tikv]
---

我要复现的 bug 是这个：https://github.com/tikv/tikv/issues/8243 ，但我对 rocksdb
不是很了解，年初有一个月接触了它，但现在几乎忘得差不多了。

我觉得我需要先对 RocksDB 有一个整体的认识，于是就有了下面这个笔记。
不过目标应该是解决下面几个问题

- [x] 什么时候会发生 compaction？
    对于 leveled compaction 来说，文件数达到一定数量的时候。
- [x] L0, Lmax 哪一层会有重叠？(怎样判断 compaction 的结果是否正常？)
    L0 会有重叠，是从 memtable 直接刷下来的。
- [x] RocksDB 对 key/value 的编码规则？
    经过测试，seq 从 1 开始涨。
- [x] inode 和 igeneration 为什么会重复？
    inode 会被重复利用，igeneration 随机生成
- [x] 满足什么条件可以复现？
    1. inode 被重用了（老的 sst 被删除，新的 sst 使用了相同的 inode number）
    2. igeneration 和老的 sst 文件一样
    3. 新的 sst 被 compaction 了
- [x] 提升复现概率的办法 -> 有理论，但实践困难

## RocksDB Overview

https://github.com/facebook/rocksdb/wiki/RocksDB-Basics

注：这个文档的分块可能是一个值得学习的地方。

**RocksDB 的概念**

* RocksDB 会适配各种存储介质，一开始是专注在 fast storage（特别是 flash storage）。
* 它是一个 library
* 支持 point lookups，range scan，ACID

**RocksDB 起源**：从 leveldb 和 hbase 中借鉴了代码和思想。

**RocksDB 设计目标**

1. Performance

    * high random-read workloads
    * high update workloads
    * combination of both
    * 可以通过参数来调教，以适配某种 workload

2. 产品生态

    * 内置部署和调试工具

3. 向后兼容

**RocksDB 架构**

常见接口：`Get(key)`, `NewIterator()`, `Put(key, val)`, `Delete(key)`, and
`SingleDelete(key)`。

基本数据结构：memtable，sstfile，logfile

**RocksDB 特性**

* Column Families
* Updates：它会自动新建、或覆盖-删除老kv
* Gets, Iterators 和 Snapshots
* Transaction
* Prefix Iterators：使用布隆过滤器
* Persistence：WAL
* Data Checksuming：为每个 SST file block 计算 checksum
* Multi-Threaded Compactions
* Compaction Styles
* Metadata Storage
* Avoiding Stalls
* Compaction Filter
* ReadOnly Mode
* Database Debug Logs
* Data Compression
* Full Backups and Replication
* Support for Multiple Embedded Databases in the same process： partition/shard
* Block Cache -- Compressed and Uncompressed Data
* Table Cache
* I/O Control
* Stackable DB: 接口层面
* Memtables
* Merge Operator

**RocksDB 工具**

sst_dump/ldb

**RocksDB 测试**

单测 + db\_stress

**RocksDB 性能**

db\_bench

## RocksDB Compaction

https://github.com/facebook/rocksdb/wiki/Compaction

RocksDB 有几种 compaction 方式，各个算法主要是在权衡 读、写和空间放大。
TiKV 使用的是 level compaction。

看这个之前好像得先复习一下 LSM Tree。

### LSM Tree (Log Structured Merge Trees)

[LSM 算法的原理是什么？ - wuxinliulei的回答 - 知乎](https://www.zhihu.com/question/19887265/answer/78839142)

**Levelled Compaction**：每层维护指定的文件个数，第一层 key 会有重叠，其它层都不重叠。

## RocksDB Block Cache

https://github.com/facebook/rocksdb/wiki/Block-Cache
因为 bug 最终是 block cache 出了问题，所以这里看看 block cache 里面存的是什么，key 又是什么。

cache 里面缓存的是 sst 的 data block，cache key 是 sst 文件 inode + data block offset。

https://github.com/facebook/rocksdb/issues/7405#issuecomment-694595587
> block cache keys are composed of (inode number, inode generation (4 bytes), file offset)

### SST(Static Sorted Table) 结构
https://github.com/facebook/rocksdb/wiki/A-Tutorial-of-RocksDB-SST-formats
https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format

```
<beginning_of_file>
[data block 1]
[data block 2]
...
[data block N]
[meta block 1: filter block]                  (see section: "filter" Meta Block)
[meta block 2: index block]
[meta block 3: compression dictionary block]  (see section: "compression dictionary" Meta Block)
[meta block 4: range deletion block]          (see section: "range deletion" Meta Block)
[meta block 5: stats block]                   (see section: "properties" Meta Block)
...
[meta block K: future extended block]  (we may add more meta blocks in the future)
[metaindex block]
[Footer]                               (fixed size; starts at file_size - sizeof(Footer))
<end_of_file>
```

## Leveled Compaction 流程
https://github.com/facebook/rocksdb/wiki/Leveled-Compaction

特点
1. L0 key 会有重叠，其它层都不会，且都是排好序的 
2. 每一层的大小是可以配置的，Compaction 会严格按照这个大小来进行
3. 下一层存储的数据量最大，且一般来说，是上一层的 10 倍

流程
1. L0 文件数达到 `level0_file_num_compaction_trigger` 时触发当一层的 size
  达到一定程度时；释放 snapshot 的时候也会尝试对检测 bottommost 层进行 compaction。
2. 如果 compaction 后，L1 也达到上限，就会从里面选文件和 L2 merge

性能与局限
1. L0 -> L1 的 compaction 不能并行，有个 max_subcompactions 的方式来加速（不是很懂，先忽略）

L0 文件堆积问题：会影响读性能；这个时候 Leveled Compaction 可能会在 L0 内部进行 compaction。

### Compaction 和 Block Cache 的关系是什么？

没找到什么合适的资料。主要几个问题

1. 是不是只有 ingest 的时候，才会出现这个问题？不是

## inode 相关基础

https://www.howtogeek.com/465350/everything-you-ever-wanted-to-know-about-inodes-on-linux/

大概看了一遍，有一个结论：inode number 会被重用。

## Debug 相关工具

### 如何将一个 RocksDB 的 key 转换成人可读的东西？

* sst_dump 工具可以从一个 sst 文件中把 kv 的 key 和 value 以 hex 的形式输出
* tikv-ctl 可以将一个 hex 形式的 key 转换成可读形式的 string
    https://github.com/tikv/tikv/blob/3da2428b64fb626b6f91be7e329804c367316376/cmd/src/bin/tikv-ctl.rs#L1857-L1862
    比较好奇，这个 escape 算法是一个标准吗？
* tidb-ctl decoder 可以解析这个可读性的 string 

```sh
$ tiup ctl tikv --to-escaped 7A7480000000000000FF6E5F728000000000FF03D9230000000000FAFA2E93D5D187FED1ee
zt\200\000\000\000\000\000\000\377n_r\200\000\000\000\000\377\003\331#\000\000\000\000\000\372\372.\223\325\321\207\376\321
$ tidb-ctl  decoder  "t\200\000\000\000\000\000\000\377n_r\200\000\000\000\000\377\003\331#\000\000\000\000\000\372\372.\223\325\321\207\376\321"
```

这里有个更好的工具 https://github.com/disksing/mok