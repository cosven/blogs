---
title: "读《Operating Systems: Three Easy Pieces》"
date: 2023-05-29 14:00:00 +08:00
permalink: /blogs/19
tags: ["Operating System"]
categories: [阅读]
---

2023-05-29 更新：若想**深入**的传播一个知识或描述一件事情，按“主题”维度来进行组织，
是我目前见过的最有效的方式之一。这本书分四个主题，虚拟化(Virtualization)、
并发(Concurrency)、存储(Persistence)、安全(Security)。今日看了内存虚拟化这块，
由浅入深，引经据典，清晰易懂。好书呀！

- [-] 虚拟化（CPU）
  - [x] 2020-10-19 第4章 进程
  - [x] 2017-03-14 第7章 CPU 调度
- [-] 虚拟化（内存）
  - [x] 2023-05-29 第12~16章

## 虚拟化（CPU）
### 进程

进程就是一个运行中的程序。怎样同时运行多个程序？time sharing + context switch + scheduling。

**进程由什么组成？**看进程的机器状态（machine state）是什么:
- 内存中有什么？地址空间：进程可以寻址的内存。
- 寄存器，部分特殊的寄存器，如 program counter(PC, 也称 instruction pointer IP)；
  stack pointer，frame pointer
- 存储设备，I/O information（比如打开的文件列表）等

**程序怎样变成进程？**
1. 从 disk 到内存
2. 分配 run-time stack
3. 堆内存分配？
4. 初始化 I/O，比如 stdin/stdout/stderr

**进程状态？**Running 和 Ready 可以自由切换；Running 遇到 I/O 变成 Blocked；I/O 完成变成 Ready。

**进程相关的数据结构？**
> PCB: process control block, just a structure that contains information about a specific process.

**常见的相关问题?**
top 中显示的 D(disk sleep) 状态：https://stackoverflow.com/a/1475715/4302892。
在进行 read/write 等系统调用时，进程会进入这个状态。

- Sleeping 状态：一个进程需要等待某个资源，进程可以主动进入该状态，也可以是操作系统调度它进入该状态。
- Runnable 状态：一个进程资源都有了，但缺 CPU。
- Stop 状态：比如 zombie（为什么要有 zombie 状态？父进程需要知道子进程的状态）。

### CPU 调度

参考资料：<http://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched.pdf>

本来的目的：了解 Round Robin 算法。记忆中 `Round Robin` 也是最早出现在进程调度里面的，
于是找了本书来看下。如果之后能瞄一眼源代码的话，应该会比较好把。

这一章解决的几个问题：
- [ ] How should we develop a basic framework for thinking about scheduling policies?
  - 确认评价标准？
- [x] What are the key assumptions?
  - job 一运行就运行到底；job 运行消耗同样的时间
  - job 同时达到
  - job 运行时间已知
  - job 没有 I/O 操作
- [x] What metrics are important?
  - turn around time
    - Shortest Job First
    - Shortest Time-to-Completion First (STCF)（job 不是同时达到的）
  - responde time
    - Round Robin
- [x] What basic approaches have been used in the earliest of computer systems?
  - FIFO

Round Robin 基本思想
> The basic idea is simple: instead of running jobs to completion,
> RR runs a job for a time slice (sometimes called a scheduling quantum)
> and then switches to the next job in the run queue.

## 虚拟化（内存）
### 地址空间
它从地址空间（address space）开始讲，地址空间是对物理内存的抽象。地址空间最主要的一个用处是，
它简化了程序使用内存的方式（easy to use）。有点惊讶，从易用性出发（其实也有性能方面的考量），
竟然可以有这么伟大的发明。

内存虚拟化有几个目标：
1. 透明（transparentcy）：程序可以假设自己拥有所有内存。
2. 高效（efficiency）：时间和空间。比如 TLB 技术。
3. 安全（protection）：进程和进程之间是隔离的（isolation）。

课后作业里面提到 pmap 这个工具，感觉挺有意思的，但不知道实际有哪些应用场景。
问题了下 chatgpt，说是可以分析内存碎片、安全审计等等。

### 插曲：内存相关接口

这一章主要介绍了 malloc/free 等在对上申请/释放内存的接口，没细看。
有一个刷新我认知的知识，malloc 和 free 不是系统调用，brk,sbrk,mmap 这种才是，
brk 和 sbrk 都是给库函数用的。然后 calloc 和 realloc 这种函数也有一些使用场景。

书里面提到了几种常见的内存使用错误，挺有意思的，摘抄一下

