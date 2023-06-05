---
title: "读《Operating Systems: Three Easy Pieces》"
date: 2023-05-29 14:00:00 +08:00
permalink: /blogs/19
tags: ["Operating System"]
categories: [阅读]
---

2023-05-29：若想**深入**的传播一个知识或描述一件事情，按“主题”维度来进行组织，
是我目前见过的最有效的方式之一。这本书分四个主题，虚拟化(Virtualization)、
并发(Concurrency)、存储(Persistence)、安全(Security)。今日看了内存虚拟化这块，
由浅入深，引经据典，清晰易懂。好书！这本书另外一个非常赞的地方则是，
它在每一章开始部分都会提出一个问题来引导读者，问题是最好的老师！

- [-] CPU虚拟化
  - [x] 2020-10-19 第4章 进程
  - [x] 2017-03-14 第7章 CPU 调度
- [x] 内存虚拟化(第12-24章)
- [ ] 并发
- [-] 持久化
  - [x] 2023-06-06 第36章 I/O 设备

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

### 空闲空间管理（Free-Space Management）

这一章主要是讲内存分配这个主题，其中又侧重 free-sapce management 这个问题点,
这个问题的一个核心子问题就是外部碎片问题（external fragmentation）。
讲了内存分配需要考虑哪几个基本问题，然后又介绍了几种简单的策略帮助读者理解这个问题，
然后又介绍了几种实践中用到的策略。最后总结，主要从效率和空间浪费两个角度来考虑这个问题，
不同的负载下，有不同的最佳策略，是一个利弊权衡的过程。

讨论是从研究用户态内存分配器入手，讨论时先忽略内部碎片这个问题。
讨论的另外一个重要假设是分配出去的内存不会被迁移到其它地方，也不会对
free-space 进行 compaction 这样的操作。还假设分配的内存是一个连续的、
固定大小的区域。

介绍了底层的一些机制（也就是内存分配频繁涉及的一些问题和策略）：
1. Splitting and Coalescing（free list 的分裂和合并）。以满足内存申请需求。
2. Tracking The Size Of Allocated Regions。free 接口接受的参数是一个指针，
   allocator 需要知道指向的内存块的大小。一个常见的实现办法是划分一个 header
   来存储这个大小的信息。(RocksDB 就是依赖这种办法来统计 block 大小的。)
3. Embedding A Free List。大意是这个“空闲空间管理”的数据结构本身也要占内存，
   怎样存储它是一个常见问题：而常见方式是在头部存链表的 head 节点。
4. Growing The Heap。一种常见实现是应用调用 sbrk 系统调用，
   操作系统会把一些空闲的 page 分配给它（物理上不一定是连续的，进程看到的是连续的）。

空闲空间的几种常见管理策略
1. best fit：空间浪费少，但搜索效率低。
2. worst fit: ？找一个空间最大的块，仍然是全局搜索。
3. first fit: 。
4. next fit：每次搜索完，记录指针所在位置，下次搜索是上次的位置继续。

书中说这几种策略只是非常初级的策略，实际会更复杂。这几种只是让读者有个印象。
后面又列了一些实际应用的一些（优化）策略：
1. Segregated Lists。大概意思是把空闲空间分几个链表来管理，
   每个链表管理的空闲空间大小都是一样的，比如都是 8Byte/16Byte。
   这样在效率和空间两方面都不错。书中举了 `slab allocator` 这个例子。
2. Buddy Allocator。考虑到 Coalescing 这个操作对分配器来说是非常重要的，
   举的例子是 `binary buddy allocator`。
3. 其它：使用一些平衡二叉树等数据结构来保存空闲空间，提升搜索效率。

### Paging: Introduction

这一章我只是粗略的看了下，它主要介绍了 paging 这个思想，操作系统是如何实现这种策略的。
操作系统把物理内存分成大小相同的页（**page**），用户态的分配器来申请内存的时候，
也是申请一页或者多页。每个进程会有一个 **page table** 来保存虚拟地址和物理地址的映射。
page table 这个东西设计时又需要考虑几个因素：效率和空间占用。后面两章会讲一个高效的
paging 实现是怎样的。

这里面说了几个概念
1. Instead of splitting up a process’s address space into some number of
   variable-sized logical segments (e.g., code, heap, stack), we divide
   it into fixed-sized units, each of which we call a **page**.
2. we view physical memory as an array of fixed-sized slots called
   **page frames**, each of these frames can contain a single
   virtual-memory page.
2. physical frame number (PFN)
2. physical page number or PPN
3. Page Table Entry (PTE): page table 由众多 entry 组成，一个 entry
   里面不仅会由虚实的映射，还有有一些 flag,比如这个 page 是否脏了，读写权限等。

