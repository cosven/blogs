---
title: AI benchmark 长啥样？有参考价值么？（1）
date: 2025-12-04 00:04:00 +08:00
permalink: /blogs/10028
tags: [AI, benchmark]
categories: [稍微正经点的]
---

**TL;DR** benchmark 有价值，榜单价值偏小。（IMO：给效率算分，榜单就能恢复它的价值。）

不知道你们有没有这些疑问哈
1. 大模型发布时，大都会刷榜（跑 benchmark），这些 benchmark 的参考价值怎么样？里面都有些什么样的测试题？能反应模型能力强弱么？
2. 光有模型不够呀，大模型自己又不会解决问题，总得有东西（现在咱都叫 agent）来调用大模型吧，厂商评测的时候，用的是啥 agent 呢？不同厂商用的 agent 相同么？

[DeepSeek-V3.2](https://mp.weixin.qq.com/s/ohsU1xRrYu9xcVD7qu5lNw) [这个表格](https://mmbiz.qpic.cn/mmbiz_png/pkWC42uvwkmkWUpqHRFeNvCZJ4dbmZuoVzdEgO2VgzAMhiabCbR6Nrbuvb4KIdlmmxYy8xVkJm6m1YpicaQRhpNw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=0)
显示它重点关注了 8 个基准测试集。其中 4 个评估推理能力的，分别是 AIME（美国数学邀请赛）, HMMT（哈佛 MIT 数学竞赛）,
HLE（人类全学科前沿难题测试）, CodeForces（世界级编程竞赛）。还有 4 个评估 Agentic 能力的，
分别是 SWE Verified, Terminal Bench 2.0，𝜏2-Bench，Tool Decathlon。前两个和程序员关系还是紧密的，
我重点研究下这两个 benchmark。

## SWE-bench Verified

下面是 AI 关于这个测试集的介绍，还是非常准确的。

> SWE-bench（Software Engineering Benchmark）是一个专门用于评估大语言模型（LLM）在真实软件工程任务中表现的基准测试，
> 最初由普林斯顿大学 NLP 团队于 2023 年发布。 它的核心任务是：给定一个 GitHub 开源项目的代码仓库和一个对应的
> issue（问题描述），模型需要像人类开发者一样，理解问题、定位代码、修改文件，并生成一个补丁（patch），
> 最终通过该项目原有的单元测试验证修复是否成功。
> SWE-bench 的关键特点：
> - 真实场景：所有任务都来源于 GitHub 上 12 个流行 Python 项目的真实 issue 和对应的 pull request（PR），共 2,294 个任务实例。
> - 执行验证：采用“Fail-to-Pass”测试机制，即修复前测试失败，修复后测试通过，才算成功。
> - 高难度：任务涉及多文件理解、跨模块调用、复杂依赖关系等，远超传统代码补全或算法题。

### 其中一个题目

看一个真实的题，[链接是这个](https://github.com/pylint-dev/pylint/issues/6471)，是一个真实的 GitHub Issue，
内容我就不贴过来了，请点击链接阅读。pylint 项目算是 Python 社区的一个中型项目，阅读 issue 可以发现，
这个题本身是真实存在的且有现实意义的，有一定的难度。并且在 issue 的描述中，没有透露任何解法。
总结来说：**题目是有参考价值的**，这就是普通的人类程序员遇到的问题。

### 大模型怎样解题

大模型要解 SWE Verified 基准测试集里面的题目，就必须要修改，编辑代码文件。也就是说，**必须得找一个 agent
来调用大模型**，这样才能解题。那看看 DeepSeek 的打榜分数是用啥 agent 测出来的吧？

读一下 DeepSeek 论文，有一段文字解答了这个疑惑。答案是：使用内部框架去测的。

> For SWE-bench Verified, the primary score was obtained using our internal framework.
> Robustness tests across other settings—including the Claude Code and RooCode frameworks,
> as well as non-thinking mode—produced consistent results, ranging from 72 to 74.

有点可惜哈，看不到 DeepSeek 的 agent 代码和评估流程。不过，从这一小段文字，
**这下你知道该用哪个 agent 来编码了吧 [阴险笑]**。待我回头去试试 RooCode！

虽然看不到 DeepSeek 的评估流程，但找找资料可以发现，这个榜单的作者，他们已经提供了多个工具来帮助评测，
其中一个工具（agent）是 [mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent)。
这个 agent 核心代码只有 100，它想法非常大胆，我感觉非常有意思。我尝试概括一下它的核心思想。

> 用户从提出问题到看到最终的解决方案，只需要做两件事：一是在启动后把问题描述输入给它；
> 二是持续按 enter 键，直到模型自己说“我已经解决完毕了”。
> 这个 agent 让大模型“用且只用 bash 命令”来解决用户提出的编码问题。
> 模型在接受到用户的问题之后，会尝试自己去生成 bash 命令来获取上下文，或者编辑文件。

你可能会好奇，解决上面这个 pylint 问题，那肯定需要修改代码呀。大模型光用 bash 命令，也能修改代码？
答案是yes，现在的大模型（我用 DeepSeek-v3.2 测试了一下）真的会用 bash 命令来编辑文件。举个例子，
在我测试的过程中，大模型会生成一条这样的 bash 命令，来编辑 `pylinter.py` 文件
```bash
cd /Users/cosven/code/pylint/pylint/lint && python3 << 'EOF'
import os
...
...
content = xxx
...
# Write the modified file
with open('pylinter.py', 'w') as f:
    f.write(content)

print("Successfully modified pylinter.py")
EOF
```

除了这个“剑走偏锋”的 agent 外，还有很多其它 agent 也能辅助做类似的评测，我之前也[手动测过几个](https://cosven.me/blogs/10024)，
感兴趣的话，你可以看看。

## 一些思考

虽然目前只看了一个 benchmark，但对之前的两个问题，已经有了初步答案：参考价值还是有的，因为 benchmark 的测试题都是真实存在的。
但，这个 benchmark 的评测指标还是太局限了。举个例子，对于 SWE-bench Verified，各厂商只展现了一个指标，那就是 Resolved 率（也就是问题解决率）。
而人用大模型、或者 agent 来解决实际问题，**不仅会关注最终的“解决率”，还有一个重要的指标是“效率”**。我相信，
以后大模型和 agent 的 benchmark，一定会把效率考虑进去。

我用 mini-swe-agent + DeepSeek-V3.2 来尝试解决上面那个 pylint issue。硬是和大模型来回对话了 44 轮，
它才开始尝试修改代码来修复问题。而这时候，它已经花了我好几分钟了，同时花了我 3 毛钱（心疼啊【狗头】）。
我要正常用的话，别说 44 轮，第 2 轮它表现不符我意，我就不会继续和它对话了。

所以说，benchmark 是有用的，但榜单是没用的。好的，得到一个自己满意的答案，睡觉！

再想一下，这么“傻”的 agent 都能解决问题，且能让解决率达到 74.4%（DeepSeek-V3.2 测出来 73.1%）。
现在的大模型真是强的有点可怕，潜力很大。同时，agent 怎样发挥模型的本领，要/能做的事情还很多！
