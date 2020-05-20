---
layout: post
title: Golang 学习笔记
date: 2020-05-19 18:38:00 +08:00
tags: [golang]
---

把之前的每周阅读的 Golang 部分总结到这里。


### golang 项目目录结构

<https://github.com/golang-standards/project-layout>

golang 项目目录结构标准，讲了常见的 pkg/cmd/configs 等目录的作用。

### Godoc: documenting Go code

文章没啥技术含量，[链接](https://blog.golang.org/godoc-documenting-go-code)。

文章简单讲了 go 的函数文档该怎么写，还说 godoc 借鉴了 Python docstring 和 Javadoc 的文档设计，但是比它们更简单。

有个有趣的点是：godoc 没有机制给函数函数加注释，所以问题来了？到底有没有必要给函数参数加注释呢？


### logging
出发点：我知道 go 标准库的 log 有些不足，不过 TiDB 的 log 代码又有些复杂，暂时没有时间细看，想了解下 golang logging 的最佳实践。

[GO Logger 日志实践](https://www.imhanjm.com/2017/05/19/go%20logger%20%E6%97%A5%E5%BF%97%E5%AE%9E%E8%B7%B5/)
这篇文章讲了标准 log 的一些问题：没有 level 等，不过标准库都是在用 log 这个模块。
文章也提到了 zap 和 logrus 这两个常见的库，zap 性能优于 logrus，但 logrus 使用起来更加方便。作者还说，大公司都会定制自己的 log。

另外一个问题：TiDB 里面为什么既用了 logrus，也用了 zap？
git blame 看一下 `logutil/log.go` 这个文件，发现这样一个 PR: [*: start replacing logger with zap logger ](https://github.com/pingcap/tidb/pull/9279)

### constants
疑问：
- [x] constant 是否需要给定类型？
  - 不需要，从[这篇博客](https://blog.golang.org/constants)可以看出
- [x] 实践中，constant 和枚举的关系和区别？
  - 没看到相关资料，不过 golang 的枚举常常会使用 iota 来完成
- [x] constant 命名是否要加入 namespace？命名是否全部大写？
  - 命名推荐[驼峰命名](https://stackoverflow.com/a/22688926/4302892)
  - namespace 没有相关资料提到，可能要从更上层来解决命名问题

### struct 方法是否需要先判断实例是否为 nil？

struct 的方法有两种定义方式，一种是用 Pointer，一种是用 Value。

这里有个[说法](https://dev.to/chen/gos-method-receiver-pointer-vs-value-1kl8) ，说是共享值就用 pointer，否则用 value。

[effective_go](https://golang.org/doc/effective_go.html#pointers_vs_values)里面只说了：用指针的话，可以改变自己。

### golang defer/panic/recover
背景：一个 web 程序如果出现意外错误，程序应该是可以返回 500 的，这样就不需要处理每个错误了？那 golang 中怎样实现这个逻辑呢？

参考文章：https://blog.golang.org/defer-panic-and-recover

文章讲 defer/panic/recover 也是 flow control 的一种，可以理解每个 goroutine 都有一个调用栈，defer 语句在每个函数结束的时候都会调用。可以想象在栈上，caller 函数下一层就是 defer，defer 之间有先进后出的特点。recover 操作需要放在 defer 里面。

### golang 方法名大小写如何抉择？
根据官方资料：要 export 给另外一个包的必须大写（语法级别的限制）。
而我自己遇到的问题是：我有一个 web 应用，里面很多逻辑本身就不需要 export，这时候，我就全部用小写吗？目前查资料的结论是的。

### golang 错误/异常 定义与处理

参考文章：https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully

自己最近写了 1k 行左右 go 代码，但是感觉 Go 代码里面到处都需要处理 error，自己写了许多重复代码，于是想看看相关资料，有没有办法避免。

<https://blog.golang.org/error-handling-and-go>

- 设计上，go 的 `Error()` 方法应该负责把上下文信息都附带进来
- `fmt.Errorf` 可以方面的 new 一个 error，常用在函数返回上
- 对于需要重复频繁处理 error 的情形，博客中说可以自己定义一个 Error 类，然后函数返回这个 Error，在一个地方做统一处理。比如 HTTP handler 的话，就是加个 err handler。

### 理解 Go interface 的 5 个关键点

https://sanyuesha.com/2017/07/22/how-to-understand-go-interface/

文章提到的几个点对目前的我挺有价值的：
1. go 不会对 类型是interface{} 的 slice 进行转换
    ```golang
    func printAll(vals []interface{}) { //1
        for _, val := range vals {
            fmt.Println(val)
        }
    }

    func main(){
        names := []string{"stanley", "david", "oscar"}
        printAll(names)
    }
    ```
    这段代码会报 cannot use names (type []string) as type []interface {} in argument to printAll 错误

1. go 中函数都是按值传递即 passed by value
1. go 会把指针进行隐式转换得到 value，但反过来则不行

文章推荐了一篇：关于 go interface 底层实现的文章。
- [ ] https://research.swtch.com/interfaces

### 一些细节

#### go 什么时候用 make，什么时候用 var?

https://stackoverflow.com/questions/25543520/declare-slice-or-make-slice

根据回答中的描述，大多数情况都是用 `var`。一般来说，当 var 不能很好满足你需求的时候，就可以考虑用 make 了。
另外，`var s []int` 的 s 是个 `nil`，而 `s := make([]int, 0)` 的 s 不是 `nil`，而是一个长度为空的 slice。

另外，var 出来的 channel 只能是个同步 channel，而 make 可以创建出一个有缓冲区的 chan。

#### select

> The select statement lets a goroutine wait on multiple communication operations.
>
> A select blocks until one of its cases can run, then it executes that case. It chooses one at random if multiple are ready.
>
> --- https://tour.golang.org/concurrency/5

#### data race 是个啥？在 golang 中表现是怎样？

参考 [https://stackoverflow.com/a/18049303/4302892](https://stackoverflow.com/a/18049303/4302892)

> The definition of a data race is pretty clear, and therefore, its discovery can be automated. A data race occurs when 2 instructions from different threads access the same memory location, at least one of these accesses is a write and there is no synchronization that is mandating any particular order among these accesses.

看了这个定义，就比较容易理解一个现象：golang 进程因为 data race 而退出。

参考 [https://blog.regehr.org/archives/490](https://blog.regehr.org/archives/490)
race condition: 竞态条件

> A race condition is a flaw that occurs when the timing or ordering of events affects a program’s correctness.

In practice there is considerable overlap: many race conditions are due to data races, and many data races lead to race conditions. On the other hand, we can have race conditions without data races and data races without race conditions.

**附加问题** `critical section` 和两者的关系又是怎样？
参考资料：[http://ifeve.com/race-conditions-and-critical-sections/](http://ifeve.com/race-conditions-and-critical-sections/)

> 当两个线程竞争同一资源时，如果对资源的访问顺序敏感，就称存在竞态条件。
> 导致竞态条件发生的代码区称作临界区。

从这句话，可以推断出，临界区和 data race 是没啥关系的。
