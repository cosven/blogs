---
title: 给 FeelUOwn 设计合理的 DSL - 简单考古 shell 的词法分析规则
date: 2022-01-21 22:47:00 +08:00
permalink: /blogs/10014
tags: [music, player, lexer]
categories: [音乐, 稍微正经点的]
---


## 一点背景

音乐播放器 [FeelUown][feeluown] 向外提供了基于 TCP 的 RPC 服务，该服务提供了控制播放器的接口。
这些接口可以让使用者控制播放器的播放、暂停、搜索等功能。用户可以使用 telnet/netcat
连接到服务端口，输入相应的命令文本来调用这些接口；也能够通过 fuo 命令行工具来调用。
但怎样让命令文本和 fuo 命令行有一个统一的使用方式呢？

## 困境

在当前的设计中，命令文本和 fuo 命令行的使用方式存在不一致。举个例子，
在 netcat 中输入如下文本可以实现“从关键字‘周杰伦’从网易搜索符合要求的歌曲，并以 json 格式返回”
```
search 周杰伦 [source=netease,type=song] #: format=json
```
使用 fuo 命令需要输入如下
```sh
fuo search 周杰伦 source=netease,type=song --format=json
```

当前的设计是否合理？怎样修改才能让两种更加统一呢？带着这两个问题，
我决定先看看 shell 是怎样解析命令行文本的，了解它有哪些拓展的可能性。
要知道 shell 如何解析命令行文本，我觉得可以先看看它的词法分析是如何实现的。
我找到两份资料，一份是 shell 的详细说明文档；一份是名为 shlex 的 Python 标准库。

## 探索

### Shell 的 token 识别规则

规则细节参考 [Shell 详细说明文档][shell-spec] 的 Token Recognition 章节。
Token 识别一个重要的内容是明确 token 与 token 之间的分隔。从规则细节中，
可以发现 token 的分割符是空格（\<blank\>）。而 token 的首字符有如下几种：

- io_here 的标识符
- 操作符或单词的首字符
- 引号和反斜线（quote, \<backslash\>）
- 表达式标记（$, \`）
- 注释（\#）

### Python shlex 库如何模拟 shell 的解析

影响 shlex 与常见 Unix shells 的兼容性的参数主要有四个：`wordchars`, `punctuation_chars`,
`posix` 和 `whitespace_split`。

`wordchars` 的默认包含所有的 ASCII 字母数字（ASCII alphanumerics），以及下划线。
`punctuation_chars` 设置为 True 的时候，`~-./*?=` 这些字符也会包含在 `wordchars` 集合内。
而 `();<>|&` 这些字符则会被解析为单独的标记（token），`posix` 为 `True` 时，
拉丁语的重音字符也包含在 `wordchars` 集合内。

不难发现，`wordchars` 相当于一个白名单，`punctuation_chars` 则相当于一个黑名单。
在白名单内的字符都会被当做单词中的一个字符，而在黑名单的字符都会被解析成一个单独的标记。
`posix` 参数是通过改变这个白名单来间接改变 shlex 的行为。而当 `whitespace_split`
为 `True` 时，shlex 完全忽略白名单。

总的来说，为了让 shlex 尽可能模拟 shell 的行为，可以打开如下

```python
>>> import shlex
>>> s = shlex.shlex('cmd x y z --o1=o1 --o2 o2?',
...                 punctuation_chars=True, posix=True)
>>> s.whitespace_split = True
>>> list(s)
['cmd', 'x', 'y', 'z', '--o1=o1', '--o2', 'o2?']
```

当然，这并不等同于 shell 的解析行为，从 [shell 的详细说明文档][shell-spec]中可以看到，
`$\\"'\`` 这几个也有特殊的处理方式。举个例子：
```python
>>> import shlex
>>> s = shlex.shlex('cmd x y z --o1=o1 --o2 $o2',
...                 punctuation_chars=True, posix=True)
>>> s.whitespace_split = True
>>> list(s)
['cmd', 'x', 'y', 'z', '--o1=o1', '--o2', '$o2']
```

而 shell 会把 `$o2` 识别为一个变量，并在运行的时候进行替换（substitution）。

> **_关于 ASCII alphanumerics_** 根据[维基百科][Alphanumeric]记录，在 POSIX 标准中，
> 它定义如下：
> In the POSIX/C[2] locale, there are either 36 (A-Z and 0-9, case insensitive)
> or 62 (A-Z, a-z and 0-9, case-sensitive) alphanumeric characters.
>
> 因此，我们可以看到很多语言的词法分析器会有类似如下的正则
> ```python
> re.compile(r'[a-zA-Z0-9_]')
> ```

## 结论

刚脑袋实在转不动了，于是去洗了澡澡。我就在想啊，为啥不直接复用一下 shell 的格式呢？
百利而无一害吧。好，又水了一篇博客，水总比不写好，嘿嘿。

[Alphanumeric]: https://en.wikipedia.org/wiki/Alphanumericals
[shell-spec]: https://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html
[feeluown]: https://github.com/feeluown/FeelUOwn