### Paging: Faster Translations (TLBs)

这一章看的也比较粗略，它主要介绍了 TLB 的作用和具体实现。也简介了 TLB 的不足。
后续有需求的话，其实可以再细致的看一下。

注：这个问题也是从两个角度出发：OS 角度和硬件角度。上面提到的很多问题都是从这两个角度来思考。

TLB 算法简述：算法的输入是虚拟地址，输出是物理内存地址。TLB 接收到请求时，
如果发现虚拟地址对应的 TlbEntry 在缓存里，直接返回
（返回前还会判断一下这个 TlbEntry 的一些状态，比如 ProtectBits 的值）。
如果不在缓存，算一下，然后加载到缓存，然后走类似的返回逻辑。
传统的 x86 架构的 TLB 是一个硬件单元，一些现代的新架构，如 RISC
可以在软件层面来管理 TLB。

这种缓存在很多场景都非常有用，比如遍历一个数组。原因在于其空间局部性（spatial locality）。

重要概念
1. TLB: translation-lookaside buffer。这个名字有一些历史原因，叫做
   address-translation cache 更形象。它也是 MMU 的一部分。
2. TLB hit/miss：缓存就会有 hit 和 miss，很合理。
3. locality：There are usually two types of locality: temporal locality
   and spatial locality. With temporal locality, the idea is that an
   instruction or data item that has been recently accessed will likely
   be re-accessed soon in the future.

### Paging: Smaller Tables

粗略的读这一章，这一章主要介绍了几种 page table 的优化方案（对于特定场景来说）
1. Bigger Pages：这种思想的一个问题在于会加重 internal fragmentation 问题。
2. Hybrid Approach: Paging and Segments。Hybird 的方法在均衡两者优劣的背景下，
   通常还会给逻辑处理引入复杂度。
3. Multi-level Page Tables： 想象一下，把 page 分几个字文件夹（page directory）。
4. Inverted Page Tables: 只存一个 page table, 然后在这个 page table
   上记录一个 page 被哪些进程使用。
5. Swapping the Page Tables to Disk。


### 跳过剩下的3个章节

越后面的章节，内容越细，也就是说，用到的概率越小了。我自己暂时还没有产生这方面的疑问，
阅读的效率可能比较低，遂先跳过。但不得不说的是，计算机很多问题的本质都是相似的，
比如缓存；空间换时间；两种方法混合（均衡优劣）；分段和分页思想。我想，
以后应该还会遇到这种问题，到时候再来看应该能有新的收获。

1. Swapping：Mechanisms
2. Swapping：Polices
3. Complete VM Systems：介绍了 Linux 的 VM system 是怎么实现的，先跳过，嘻嘻。

[rocksdb-valgrind-issue]: https://github.com/facebook/rocksdb/issues/9066

## 持久化（Persistence）

### I/O 设备

这一章回答的关键问题是：怎样把 I/O 集成到系统中。

从系统架构角度看：书中提到一个点，高性能的 I/O 设备通过一个通用的 **I/O bus** 来连接，
在现代系统中通常对应 PCI。低性能的设备通过 **peripheral bus** 来连接，比如 SCSI, SATA
或 USB。注：这让我联想到 PCIE SSD。再次搜了下，PCI-E 和 SATA 一个典型的区别在于 SATA
最高带宽是 6Gbps（第三代 SATA，即 SATA3）。另有资料表示 PCI-E 的带宽对现代部分显卡已经不够用了。
书中给的两张系统架构图（以前和现代）非常直观。

从设备角度看：设备通常有两个重要组成部分，一个是设备对系统提供的硬件接口；
另外一个是它的内部结构（比如现代的 RAID 控制器由一个固件来实现它的功能）。

从设备和系统的（通信/交互）协议来看：书中举了一个例子，一个设备提供的接口由三个
Registers 组成，status, command, data。然后计算机可以通过 polling
的方式来处理通信过程。但 polling 在部分场景会有性能不够好的问题，接着进入下一章：
通过中断（Interrupts）来减少 CPU 开销。polling 的方法也叫做 PIO（Programmed I/O）。
当设备速度足够快的时候，中断可能会降低性能。

当使用 PIO 这种方法来传输大量数据时，CPU 会花很多时间在内存和设备之间拷贝数据。
DMA（Direct Memory Access） 这种办法可以用来解决这一问题。

设备驱动（device driver）：让设备可以与操作系统以最优的方式进行数据传输。
这里有一个案例学习，一个简单的 IDE Disk Driver。
