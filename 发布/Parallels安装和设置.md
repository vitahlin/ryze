---
title: Parallels安装和设置
slug: parallels-installation-setup
date: 2024-09-08
desc: 本文详细介绍了在 macOS M1 环境下使用 Parallels Desktop 安装 Ubuntu 虚拟机的步骤，包括如何选择合适的镜像、安装过程中的设置、以及如何通过 SSH 连接 Ubuntu 虚拟机。还介绍了如何安装 Parallels Tools 以实现文件共享功能，并提供了常见问题的解决方法和实用参考链接，帮助用户顺利完成虚拟机的安装与配置。​
featuredImage: https://sonder.vitah.me/featured/71a0eeda902c98b57ae5ca5f4845c93a.webp
categories:
  - Tools
tags:
  - VirtualMachine
---

# 环境准备

1. Parallels 版本19
2. macOS M1 版本 14.5
3. 下载 ubuntu的 iso 镜像

# 安装

## 选择合适的镜像

这里以安装 Ubuntu 为例，我们可以安装 Parallels 提供支持的 Ubuntu镜像，也可以选择自己下载的镜像，这里以 Ubuntu 22.04 为例，下载的 ios 文件为：ubuntu-22.04.4-live-server-amd64.iso。下载地址：

[https://old-releases.ubuntu.com/releases/](https://old-releases.ubuntu.com/releases/)

实际上这个文件是支持 M1 芯片的，但是在Parallels进行安装的时候会提示错误：

> **The specified image cannot be used because your Mac is equipped with the Apple M series chip that doesn’t support Intel-based operating systems.**

具体错误如图：
![](https://sonder.vitah.me/blog/2024/6ef0ea1511a063980e0527dd6cd99986.webp)

这个问题应该是 Parallels 本身的 BUG，换成 Ubuntu22.04.3 的服务器镜像就能正常识别并且安装。

## 安装时设置

如果有需要还可以修改虚拟机配置，比如CPU核心数和内存，**注意这个只能在安装时修改。**

![](https://sonder.vitah.me/blog/2024/a97e1bbf4d3a86cdfae1ebe9e3bbbbef.webp)

后续直接一步步安装即可。

# 设置

## ssh连接ubuntu虚拟机

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

#### 1. 设置源

先关闭虚拟机，然后设置 CD/DVD 的源，把它连接到 `ptl-tools-lin-arm.iso`，如图：
![](https://sonder.vitah.me/blog/2024/00b7519e2624989c134195ed24cc31df.webp)

这个镜像文件的位置在  `/Applications/Parallels Desktop.app/Contents/Resources/Tools/` 目录下，针对不同的主机和 CPU 选择合适的镜像，其中 `lin` 代表 linux，`arm` 代表支持 arm 芯片的。

#### 2. 调整启动顺序

![](https://sonder.vitah.me/blog/2024/daed411f15101a73b31349b2a265b6e5.webp)

如图，调整启动顺序，把 CD/DVD 调到第一位，接着是硬盘，然后 reboot 重启虚拟机。

#### 3. 挂载目录

重启完成后，进入ubuntu虚拟机，

```bash
cd /media
ls
```

这时候会发现目录下没有任何文件，是空的。继续执行操作：

```bash
vitah@ubuntu-vm:~$ cd /media
vitah@ubuntu-vm:/media$ ls

vitah@ubuntu-vm:/media$ sudo mkdir cdrom
[sudo] password for vitah: 

vitah@ubuntu-vm:/media$ sudo mount -o exec /dev/cdrom /media/cdrom
mount: /media/cdrom: WARNING: source write-protected, mounted read-only.
```

#### 4. 执行安装脚本

挂载后，在 `/media` 目录下就出现了 `/cdrom` 文件夹，执行安装脚本：

```bash
vitah@ubuntu-vm:/media$ ls
cdrom

vitah@ubuntu-vm:/media$ cd cdrom
vitah@ubuntu-vm:/media/cdrom$ ls
install  install-gui  installer  kmods  tools  version
vitah@ubuntu-vm:/media/cdrom$ sudo ./instal
```

执行 `./install` 后会出现如下界面，一步一步确定，现在就是在安装中，等待安装即可：
![](https://sonder.vitah.me/blog/2024/192baee0f3627dc191f442e19114e638.webp)


### 设置本机共享文件夹

安装完后后停止虚拟机，在设置中设置共享目录，如图：

![](https://sonder.vitah.me/blog/2024/c10f9cd6b84ba3f58bfb7c29bcfb1be9.webp)


然后设置文件夹 `/vm` 进行共享：

![](https://sonder.vitah.me/blog/2024/10d99074df7de2e5944dea5db5156d0b.webp)


设置完成后重启虚拟机，就能在 `/media/psf` 目录下看到共享的文件夹：

```bash
vitah@ubuntu-vm:~$ cd /media/psf
vitah@ubuntu-vm:/media/psf$ ls
vm
```

# 参考链接

- [parallels+desktop+linux共享文件夹,Parallels Desktop共享本机目录给虚拟机-CSDN博客](https://blog.csdn.net/weixin_42575505/article/details/116674141)
- [Getting an Ubuntu 22 Desktop Virtual Machine on Apple Silicon](https://www.nequalsonelifestyle.com/2022/07/22/getting-ubuntu-22-desktop-vm-on-apple-silicon/#creating-the-virtual-machine)
