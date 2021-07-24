---
title: How go test works?
date: 2021-07-24 09:04:00 +08:00
permalink: /blogs/10015
tags: [golang, testing]
categories: [稍微正经点的]
---

Golang 生态许多测试“框架”都是基于 testing 标准库来构建的。比如 testify，ginkgo 等。
不管使用哪个框架来编写测试用例，我们都可以通过 `go test` 命令来触发用例的执行，
看代码先看 main，于是就想看看 go test 内部的实现机制如何。

## 带着问题思考

1. 为什么可以通过运行 `go test ./...` 来执行所有包的测试用例？
   go 不是一门需要编译的语言么，这里面似乎没有编译的过程？
2. 基于 golang 实现的 TiDB 在批量执行测试用例时，提出一个[需求][issue-26289]：
   控制单个测试用例的执行时间。
3. 为啥生态里面的测试框架都围绕 testing 来进行？不另辟蹊径？

脑子里面其实也有另外一些问题，比如：go test 怎样实现用例的并发运行的？
不过这些不是本次思考的重点。

## 代码阅读

下次重读起来方便一点。

### 入口

找到 go test main 的入口 [golang v1.16.6 go test 的入口函数][go-test-main]。
这个 test 包里面 6 个文件，test.go 包含了测试运行的主要逻辑，`runTest` 函数为入口。
其它文件扫了下内容，无关紧要。
```
test
- cover.go
- flagsdefs.go
- flagsdef_test.go
- genflags.go
- test.go
- testflag.go
```

吐槽1：找到入口之后，想看下这个 test package 以及里面的文件大概都干了啥事情，
但比较蛋疼的是代码里面并没有这样的注释，也没有类似 `doc.go` 文档，
也没有类似 Python docstring 的东西。于是只能人肉快速遍历一遍代码。

### 关键逻辑概览

`runTest` [L579][L579] 下面几行看起来就比较重要。
```golang
579 func runTest(...)

599     work.FindExecCmd() // initialize cached result
        ...
        ...
600     work.BuildInit()
601	    work.VetFlags = testVet.flags
602	    work.VetExplicit = testVet.explicit
603
605	    pkgOpts := load.PackageOpts{ModResolveTests: true}
606	    pkgs = load.PackagesAndErrors(ctx, pkgOpts, pkgArgs)
607	    load.CheckPackageErrors(pkgs)
        ...
        ...
648     var b work.Builder
649     b.Init()
        ...
        ...
778     // Prepare build + run + print actions for all packages being tested.
779     for _, p := range pkgs {
        ...
        ...
800     }
        ...
        ...
802     // Ultimately the goal is to print the output.
803     root := &work.Action{Mode: "go test", Func: printExitStatus, Deps: prints}
        ...
        ...
826     b.Do(ctx, root)
```

吐槽2：但是 `work` 是啥意思，这名字就很怪，又没有注释。我也不想每行都自己去读，
于是我想起贵司有人以前搞了 go 夜读。我想这东西应该已经有人解释过了把。嘿嘿。
果然：[第 20 期 2018-11-15 go test 及测试覆盖率](https://talkgo.org/t/topic/39)。
不过视频看下来要 1 个半小时，快速点了几个帧，和我想了解的内容不太一样。

使用 GitHub 的代码跳转功能，点进上面几个关键函数看了下它们的作用。发现这块逻辑本身还是挺清晰的。
有几个关键的结构体。

work.Action：go build 是一个 action，go test 也是一个 action，test 完成之后，
print 测试报告也是一个 action。action 可以依赖若干其它 actions。保存在 Deps 属性中。
如下的 prints action 会包含每个 package 测试的输出任务，每个 print 任务会依赖一个 run action，
run action 会依赖 build action。
```golang
root := &work.Action{Mode: "go test", Func: printExitStatus, Deps: prints}
```

b.Do 会触发这些 actions 的运行，没啥奇怪的地方。
```
b.Do(ctx, root)
```

### build/run action 内部实现

各种代码跳转后，发现有一块非常“有意思”的代码。它的逻辑是为待测试的 package 生成一个用于执行测试的 main。

[test main template](https://github.com/golang/go/blob/052da5717e02659da49707873b3868fe36f2aaf0/src/cmd/go/internal/load/test.go#L681-L786)

该 main 的主要逻辑如下
```
func main() {
{{if .Cover}}
	testing.RegisterCover(testing.Cover{
		Mode: {{printf "%q" .Cover.Mode}},
		Counters: coverCounters,
		Blocks: coverBlocks,
		CoveredPackages: {{printf "%q" .Covered}},
	})
{{end}}
	m := testing.MainStart(testdeps.TestDeps{}, tests, benchmarks, examples)
{{with .TestMain}}
	{{.Package}}.{{.Name}}(m)
	os.Exit(int(reflect.ValueOf(m).Elem().FieldByName("exitCode").Int()))
{{else}}
	os.Exit(m.Run())
{{end}}
}
```

看到这里，`go test` 的主要逻辑已经看完辽。继续深入的话，则需要看看 testing 包辽。

## 小结

总体来说，`go test` 大的流程还是比较简单的
1. 先判断需要给哪些 package 执行测试。这个主要是用户在 go test 的参数中指定。
2. 给每个 package 生成一个 testmain.go 的文件，这些文件都共用一个模板，模板中强依赖 testing 库。
   也就是说测试的具体执行逻辑还得看 testing 库的实现才知道。
3. 然后分别执行 go build 生成 binary，接着运行这些 binary。

回头看看之前的三个问题
1. 第一个问题已经解答。
2. 第二个问题得等后续看了 testing 库的实现才知道如何实现。
3. 第三个问题也已经基本得到解答，go test 命令和 testing 库是强耦合。如果想不依赖 testing 库，
   则需要自己开发一个 xxx-test 的命令。

## 划水

又水了一篇，真好。台风的周末！接下来调研下去青海游玩的计划！

一人听歌，享受、但思绪容易复杂。<br/>
众人听歌，口味难调。<br/>
臭味相投不容易。



[issue-26289]: https://github.com/pingcap/tidb/issues/26289
[go-test-pkg]: https://github.com/golang/go/blob/052da5717e/src/cmd/go/internal/test/test.go#L41
[L579]: https://github.com/golang/go/blob/052da5717e/src/cmd/go/internal/test/test.go#L579
