---
title: "Linux 桌面环境使用经验"
date: 2019-04-20 14:33:23 +00:00
permalink: /blogs/93
tags: [linux, kde, gnome]
categories: [工具啥的]
---
记录一下自己过去的使用感受和经验，避免重复踩坑。都是一些琐碎的事情。

## 系统初始化

初始化流程大致如下

1. 下载这两个 repo，其中 rcfiles 为必须品
   - rcfiles `git clone https://github.com:cosven/rcfiles.git`
   - Emacs `git clone https://github.com:cosven/.emacs.d.git`
2. 配置 clash 服务
3. 安装 dropbox，提取 ssh 密钥
4. 安装输入法 fcitx
5. 其它必须软件：chrome, telegram, konsole

## 配置 clash 服务

目前看到的最佳实践是把 clash 做成 systemd 的服务，clash.service 文件可以参考 rcfiles。
直接从 rcfiles 中软链到 `~/.config/systemd/user/`，然后 `systemctl --user daemon-reload`，
然后 enable 一下这个服务即可。

需要注意的是 clash 的配置文件，一般来说 clash 的配置文件可以直接从服务提供商下载，
基本上不需要改它的配置。port 以及 external-ui 可能需要小改一下。
ui 文件可以下载 clash-dashboard 的 gh-pages 上下载，git clone master 似乎不行。
clash 运行的 cwd 可能是 home 目录，因此配置的时候最好把路径写全。避免折腾。

## 软件对比

**GNOME 和 KDE**

在 2019 年初，我个人感觉 GNOME 唯一的优势是可以更好的处理：HiDPI 显示屏 + 普通显示屏。
而 KDE 只能统一一种分辨率。据说这是 Xorg 的问题，等 KDE on wayland？

**Chromium 和 Firefox**

Chromium 一个很头疼的问题就是我切换桌面环境的时候，需要重新登录，cookie 啥的都没了，而 firefox 不会。
但是 firefox 是真的难用，下载的交互和 IE 一样，还尝试打开文件，很奇怪。另外，地址栏的弹窗似乎总是置顶的，
感觉是个 bug。

_UPDATE 2019-4-23_: Chromium 内置了 KDE wallet 的支持，disable 它，就不需要登录了。

**Terminal Emulator: tilix 和 Konsole**

**albert 等东西不是很好用**

**snapd 是个大坑**

1. 很多 snap 的应用似乎都没人维护了，比如 docker
2. 对比 snap 和 AppImage，感觉 AppImage 生态要好些，用起来也比较方便

**截图**

