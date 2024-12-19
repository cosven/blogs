---
title: 记点笔记（2024-10）
date: 2024-10-22 11:05:00 +08:00
permalink: /blogs/10022
tags: [笔记]
categories: [阅读]
---

强行记一点，养成好习惯！

## AI 编辑器

刚接触 AI 编辑器是在 10 月份，当时这对我来说还是个新奇的话题。
现在 12 月，两个月过去了，周边同事或朋友几乎都用上了 cursor 或 windsurf。
是个程序员可能都尝试或听说过了。有人喜欢它的 chat 功能，有人喜欢它的 tab 功能。

[这篇文章][cursor-review] 对cursor 的介绍的不错。

## 并发模型

今天在看什么资料的时候，看到并发模型的概念，这是我一直都很好奇的东西。我不需要掌握使用它，
但需要了解它大概的样子，知道它的基本使用场景。

看了一些资料后，我对“并发模型”的基本理解
1. 常听说的并发模型有：线程与锁；函数式编程；actor 模型；CSP。
2. （我理解）不同的并发模型主要考虑两方面的东西：代码可维护性；性能。
   不同的并发模型，使用的技术仍然是队列、线程池、协程这些老东西。

Actor 模型的基本印象
1. 这个模型有几个概念：信箱（mailbox）；actor。
   1. 信箱对应到真正的实现，一般也就是一个队列。
   2. actor 通常会无限循环，从 mailbox 接受并处理消息。
2. Actor 模型通过异步消息来通信，发送者和接受者之间不共享状态。
2. 发送消息这个操作本身非阻塞。消息并不是直接发送到一个 actor，而是发送到一个信箱。

CSP 模型的基本印象
1. CSP 模型不关注发送消息的实体，而是关注发送消息时使用的 channel (通道)。
   channel 是第一类 对象，它不像进程那样与信箱是紧轉合的，而是可以单独创建和读写，
   并在进程之间传递。

CSP 模型与 Actor 模型的差别（来自 AI 的解释）
1. 通信方式不一样：CSP 通过**共享的 channel**来通信。而 Actor 模型通过传递消息。
2. 并发实体不一样：CSP 模型并发实体是进程。而 Actor 模型的并发实体是 actor。

其它问题
1. Actor 模型与“生产者/消费者”模型的关系？它们本身就不同一个抽象层级的概念。
2. 数据库的 pipeline 模型是什么东西？和 CPU 的 pipeline 相似。
3. “线程与锁”，“Actor 模型”，“CSP 模型” 这三个的异同？看一下七周七并发每个章节的复习章节。

## 协程与事件循环

协程需要一个循环来驱动它们的状态变化。

比如对于 Python 的 asyncio 来说，在 Linux 环境下，它会使用 epoll 来驱动。
那对于 goroutine 或者 rust 的协程，它们使用什么来驱动的呢？花了点时间研究了下，
看起来它们会自己来实现。简单理解的话，它们会实现一个消息队列，然后轮询这个队列。
下面两篇文章（尤其是第二篇）比较详细的描述了这个“驱动”，它称之为 scheduler。

- https://kerkour.com/rust-async-await-what-is-a-runtime
- https://tokio.rs/blog/2019-10-scheduler#the-next-generation-tokio-scheduler

## Doris MOW 实现原理

参考资料1：https://zhuanlan.zhihu.com/p/590643211

2024-12-19：最近有种感觉是没有抓住这个话题的重点，这个东西可能应该鸽了。

关键概念或问题
1. 主键索引
2. delete bitmap
3. todo: 所有的“主键行号”到底是怎么计算的？

在 Doris，Delete bitmap 在代码层对应了 “Roaring BitMap” 这个数据结构。这里涉及几个概念：Bitmap（位图），
Roaring Bitmap（压缩位图）。Bitmap 实现了数字和 bit 一对一。Roaring Bitmap 是一种优化，
处理稀疏数据时的性能更好，尤其是空间性能。

[cursor-review]: https://www.arguingwithalgorithms.com/posts/cursor-review.html
