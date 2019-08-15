---
layout: post
title: 远程显示Linux服务器GUI程序VisualVM
categories: Test
description: VisualVM 通过检测 JVM 中加载的类和对象信息等帮助我们分析内存使用情况，我们可以通过 VisualVM 的监视标签对应用程序进行内存分析。
keywords: VisualVM, Java, Analyzer 
---

# 远程显示Linux服务器GUI程序VisualVM

## 一、简介

有些时候，有些程序可能需要依赖图形界面才能启动，例如安装Oracle时（其实oracle支持命令行安装），例如需要启动一个图形界面的浏览器如firefox。
 作为服务端的系统，通常不会安装臃肿的图形界面。
 那么如何在不安装图形界面的的情况下启动图形界面的？听起来很矛盾，但是实际上是可行的。

X Window System（常被简称为X11或X），是一套基于X display protocol的windowing system，X GUI环境的功能包括窗口的绘制、移动，以及与鼠标、键盘等输入设备的交互。

X采用C/S模型（这是关键）：一个X server 和多个应用程序（client）通信。server接收client的请求绘制窗口，并将来自鼠标、键盘等设备的输入传递给client。
 因此 X server和client可以位于同一计算机上，例如在Linux主机上使用KDE等桌面环境就是这种模式。X server也可以通过同构网络、异构网络或Internet与client通信。
 X server与client之间的通信是不加密的，这个问题可以通过SSH解决。SSH是Secure Shell的简称，SSH可以看作是通信被加密压缩版的telnet。
 需要用到SSH的forwarding功能，当X server与client所在计算机都支持SSH协议时，X server与client之间不安全的TCP/IP连接可以转送到（forwarding）二者之间建立的SSH连接上。

了解原理后，我们就可以在本地自建X服务，然后服务器作为X client，把绘图的请求发给本地的X server。 这样就实现了本地显示图像的目的。

## 二、使用教程

1. 配置CentOS的sshd
    修改sshd配置文件：`/etc/ssh/sshd_config`            
    找到如下配置信息并去掉前面注释：

   ```shell
   AllowTcpForwarding yes
   ##启用X11 Forwarding
   X11Forwarding yes
   X11UseLocalhost no
   ```

   检查下linux服务器上是否安装了xorg-x11-xauth，如果没有则需要先安装xauth，命令如下：

   ```shell
   yum install xorg-x11-xauth
   ```

   安装所需软件包：

   ```shell
   yum install -y xorg-x11-xauth           #安装x11组件包        
   yum -y install wqy-zenhei-fonts*        #安装中文字库      
   yum -y install ibus-libpinyin*          #安装中文输入法
   ```

   重启sshd服务

   ```shell
   service sshd restart
   ```

2. 安装配置Xming
    下载并安装Xming或Xmanager，地址：

   - Ximing: <https://xming.en.softonic.com/>
   - Xmanager:<https://www.netsarang.com/zh/xmanager/>

3.  然后运行`XLaunch.exe`，若不知道具体参数保持默认下一步即可。

4. 客户端配置
    使用SSH客户端登陆CentOS，建议使用Xshell或Putty。
    然后在**SSH-X11转发**中开启X11转发，然后在命令行运行带GUI的应用程序即可。

5. 报错解决
    若登陆提示`The remote SSH server rejected X11 forwarding request.`      
    那么运行以下命令：

   ```shell
   yum install -y xorg-x11-xauth xorg-x11-utils xorg-x11-fonts-*
   ```

   说明：使用X11 Forwarding需要安装rpm包`xorg-x11-xauth`，如果你在安装CentOS系统时，选择了安装X Window System，那这个包是默认安装的。

