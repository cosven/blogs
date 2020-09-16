---
title: Linux 环境搭建经验
date: 2020-09-16 10:00:00 +08:00
permalink: /blogs/10006
tags: [linux, daily]
---

没几个月都要折腾一次 linux，比如初始化一个开发机呀，于是就记录一下。

发现自己之前还记录过 [Linux 桌面环境使用经验](http://cosven.me/blogs/93)，
之前这一篇算是 GUI 相关的，这一片偏命令行相关的把。

### 使用 kvm 安装虚拟机

一个示例脚本

```sh
#!/bin/bash

virt-install --name=cosven-dev \
  --vcpus=8 \
  --memory=16384 \
  --graphics vnc,listen=0.0.0.0 \
  --console pty,target_type=serial \
  --cdrom=/data0/cosven/ubuntu-18.04.4-live-server-amd64.iso \
  --disk path=/data0/vms/cosven-dev,size=128,format=qcow2,sparse=false \
  --os-variant=ubuntu18.04 \
  --debug
```

### 初始化一台 linux 机器供自己使用

1. 免密登录服务器 id_rsa.pub -> authorized_keys
2. git clone git@github.com:cosven/rcfiles.git
3. git clone git@github.com:cosven/.emacs.d.git
4. 安装 https://github.com/BurntSushi/ripgrep
5. 安装 fzf https://github.com/junegunn/fzf#using-git


### 在 linux 启动一个 http proxy

关于 proxy：几乎所有 linux 软件都会识别 `http_proxy` 这个环境变量，
还有一部分软件会识别 `all_proxy` 这个环境变量，
比如 `export all_proxy="socks5://127.0.0.1:1090"`

1. 简易临时的 proxy 可以使用 ssh 的 socks proxy `ssh -D PORT USER@HOST`
2. 简易不临时的推荐使用 [tinyproxy](https://www.archlinux.org/packages/?name=tinyproxy)。
   apt/aur 都可以直接安装。使用前将配置文件（一般在 `/etc/tinyproxy/tinyproxy.conf`）
   中的 `Allow 127.0.0.1` 注释掉，这样就可以允许所有连接了。

`all_proxy` 据说一般是用来设置 socks 代理地址的，已知的是 curl 会识别它。
目前没有搜到关于这个环境变量的官方的解释。

简单搜索了下，它可以转发 TCP/UDP 请求。大概可以理解为它在真正的 IP 包上又包了一层。

#### 如何让 git 使用 proxy 呢？

暂时想到的办法是让 git 走 http 协议来 push `git push http://xx master`。
如果 `git push git@github..` 这种形式的话，目前还没找到好办法。
讲道理它应该也是走 socks 代理的才对，如果 git 内部支持的话，但实测好像不行。

### 放开 Open Files 限制

修改 `/etc/security/limits.conf` 文件配置，添加一行 nofile 的配置

```
#ftp             -       chroot          /ftp
#@student        -       maxlogins       4

*                -       nofile          1000000
# End of file
```

这个对已经启动的 bash 的子进程不生效，还可能对某些系统不生效，希望后面可以补充下。
另外有个问题：
- [ ] A 主机达到 limit，B 主机尝试建立与 A 的链接，B 收到的是什么？timeout/refused？
