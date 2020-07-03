---
title: Windows 10的使用心得
permalink: article/win-setup
toc:
  number: false
  enable: true
date: 2020-07-01 13:59:08
tags:
categories:
---

在公司上班配了一台Windows 10台式机，戴尔的商业机，配置

- Intel i7 9700
- Intel UHD630集显
- AMD RX550集显
- 32G RAM
- 512 Samsung SSD
- 两块宏基2K屏
- NB双屏挂架

## 番茄问题

`npm`下载缓慢问题

Win下的ssr配置并不麻烦，但诸如Powershell, cmd的终端使用代理就很麻烦了，尤其是相比unix系统简单设置一下环境变量就搞定，windows的各种命令行软件接入互联网走的路径差异很大。`git`提高下载速度可以直接通过配置`git`的私有变量实现

```
# 鼠标停留在状态栏的小火箭上会显示端口
git config set http.proxy 'socks5://127.0.0.1:1080'
git config set https.proxy 'socks5://127.0.01:1080'
```

而且直连socks，下载速度飞快。

`npm install`在ssr设置了全局代理的情况下还是会下载卡死，`npm config`配置代理实测并不能解决问题。可以配置淘宝源结局，如果已经设置了代理要`npm config edit`把代理相关的行删掉，再

```
npm config set registry http://registry.npm.taobao.org
```

然后

```
npm install
```

速度飞快

### Linux虚拟机

本科上C++的时候用过VirtualBox的Ubuntu，觉得不太好用。后来在Mac下就花钱买了个Parallel，体验好很多。这次尝试了VMware的虚拟机，VMware也是付费程序，不过由于不重视保护版权，激活码在网上并不难找到。有能力的话还是要支持正版。

如何用VMware在Windows下安装Ubuntu，[手把手教程](https://zhuanlan.zhihu.com/p/41940739)。

安装好Ubuntu虚拟机后，有个复杂的问题就是配置虚拟机的代理。即使在主系统上打开ssr，虚拟机上也是无法直接翻墙的。

- 右键小火箭，选项设置
- 勾选“允许来自局域网的连接”，端口设置为1080
- VMware的网络选择为NAT模式，注意不是桥接模式！
- 在shell里`ipconfig`，VMnet8对应的一项就是NAT模式的网络，复制IPv4地址，比如我的是`192.168.43.1`
- 切换到Ubuntu，Settings>Network，Network Proxy设置为Manual
- 所有Proxy都是`192.168.43.1:1080`

打开火狐浏览器测试google.com能否访问。



如何在Win和Linux之间共享文件

出于安全方面的考虑VMware默认是不打开文件共享