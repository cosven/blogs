---
title: 读《Attention is all you need》
date: 2026-06-12 00:00:00 +08:00
permalink: /blogs/10033
tags: [AI, benchmark, evaluation]
categories: [稍微正经点的]
---

第一个视频：[直观解释 transformer](https://bilibili.com/video/BV13z421U7cs)
第二个视频：[直观解释注意力机制](https://www.bilibili.com/video/BV1TZ421j7Ke)

论文读后感（看视频+和AI聊天）
1. GPT 解决的核心问题，或者说 Transformer 这种架构解决的核心问题：让上下文建联变得非常高效。
   以前的算法也能建联，但局限性比较多，比如长距离依赖难、并行效率差、扩展到大规模数据难。
2. 怎么解决的？靠 Attention。
3. 与 Transformer 相同级的概念是 RNN、LSTM、CNN 等概念。
   - 在 RNN 中，按顺序传递信息，缺点是不能并行，长距离依赖难。
   - LSTM，带门控的 RNN，比普通 RNN 更能记长上下文。仍然按顺序计算，扩展性差。
   - CNN，局部卷积，每层看固定窗口，堆多层扩大范围，可并行，局部模式强，长距离关系会膨胀。
   - Transformer 是用 self-attention，每个 token 可以直接关注其它的 token。长距离关系强，训练并行，规模化好。
4. Attention 是什么东西，核心有几个概念
   - token, embedding, query, key, value
   - `Attention(q,k,v) = softmax(QK^T / sqrt(d_k))*V`，这个公式可以理解为，它在计算 token 的注意力。
   - 常说的 KV cache 指的就是 token 的 K 和 V 向量。每一层 Attention 计算出来的 KV 都需要缓存。
   - 所以怎么理解“token的注意力”这个概念，我感觉就是一个 token 和其它 token 的关系。
     AI 说基本对，准确一点这么说：当前 token 在更新自己的表示时，应该从其它 token 读取多少信息。
5. 模型怎样产生下一个 token？当每个token的向量表示都经过上下文洗礼之后，模型会有一层（应该是叫做 LM head），
   它会取最后一个 token 的向量（简单这么理解应该没问题），和一个固定的训练好的矩阵相乘，经过 softmax 处理后，
   得到一个 token 概念排序，于是就知道下个 token 是啥了。
