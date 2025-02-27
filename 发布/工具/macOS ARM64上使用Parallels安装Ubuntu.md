---
title: Parallels安装和设置
slug: macos-arm64-parallels-install-ubuntu
date: 2024-09-08
desc: 本文详细介绍了在 macOS M1 环境下使用 Parallels Desktop 安装 Ubuntu 虚拟机的步骤，包括如何选择合适的镜像、安装过程中的设置、以及如何通过 SSH 连接 Ubuntu 虚拟机。还介绍了如何安装 Parallels Tools 以实现文件共享功能，并提供了常见问题的解决方法和实用参考链接，帮助用户顺利完成虚拟机的安装与配置。​
featuredImage: https://sonder.vitah.me/featured/71a0eeda902c98b57ae5ca5f4845c93a.webp
categories:
  - Tools
tags:
  - VirtualMachine
---

# 环境准备

1. Parallels 版本 20.2.0
2. macOS M1 15.3
3. 下载 ubuntu的 iso 镜像

# 安装

## 选择合适的镜像

我们可以安装 Parallels 提供支持的 Ubuntu 镜像，也可以选择自己下载的镜像，这里我们通过镜像安装：
![](https://sonder.vitah.me/ryze/95249c02d0552655b202b6628759d58b.webp)



下一步后，选择已经下载的iso镜像即可。比如 Ubuntu 22.04 文件为：ubuntu-22.04.4-live-server-amd64.iso，下载地址：[Index of /releases](https://old-releases.ubuntu.com/releases/)

我们下载的这个镜像就是 amd64 的，但是在Parallels进行安装的时候会提示错误：

> **The specified image cannot be used because your Mac is equipped with the Apple M series chip that doesn’t support Intel-based operating systems.**

具体错误如图：
![](https://sonder.vitah.me/ryze/25c0b85c375a51dd5e23a9cfcdb4f013.webp)

这个问题目前解决不了，换成 Ubuntu 22.04.5 的服务器镜像就能正常识别并且安装，只能尝试选择匹配的镜像。

**支持M1的Ubuntu镜像**

| 镜像名称                  | 下载地址                                                         |
| --------------------- | ------------------------------------------------------------ |
| ubuntu server 22.04.3 | magnet:?xt=urn:btih:931CAE3D8B23994E59EEAA0CEAA1C8CB011F693B |
| ubuntu server 22.04.5 | magnet:?xt=urn:btih:502D81DF4E4CD216BCE32EB05CC54B5A5359DEE0 |
| ubuntu server 24.04.1 | magnet:?xt=urn:btih:C8814DCE02E49A455F60002FBABBFE4E4D3F22A9 |

## 开始安装

选择镜像后，开始下一步，设置名称和位置：
![](https://sonder.vitah.me/ryze/a002927fbfc5e421751ae3886fff4e28.webp)

如果有需要还可以修改虚拟机配置，比如CPU核心数和内存，**注意这个只能在安装时修改，需要勾选上图中的 `Customize settings before installation`**
![](https://sonder.vitah.me/ryze/32b484c1ff7ffc6aaebc6ae483a9c83e.webp)

如果需要修改配置，选择Configure后在 Hardware 中设置CPU和内存：
![](https://sonder.vitah.me/ryze/ff847a15adccf59da1c358186b1a51be.webp)

Continue后就进入 Ubuntu Server的安装界面，选择对应的设置一步一步安装即可，安装过程可能会比较久，提示 Installation complete 后，选择Reboot完成安装。

# 安装后设置

## 使用ssh连接ubuntu虚拟机

在虚拟机中执行命令 `ip addr` 查看当前虚拟机的IP是多少，结果如下：

```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:1c:42:58:a7:e6 brd ff:ff:ff:ff:ff:ff
    inet 10.211.55.6/24 metric 100 brd 10.211.55.255 scope global dynamic enp0s5
       valid_lft 926sec preferred_lft 926sec
    inet6 fdb2:2c26:f4e4:0:21c:42ff:fe58:a7e6/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 2591679sec preferred_lft 604479sec
    inet6 fe80::21c:42ff:fe58:a7e6/64 scope link 
       valid_lft forever preferred_lft forever
```

可以看到当前虚拟机的IP地址是 10.211.55.6，后续在 macOS中就能通过 ssh 连接虚拟机：

```bash
❯ ssh vitah@10.211.55.6
vitah@10.211.55.6's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-107-generic aarch64)
```

## 目录共享

### 安装 Parallels Tools

1. 设置源
先关闭虚拟机，然后设置 CD/DVD 的源，把它连接到 ptl-tools-lin-arm.iso，如图：
![](https://sonder.vitah.me/ryze/5cf8d0167af0fe4455ced255773841d0.webp)

这个镜像文件的位置在  **/Applications/Parallels Desktop.app/Contents/Resources/Tools/** 目录下，针对不同的主机和CPU选择合适的镜像，其中 `lin` 代表 linux，`arm` 代表支持 arm 芯片的。

2. 调整启动顺序，把CD/DVD调到第一位，接着是硬盘，然后reboot重启虚拟机
![](https://sonder.vitah.me/ryze/1c7c628654342e3a144f8fabe9282d16.webp)


3. 重启完成后，进入ubuntu虚拟机，开始挂载目录
```bash
cd /media

ls
# 这时候会发现目录下没有任何文件，是空的。继续执行操作

vitah@ubu2404:~$ cd /media
vitah@ubu2404:/media$ ls

vitah@ubu2404:/media$ sudo mkdir cdrom
[sudo] password for vitah: 

vitah@ubu2404:/media$ sudo mount -o exec /dev/cdrom /media/cdrom
mount: /media/cdrom: WARNING: source write-protected, mounted read-only.

```

4. 执行安装脚本，挂载后，在 `/media` 目录下就出现了 `/cdrom` 文件夹，执脚本：

```bash
vitah@ubu2404:/media$ ls
cdrom

vitah@ubu2404:/media$ cd cdrom
vitah@ubu2404:/media/cdrom$ ls
install  install-gui  installer  kmods  tools  version
vitah@ubuntu-vm:/media/cdrom$ sudo ./install
Started installation of Parallels Guest Tools version'20.2.0.55872'
```

出现 `Started installation of Parallels Guest Tools version'20.2.0.55872'`就是在安装中，安装完成后会出现：
```bash
Parallels Guest Tools were installed successfully!
```

### 设置本机共享文件夹

安装完后后停止虚拟机，在设置中设置共享目录，如图：
![](https://sonder.vitah.me/ryze/56e5e036abd2d12f8f934c4036641283.webp)

选择Manage Folders，设置Mac文件夹目录 dev 为共享文件夹，如图：
![](https://sonder.vitah.me/ryze/f94f4409d7edadd314706396395b2a88.webp)

设置完成后重启虚拟机，就能在 `/media/psf` 目录下看到共享的文件夹：
```bash
vitah@ubu2404:~$ cd /media/psf
vitah@ubu2404:/media/psf$ ls
dev
```

# 参考链接

- [parallels+desktop+linux共享文件夹,Parallels Desktop共享本机目录给虚拟机-CSDN博客](https://blog.csdn.net/weixin_42575505/article/details/116674141)
- [Getting an Ubuntu 22 Desktop Virtual Machine on Apple Silicon](https://www.nequalsonelifestyle.com/2022/07/22/getting-ubuntu-22-desktop-vm-on-apple-silicon/#creating-the-virtual-machine)