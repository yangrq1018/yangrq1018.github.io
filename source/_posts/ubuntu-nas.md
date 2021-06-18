---
title: Ubuntu挂载NAS网络驱动器
toc:
  number: false
  enable: true
description: 介绍Ubuntu如何访问NAS文件服务器
tags:
- linux
- ubuntu
categories: 技术
permalink: article/ubuntu-nas
---

办公室内有共享文件服务器(NAS), 用户名为<username>, 有密码保护. 如何通过linux (Ubuntu 20.04), 读写NAS上的文件

NAS的ip地址是10.168.1.136,映射在了\\nas地址.如果在windows上,直接在文件浏览器的地址栏中
输入\\nas就能访问,在只有命令行的Linux里就要麻烦一点

安装`smbclient`和`samba`

```bash
sudo apt update
sudo apt install smbclient
sudo apt install samba
```

列出\\nas地址下的文件夹
```bash
smbclient -L \\nas -U <username>
```
然后按提示输入账号对应的密码

```
  Sharename       Type      Comment
        ---------       ----      -------
        IT              Disk
        Public          Disk
        交易          Disk
        市场          Disk
        研究          Disk
        程序交易    Disk
        股东          Disk
        运营          Disk
        IPC$            IPC       IPC Service ()
        home            Disk      Home directory of <username>
```
看到nas里的各个文件夹

现在把 `//nas/研究` 文件夹映射到本地文件夹 `/mnt/research`, 这样我们就能通过访问 `/mnt/research`, 读写NAS里的文件.

```bash
sudo mount -t cifs -o username=<username>,password=<password>,vers=1.0 //nas/研究 /mnt/research
```

> 注意"vers=1.0"是一定需要的,否则会报错

我们选择了cifs的文件系统格式, 有可能会报这个错

```
mount: /mnt/research: bad option; for several filesystems (e.g. nfs, cifs) you might need a /sbin/mount.<type> helper program.
```
提示我们没有这个文件系统的程序, 所以安装
```bash
sudo apt-get install cifs-utils
```

确保`/sbin/mount.cifs`文件存在, 然后上面的`mount`命令就可以成功了.

```bash
cd /mnt/research
ls -l
total 0
d--------- 1 root root  0 12月 30 09:37 '#recycle'
drwxrwxrwx 1 1027 users 0 1月  11 10:07  ****
drwxrwxrwx 1 1027 users 0 7月  28  2020  ****
drwxrwxrwx 1 1027 users 0 6月  18 14:33  ****
drwxrwxrwx 1 1034 users 0 6月  10 09:40  ****
drwxrwxrwx 1 1027 users 0 5月  26 17:38  ****
drwxrwxrwx 1 1027 users 0 6月  11 17:21  ****
```
(文件夹名隐去)

