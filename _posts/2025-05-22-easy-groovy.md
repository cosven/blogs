---
title: Easy Groovy (1)
date: 2025-05-22 23:15:00 +08:00
permalink: /blogs/10023
tags: [easy-groovy]
categories: [life]
---

代码跳转是写项目的必备功能了，而用 VSCode 写 Groovy 竟然没有这个功能，这能忍嘛？？？

工作时而需要写 Doris 的回归测试 [regression-test]，在本地，IDEA 当然是最佳选择。
无奈我写的测试用例都是压力测试类，都需要跑在远程开发机上，而 IDEA 的远程开发又很弱智（或许是我没付钱），
于是被迫转战 VSCode。但，不能忍呀！

## 简单尝试

几年前，有人折腾过一个 [groovy-guru] 的项目，它最让我心动的就是里面包含了一个
Groovy LSP，可以实现代码跳转。半年前我就是试用过，它当时就有个毛病，动不动就卡主。
这几天又狠下心来试用了一把，果然，还是用了一会就卡主了。两个 LSP 进程 CPU 占用 100%。
于是花了点时间研究了一下。

这不很久没写博客了，水一篇。滴水穿石，非一日之功 `:)`

## 卡哪里了？

第一想法是想看看 LSP 的日志。但令人惊讶的，[Groovy Language Server] 竟然连一行日志都没有，
什么垃圾玩意。当然就在想，难道 JAVA 老师都是直接 JDB 调试么？饭中和同事聊了聊，
感觉确实有这种可能呀。

既然卡主了，也没日志，那肯定就是看看 stack 喽。自己看了一会，没看出什么名堂来。
丢给 AI 分析了一下，AI 果然瞬间抓住了重点
（其实我仔细看，也能发现，但苦于没经验呀，没经验就没信心，AI 还是有点东西的）

```
根据提供的 stack.txt 分析，高 CPU 占用的直接原因是用户线程 pool-1-thread-1 在执行 Groovy 语言服务器（net.prominic.groovyls）的 AST 节点排序操作。以下是详细分析和解决方案：

1. 定位高 CPU 线程
线程名称: pool-1-thread-1
CPU 时间: 93629636.47ms（约 9.3 万秒，占进程总运行时间 94103.90 秒的 99%+）
线程状态: RUNNABLE（正在执行代码）
关键堆栈：

at net.prominic.groovyls.compiler.ast.ASTNodeVisitor.contains(ASTNodeVisitor.java:301)
at net.prominic.groovyls.compiler.ast.ASTNodeVisitor.lambda$getNodeAtLineAndColumn$1(...)
at java.util.TimSort.binarySort(TimSort.java:296)  // TimSort 排序操作
```

打了几次 stack，发现都是类似的，基本可以断定，要么是死循环了。
要么是我这个回归测试的项目太大了，它分析太耗时。

## 上 JDB 查查？

查了查，上 JDB 的前提是，java 进程已经开启了 debug 模式。也就是启动的时候要加上这个参数：
> `-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:0,quiet=y`

加这个参数的过程中也遇到了一些小坑，我之前是指定了调试端口，但由于 VSCode 疑似会启动多个
（我测试遇到的情况是两个）LSP server，于是总有一个 server 会由于端口冲突而失败。

加了调试参数之后，确实就能使用 JDB 了，但 JDB 的操作学起来也挺麻烦了。听说 JAVA
老师都用 IDEA，试了一下确实方便... 切换线程、查看变量、设置断点，太方便了。
以前写 Python，都是 print 为主，pdb 用的很少，反思一下，还是 Python 调试器没这个好用。
经过多次断点，发现问题出在了它的 getParent 方法上，一个 child 的 parent 还是它自己，
于是它和 contains 配合，直接死循环了。

<https://github.com/GroovyLanguageServer/groovy-language-server/blob/cff39b87ffb3eb0dfc483ba727914824e871247e/src/main/java/net/prominic/groovyls/compiler/ast/ASTNodeVisitor.java#L244C17-L264>
```
	public ASTNode getParent(ASTNode child) {
		if (child == null) {
			return null;
		}
		ASTNodeLookupData data = lookup.get(new ASTLookupKey(child));
		if (data == null) {
			return null;
		}
		return data.parent;
	}

	public boolean contains(ASTNode ancestor, ASTNode descendant) {
		ASTNode current = getParent(descendant);
		while (current != null) {
			if (current.equals(ancestor)) {
				return true;
			}
			current = getParent(current);
		}
		return false;
	}
```

## 为啥有死循环嘞？

这里就比较琐碎了。

1. 先分析 node.parent 是在哪里设置的？---> 看了一波代码，发现这个逻辑还是挺精简的，
   是在 ASTNodeVisitor 的 pushASTNode 方法里设置的。
2. 那这个方法又是在哪里调用的呢？---> 继续看代码，发现 ASTNodeVisitor 的 visit 方法里，
3. 之前分析已经发现，有问题的 ASTNode 是一个 ElvisOperatorExpression，读读它的代码，
   发现它的 visit 实现确实有问题。

看看这个项目的代码，感觉某种程度来说，也挺草台的（有的函数就直接就注释掉，没有任何注释）。

好，问题找到了，愉快的水了一个 PR： <https://github.com/GroovyLanguageServer/groovy-language-server/pull/102>

## LSP 修好还不够，还得更新 VSCode 插件

Groovy-guru 这个项目用的 LSP 是 Groovy Language Server，它通过 submodule
来集成的。考虑到 Groovy-guru 这个项目已经两年没更新了，于是我 Fork 了一个，
为了能发布插件，还把项目名字啥的都改了一下，在 VSCode 插件市场注册了一个账号，
发布了 [Easy Groovy v0.1.1]。

## 结语

这下我也是 VSCode 插件开发者了 `:)`

LSP、tree-sitter 这些技术还是挺有意思的，之前一直想研究研究来着。
这次算是个不错的机会，总体比较顺利，踩坑也不多。

下一步计划是增强一下 Groovy LSP 在 Doris 回归测试项目中的性能（我发现它会加载所有的 suite 文件）；
或者研究研究，看看能不能让它跳转到 [Plugins] 的代码里去。目标是对编辑器有更深了解。

好，成功的划水一篇！

[groovy-guru]: https://github.com/DontShaveTheYak/groovy-guru
[regression-test]: https://github.com/apache/doris/tree/master/regression-test
[Groovy Language Server]: https://github.com/GroovyLanguageServer/groovy-language-server
[Easy Groovy v0.1.1]: https://marketplace.visualstudio.com/items?itemName=cosven.easy-groovy
[Plugin]: https://github.com/apache/doris/tree/master/regression-test/plugins
