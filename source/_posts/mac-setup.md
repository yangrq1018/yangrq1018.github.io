---
title: Mac程序环境初始设置指南
permalink: article/mac-setup
toc:
  number: false
  enable: true
date: 2020-06-21 00:06:20
tags:
- Mac
- macOS
categories: 技术
---

更新于2020年6月。

<!-- more -->

## Python Environment

- Anaconda distrbution
  - 在Catalina的某一个版本后Dock里的Navigator就打不开了，因为Apple修改了第三方应用的权限，现在Anaconda无法放在用户根目录下的`~/Anaconda`，而只能在`~/opt`下，所以Dock里的快捷方式的目标失效了。不影响终端里调用任何Anaconda的功能，如何解决Dock图标的问题请谷歌
- Tensforflow，通过pip安装，需要打开小火箭的Global mode

##  macOS 特色, IDE, 编辑器

- Homebrew
- JetBrains Pycharm

## 其他语言和工具

- Node.js (以及打包的npm)
- 博客：Hexo和Next
- Vscode
- Xcode：很多包，比如brew需要Xcode的command line tool，或者是需要完整的Xcode应用程序
- 🍅：ShadowsocksX-NG-R8
- Oh-my-zsh：`zsh`已经预装在新的macOS上，而且是默认终端解释器
- 终端美化: iTerm
  - 推荐Pure的终端主题方案
    - 需要在Oh-my-zsh里将默认主题更改为空，使Pure生效。`ZSH_THEME=""`
- 选装：Matlab，R
- 选装：JetBrains Mono等线字体，替换终端的Monaco字体

## 办公

- 高等院校提供的正版Office 365套装，包括Word，Excel和Outlook
- Chrome
- Typora：超好用的Markdown编辑器，比Word轻度文字工作
- 选装：Latex (macTex distribution)
  - 以及适配的WYSIWYM (What you see is what you mean) 编辑器 Lyx，编辑公式较多的Latex文章非常好用

## 学习

- 欧陆词典mac版，可以快捷键打开查词小窗口，一秒钟查到想要的词然后迅速切回原来的工作

## 通讯和娱乐

- QQ
- QQ音乐
- 微信mac版
- Telegram for macOS
  - 需要配合🍅使用
  - ssr下直接打开tg是登陆不了的，输入手机号后会timeout。解决方法是手动配置tg的proxy。模式选择socks5，地址127.0.0.1，端口在小火箭的advance preference里确认，一般默认1086端口。设置proxy后重启tg，一定要重启！然后应该可以正常输入手机号码和验证码后登陆

## 设置

`~/.zshrc`里设置终端的端口使用ssr的端口。具体端口查看小火箭的设置，一般是1086。

```zsh
# ssr termimal proxy
alias proxy='export all_proxy=socks5://127.0.0.1:1086'
alias unproxy='unset all_proxy'
alias ip='curl cip.cc'
```

获得🍅能力之后，建议下载如Tensorflow之类的包的时候打开全局模式，解决源的速度慢的问题。