用 [flameshot](https://github.com/lupoDharkael/flameshot) 可以达到类似 wechat/qq 截图那样的效果

## 快捷键设置

- Alt-F2 在 KDE 下是唤起 Plasma Search。在 GNOME 下是运行命令。

**Ctrl-Alt-H, Ctrl-Alt-L 切换 workspace**

**调换 win 和 alt**

- Apple 内置键盘参考 arch 文档切换 https://wiki.archlinux.org/index.php/Apple_Keyboard#Use_un-apple-keyboard
- HHKB 切换第五个开光
- KDE settings 中的 “swap alt 和 win” 设置没有从根本上切换这两个键，用起来很难受

**F7,F8,F9 切换成 Media 快捷键**

**terminal 复制、粘贴**

改成 Alt-C，Alt-V。

## 配置 ssh-agent

我的 ssh keys 是有一个密码的，所以每次使用都需要输入一次，ssh-agent 可以解决这个问题。
调研并实践后，发现[用 systemd user 启动 ssh-agent][systemd-ssh-agent]这个方案是最靠谱。
其他对比方案
1. 在 KDE 上尝试过 walletd。不好用，一方面配置很麻烦；另一方面，一些相搭配的组件 envoy 已经不维护了。

## 翻译软件

以前感觉 goldendict 不错，但现在基本上都处于有网的环境，浏览器的“划词翻译”这款插件感觉很不错。

### 字典 - goldendict

- 在这里下载英汉词典 https://github.com/skywind3000/ECDICT 的 release 页面。
- 这里可以下载汉英词典 https://mdict.org/categories/chinese/ ，这里也包含一些汉语词典。
- goldendict 在 Plsama 环境下快捷键可能会失效，建议给它设置一个全局快捷键。
  全局快捷键运行 goldendict，让它弹窗。弹窗时，可能会弹出 menu bar，一个 workaround 是隐藏它。
  https://github.com/goldendict/goldendict/issues/739

## KDE 上的字体问题

- Chromium Browser 中中文**显示不正常**的问题，fcitx 显示中文也不正常
- Emoji 在 feeluown 和 Konsole 程序中**不能显示**
- Konsole 显示中文也不正常

### Chromium Browser 中文显示不正常
之前 Chromium 中总是不能正确的显示 `关` 这个字，经过朋友 hexchain 的提醒，他说
你这个是个日文字体，于是意识到可能是 Chromium 渲染时 fallback 到了一个日文字体。

接着做了以下几件事情：

- 在 Chromium 中将 standard font 设置为 `Noto Sans CJK SC`
- 在 Chromium 中将 serif font 设置为 `Noto Serif CJK SC`
- 在 Chromium 中将 sans font 设置为 `Noto Sans CJK SC`

于是网页中的中文字体显示都正常的（不需要重启浏览器），但是还有个问题，
地址栏中仍然不能正确的显示中文，尝试重启浏览器，无效，请看后文。

### Konsole 不能显示 Emoji，显示中文不正常
按照下面步骤即可进行修复 Konsole 不能显示 Emoji 的问题：

- 修改 `~/.fonts.conf` 文件如下（ `~/.fonts.conf` 是 `~/.config/fontconfig/fonts.conf` 的软链）
- 运行 `fc-cache -f -v` 命令
- 重启 Konsole，即可发现 Emoji 已经显示正常，但是中文显示异常

```conf
<?xml version="1.0"?><!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
 <!-- <dir>~/.fonts</dir> -->
  <alias>
    <family>monospace</family>
    <prefer>
      <family>Ubuntu Mono</family>
      <family>Noto Color Emoji</family>
    </prefer>
  </alias>
```

我猜测 Konsole 的字体渲染流程应该是这样的：
Konsole 首先看用户在设置页面中设置了什么字体，如果这个字体能够渲染某个字，那么就用这个字体。
如果不能渲染某个字，Konsole 会去 fonts.conf 的 monospace 家族去找字体来渲染这个字。
所以一开始的时候，Konsole 不能显示 Emoji，因为它找不到字体来渲染 Emoji。
我们给 monospace 家族设置了 `<family>Noto Color Emoji</family>` 字体当作 fallback 字体之后，
Konsole 也就正常了。

但这时，中文字体显示仍然 **不正常**，我怀疑它应该是 fallback 到了一个什么系统日文字体。
所以我们要做的就是给它一个可以 fallback 的中文字体 `<family>Noto Sans CJK SC</family>`

```conf
<?xml version="1.0"?><!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
 <!-- <dir>~/.fonts</dir> -->
  <alias>
    <family>monospace</family>
    <prefer>
      <family>Ubuntu Mono</family>
      <family>Noto Sans CJK SC</family>
      <family>Noto Color Emoji</family>
    </prefer>
  </alias>
```

### Chromium 地址栏和 fcitx 显示中文不正常，FeelUOwn 不能显示 Emoji
按照上述思路：修改 fonts.conf 配置文件，运行 `fc-cache -f` 命令，重启这两个应用。
可以发现 Chromium Browser 地址栏可以正确的显示中文，FeelUOwn 也可以正常的显示 emoji 了。
最终配置请看 gist：https://gist.github.com/cosven/a0fae9230d63c4c80646b4e9ca647c0d

### 总结

经过三番五次的对字体配置的修改，有以下心得：

1. fonts.conf 会影响所有的 GTK/QT 软件
2. Chromium Browser 网页正文有自己的字体配置，可以进入 Chromium 的 settings 中设置。
但是它的地址栏貌似不会受这个配置影响（可能由于地址这一部分是 GTK 实现的？）
3. 系统有个默认的字体 fallback 逻辑
3. 在 `<prefer></prefer>` 中配置多个字体可以实现字体 fallback，它优先于系统默认 fallback 逻辑，**这个解决以上问题的关键**
4. 我们可以在 fonts.conf 中给每个种类（sans, sans-serif, monospace 三种）的字体都指定一个 fallback 字体


## 其它常见问题
### 突如其来的高 CPU

1. 检查 chrome 页面是否有动画、gif 等（其实还包括 Slack 等一些基于 webkit 的应用）

### 不能完全睡眠

（不确定下面这个方案是否真的可行，目前看似乎可以）
参考这个 https://wiki.archlinux.org/index.php/Mac#Wake_Up_After_Suspend

- ARPT 代表 wifi
- LID0 代表笔记本盖子

注：还有的人猜测是 USB 导致的，也有人猜测是鼠标导致的，我感觉把这些全部设置为 disable 就好了。

### Chrome 播放声音卡顿

这种方法，亲测可以解决。据说还可以通常杀掉 chrome 的 Utilities Audio 进程来解决。
```
pulseaudio -k
pulseaudio --start
```

[systemd-ssh-agent]: https://wiki.archlinux.org/title/SSH_keys_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E7%94%A8_systemd_user_%E5%90%AF%E5%8A%A8_ssh-agent)

### GRUB 引导被破坏
我自己遇到过了 windows 大更新把 grub 引导搞坏了问题，这个问题 arch 社区说经常会遇到。
我猜测是因为 windows 更新时，在自己的分区范围内，又搞了个新分区 windows recovery ，
导致 grub 找不到正确的分区了 🤔。瞎猜一下，如果 linux 的分区排在 windows 分区的前面，
或许可以规避这个问题。不过好在是直接进入 grub rescue 页面，修复起来还是挺快的。

大概就是以下几条命令吧
```
ls (hd0,gpt5)  # 先试试 linux 安装在哪个分区上
set root=(hd0,gptN)/
insmod normal
normal
```