1. 忘记申请内存了
    ```c
    char *src = "hello";
    char *dst; // oops! unallocated
    strcpy(dst, src); // segfault and die
    ```
2. 内存申请的不够（注：虽然大概率能正确运行，但是它并不正确。malloc 的长度应该要再 +1）
    ```c
    char *src = "hello";
    char *dst = (char *) malloc(strlen(src)); // too small!
    strcpy(dst, src); // work properly
    ```
3. 忘记初始化申请到的内存了
4. 忘记释放内存
5. 还没用完就把内存释放了，通常也叫做 dangling pointer。
6. 重复释放内存，通常也叫做 double free。
7. free 的参数不对，理应传入 malloc 返回的指针。

书里面还提到 purify 和 valgrind 这两个内存工具。valgrind 听到好多次了。
跟着课后习题，试玩了一下 valgrind，算是上了个手。搜了下 “valgrind rocksdb bug”，
发现确实有个 [issue][rocksdb-valgrind-issue] 说自己遇到 SIGSEGV，
然后用 valgrind 可以更方便的定位错误。从 issue 的信息中可以看到，valgrind
可以把出问题的栈完整的打印出来，理论上对问题定位应该挺有用的，不知道它性能如何，
能不能在 E2E 测试中使用。

### 地址转换

这一章主要还是介绍了地址转换这个概念。书里还介绍了 Base-and-bounds 这种最基础的方法，
来帮助读者理解地址转换，但这种方法也有很多不足指出。我想后面几章应该会讲它的一些演进。

地址转换（address translation）也是为“高效且灵活的虚拟化内存”这一目标服务的，
把地址空间和实际物理内存地址映射起来。地址转换有很多的实现办法。书中先把需求最简化，
然后讲了一种硬件方法（Dynamic (Hardware-based) Relocation），这里它讲到一个方法是
Base-and-bounds：在硬件上把一段内存的左右边界地址保存起来，
然后进程实际用的就是在这段内存内进行偏移。这里顺便引出一个概念，内存管理单元（MMU）。

> We should note that the base and bounds registers are hardware struc-
tures kept on the chip (one pair per CPU). Sometimes people call the
part of the processor that helps with address translation the memory
management unit (MMU);

这让我想起一个故事：大家在讨论进程、线程、协程的时候，经常会聊到“上下文切换”的成本高低。
当时就有个人提了个问题，你们知道“上下文切换”要切换哪些东西吗？现在算是能回答一点了。

### 分段（Segmentation）

这一章主要介绍了分段的基本思想和原理，以及它的不足。整体来说还是比较易懂的。

分段这个思想可以解决 Base-and-bounds 这个方法带来的空间浪费问题。
Base-and-bounds 这个方法带来的空间浪费问题主要侧重在 internal fragmentation。
这个思想至少可以追溯到 1960 年代初期。简单来说：就是把一个区间分成多个区间。
具体来说，书中举的例子是把 stack/heap/code 这三个部分各看成一个 segment。

书里提到 segmentation fault 这个概念，记录一下
> The term segmentation fault or violation arises from a memory access
on a segmented machine to an illegal address. Humorously, the term
persists, even on machines with no support for segmentation at all. Or
not so humorously, if you can’t figure out why your code keeps faulting.

即使有了 Segmentation 这种方法，external fragmentation 的问题仍然需要被解决（缓解）。
> The general problem that arises is that physical memory quickly becomes
full of little holes of free space, making it difficult to allocate new
segments, or to grow existing ones. We call this problem external fragmentation

这个问题在各种算法中都存在，只是说哪个更优，比如有一些策略：
> including classic algorithms like best-fit (which keeps a list of free spaces
and returns the one closest in size that satisfies the desired allocation to
the requester), worst-fit, first-fit, and more complex schemes like buddy
algorithm

### [WIP] 空闲空间管理（Free-Space Management）

这一章主要是讲内存分配这个主题，其中又侧重 free-sapce management 这个问题点,
这个问题的一个核心子问题就是外部碎片问题（external fragmentation）。

讨论是从研究用户态内存分配器入手，讨论时先忽略内部碎片这个问题。
讨论的另外一个重要假设是分配出去的内存不会被迁移到其它地方，也不会对
free-space 进行 compaction 这样的操作。还假设分配的内存是一个连续的、
固定大小的区域。

介绍了底层的一些机制（也就是内存分配频繁涉及的一些概念和策略）：
1. Splitting and Coalescing（free list 的分裂和合并）。以满足内存申请需求。
2.

[rocksdb-valgrind-issue]: https://github.com/facebook/rocksdb/issues/9066
