---
title: Win/Linux虚拟环境使用笔记
permalink: article/win-setup
toc:
  number: false
  enable: true
date: 2020-07-01 13:59:08
tags:
categories:
---

在公司搬砖用的环境是一台win台式机，i7+A卡，配上双屏，一屏竖着看代码一屏横着看网页，全都公司报销了。自己配了个Niz的82键配列静电容键盘，35g压力，打起来像戳棉花糖一样。

<!-- more -->


## 怎么番茄
现在用的机场服务是[CNIX](https://ntt-co-jp.club)

订阅软件，Windows首推Clash for Win; macOS平台首推clash X。

### Git代理

代理是SS/SSR的话可以用socks5，下面两种方法本质上一样

```
# 鼠标停留在状态栏的小火箭上会显示端口
git config --global http.proxy 'socks5://127.0.0.1:7890'
git config --global https.proxy 'socks5://127.0.0.1:7890'
```

也可以直接编辑文件

```
vim ~/.gitconfig
# [http]
#     proxy = http://<主机ip>:<7890>
```

### `npm`淘宝源
`npm install`在ssr设置了全局代理的情况下还是会下载卡死，`npm config`配置代理实测并不能解决问题。可以配置淘宝源结局，如果已经设置了代理要`npm config edit`把代理相关的行删掉，再

```
npm config set registry http://registry.npm.taobao.org
```


## Linux虚拟机

以前用过VirtualBox和Mac的Parallel Desktop，总体来说Parallel Desktop的体验要好一些，毕竟收费，但还是存在一些比较烦人的bug。这次尝新了VMware Workstation。VMware也是付费程序，但由于不重视保护版权，激活码在网上很多，商用的话还是要支持正版。

用VMware在Windows下安装Ubuntu的[手把手教程](https://zhuanlan.zhihu.com/p/41940739)。

### 配置虚拟机的代理

- 假设主机上番茄软件用的是[Clash for win](https://github.com/Fndroid/clash_for_windows_pkg/releases)，General > 勾选Allow LAN（允许局域网的连接），记录下Port端口（默认7890）
- 切换到Ubuntu，Settings>Network，Network Proxy设置为Manual
- 四个选项都设置为`<主机ip>:7890`

VMware的网络模式最好是桥接模式，不推荐用NAT模式，虚拟机上的web服务会没办法广播到全局域网，给同事demo东西会不方便。桥接模式下，虚拟机和主机是并列关系，有自己的独立ip。但估计是权限的原因虚拟机虽然可以上网但是不能独立配置代理，只能通过主机的LAN连接走主机的代理。

### 配置`apt-get`使用代理

```bash
sudo vim /etc/apt/apt.conf
```

内容为`Acquire::http::proxy "http://<主机ip>:7890"`


## 在Win和Linux之间共享文件

出于安全考虑，VMware默认是打开文件共享，关机后在虚拟机设置菜单打开，选择主机上需要共享的目录。这里有个bug，重启虚拟机会导致虚拟机丢失共享的路径，需要再次手动锚定。

把这一段放在你的`.bashrc`或者`.zshrc`

```bash
alias mount='/usr/bin/vmhgfs-fuse .host:/ /home/yangrq/share -o subtype=vmhgfs-fuse,allow_other'
```

每次重启在shell里运行`mount`。


##远程办公
公司配了蒲公英的内网穿透服务，其实就是个各个大学都有的校园网vpn服务。下载蒲公英的客户端（全平台都有，包括ios），挂上代理，可以访问内网上的共享文件，web服务，或者远程桌面。远程桌面的软件可以用微软自己的Remote Desktop，或者蒲公英的产品向日葵。

对于数据库架设在局域网内的环境，有这个内网穿透在家可以持续开发调试，还是很方便的。

## Oracle数据库
因为现在写的东西可视化用的是eCharts（Python封装调用）+ Flask的组合，prototype当然要用一个开发足够快的数据库，选了non-SQL的数据库mongoDB，第一次使用非关系数据库，不过很好上手，eCharts本体又是个JS写的库（百度程序员很猛），这一套前后端交互用起来省不少心。

等IT把Oracle数据库弄好，转债和股票的数据就上生产环境了。以前在工行实习的时候，学了python怎么开发Oracle数据库。Oracle那套东西体型太臃肿，80%的东西都是BAT这种体量的公司才用得上的。感谢我坑的DBM课，SQL还是会写一点的😤。


