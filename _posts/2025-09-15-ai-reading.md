---
title: AI 笔记
date: 2026-03-09 01:50:00 +08:00
permalink: /blogs/10029
tags: [笔记, AI]
categories: [阅读]
---

## 读《Evaluating Memory in LLM Agents via Incremental Multi-Turn Interactions》(03-10)

和 AI 一起读了一下这篇论文，也把它的 benchmark 手动跑了一下。个人感觉这篇论文有两个非常有学习价值的点

1. 提出核心的评价维度是4个。
    > In this paper, based on classic theories from memory science and cognitive science, we identify four core competencies essential for memory agents: accurate retrieval, test-time learning, long-range understanding, and selective forgetting.

2. （AI 总结的）两个数据集。这篇论文为了填补现有基准在评估“准确检索”和“选择性遗忘”上的空白，
    专门构建了两个全新的数据集：EventQA 和 FactConsolidation。此外，作者强调，虽然现有很多长文本数据集，
    但它们大多是一次性输入给模型的。为了评测记忆代理，作者还将许多现有的长文本数据集（如 Document QA, LongMemEval 等）
    进行了重构，把它们切分成对话块（chunks），要求代理随时间推移增量式地读取和记忆。

看一个 accurate retrieval 的 benchmark 题示例（对这个 benchmark 有个基本的感受）：
```
Answer the question based on the memorized documents. Only give me the answer and do not output any other words.

 Now Answer the Question: In what country is Normandy located?
```

看一个 selective forgetting (multiple-hop) 的 benchmark 题示例（对这个 benchmark 有个基本的感受）：
```
Pretend you are a knowledge management system. Each fact in the knowledge pool is provided with a serial number at the beginning, and the newer fact has larger serial number.
 You need to solve the conflicts of facts in the knowledge pool by finding the newest fact with larger serial number. You need to answer a question based on this rule. You should give a very concise answer without saying other words for the question **only** from the knowledge pool you have memorized rather than the real facts in real world.

For example:

 [Knowledge Pool]

 Question: Based on the provided Knowledge Pool, what is the name of the current president of Russia?
Answer: Donald Trump

 Now Answer the Question: Based on the provided Knowledge Pool, What is the country of citizenship of the spouse of the author of Our Mutual Friend?
Answer:
 ```

读后感：这几天看了两三个 agent 评测相关的博客或论文，总结下来有个共同的感受，找到核心的评测维度非常关键。
比如记忆系统，作者这里提炼了四个评测维度，然后每个维度设计评测数据集。再比如 03-06 读的那篇文章，
它的 ChatBI 评测也提炼了一些维度。之前还看到说 agent 评测，一个维度是评估它调用工具的能力。

## AI 这么强，我还能赚钱么？(03-09)

想想，我是否还能通过数据库测试这个工具来赚钱？ -> 结论还是可以的，但思路和人数有大变化
1. 测试基建
    1. 【硬需求：懂测试基建的架构师】 基建选型：需要人工的调研和决策，划分系统边界。
    2. 【实际需要】对于每个系统来说：需要指定系统的设计和验收方案。
2. 测试设计
    1. 【硬需求：懂测试的架构师】测试设计：需要人工的设计。划分系统边界。
    2. 【硬需求】场景测试：产品需求变成场景
    3. 【实际需要】各类系统测试：需要有人结合架构师的设计，以及动手能力来落地测试。
    4. 一个变化是 feature 测试将来一定是研发自己负责闭环。

## 读《2025，我们这样评测 AI》笔记（2026-03-06）

看这篇文章的初衷是我在本地也构建一些测试 agent。但怎么把这些 agent 分享给其他人，
怎样让其他人知道这些 agent 是真的能够解决一些实际问题呢。我觉得这就涉及 agent 的测试/评测。

文章链接：https://testerhome.com/topics/43475

有几个例子有利于理解 Agent 测试，一个是 ChatBI。
文中提到，AI 评测痛点：
- 评测数据集的构建：真实性，权威性，（以及全面性）
- 有效断言：开发型问题的判定；标准答案的语义理解
- 数据集规模（效率）

评测举例，ChatBI：
1. 评测预料生成：从业务理解角度拆出 “原子指标”，“加工逻辑”，和“分析维度”三个维度，从这三个维度确保全面性。
2. 评测指标：以查询结果是否正确为核心。查询结果是否正确的判断才用人工先给出一个标准答案。
3. 评测工具：自动生成评测语料。
4. 难点：用户问法多样性（比如复合问法）；断言效率与门槛。

RAG 评测：
看下来，我的感受是作者没有讲清楚。
1. 全集是啥：提到了单篇、多篇与标题。末尾作者也提到，这个还不够。
2. 评测指标：没有拆成可以量化的指标。
3. 局部最优的陷阱

自己想想这个问题。如果把自己当成一个 RAG 系统，当一个问题来的时候
1. 首先，需要判断这个问题是否能回答。
    1. 开放型问题，按理说这类问题不是 RAG 系统能解决的。
    2. 确定性问题，则可以回答知道或不知道。然后再有答案的准确性。
       1. 推理性，统计性问题
       2. 检索性问题
感觉从这个维度去拆分全集会好一些？算了，放弃看这个。

Agentic 智能体评测：感谢写的没有特别清楚。

## happy-llm 学习笔记（2026-03-05）

看了它的前两章，将 Transfermer 的架构，和老的神经网络架构做了对比。说它主要解决了两个问题，
一个是长距离依赖问题，一个是并行计算问题。接着又说了 LLM 另外一个核心概念，就是 Attention。
它的核心思想是：每个 token 都可以和输入序列中的任意一个 token 进行交互。算法没看太明白。
先放下来，等后续有机会再看。

## AI 代替测试工程师么？（2026-02-28）

（数据库）测试工程师的工作内容：
- 测试
    - 基础测试集和特性测试
    - 测试设计，测试实现，测试执行，测试结果分析
    - 测试基建
- Bug review
- 质量度量
- 胶水工作：发版；流水线维护等

最关键还是跑通这个流程：测试设计、编写、运行、沉淀用例。

个人感觉，AI 与人最大的区别在于它无法很好的沉淀记忆。而目前还没有很好的记忆组件，
听说 claude-mem 也有性能太差的问题。

按照我目前的调研，没有一个真正能工作的很好的 “长期记忆” 系统。基于 markdown 记忆的有 openclaw,
memsearch 等记忆系统。还有 claude 最新的记忆系统，也是基于 markdown 的。这一类奉行的是
Markdown is the source of truth。还有一种基于向量库的。supermemory, memori 等 SAAS 服务，
这种不知道真实效果如何，看资料效果好一点。

## AI Agent 开发 (2025-09-09)

https://www.parlant.io/blog/how-parlant-guarantees-compliance/

这篇文章说了 Agent 开发的几个痛点问题，并概要描述 parlant 是怎样解决的。

文章提了两个衡量 Agent 效果的指标：失败频率；失败的严重程度。它还说了5种具体的失败类型。
它指出传统 agent 的核心问题的根源来自：[Curse of Instructions](https://openreview.net/forum?id=R6q67CDBCH) 。
ps：它将‘依靠一段 prompt 来约束系统’的 agent 叫做传统。
这篇论文提到了一个研究观点：大模型遵守一两条规则是比较有效的，
当规则变多到10来条的时候，它遵守的准确定会大幅降低。

parlant 通过一套系统，让模型每次只需要遵守少数几条规则，来解决上述问题。
它的理论基础是 [Attentive Reasoning Queries (ARQs)](https://arxiv.org/abs/2503.03669) 这篇论文。
另外，它也说自己这套规则（agentic rule）和在代码里面直接编写规则（scripted rule）还是有区别的。

读后感：这篇文章让我想起了 Gemini Cli 的 prompt，它定义了几种任务场景，然后为每种任务场景定义了一个工作流。
它的目的也是想让大模型按照某个思路来做事。我主观觉得 Gemini Cli 的局限性是比较明显的，
它 prompt 提到的规则势必会很泛。
