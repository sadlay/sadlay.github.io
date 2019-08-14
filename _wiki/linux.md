---
layout: wiki
title: Linux/Unix
categories: Linux
description: 类 Unix 系统下的一些常用命令和用法。
keywords: Linux
---

类 Unix 系统下的一些常用命令和用法。

## 实用命令

### fuser

查看文件被谁占用。

```sh
fuser -u .linux.md.swp
```

### id

查看当前用户、组 id。

### lsof

查看打开的文件列表。

> An  open  file  may  be  a  regular  file,  a directory, a block special file, a character special file, an executing text reference, a library, a stream or a network file (Internet socket, NFS file or UNIX domain socket.)  A specific file or all the files in a file system may be selected by path.

#### 查看网络相关的文件占用

```sh
lsof -i
```

#### 查看端口占用

```sh
lsof -i tcp:5037
```

#### 查看某个文件被谁占用

```sh
lsof .linux.md.swp
```

#### 查看某个用户占用的文件信息

```sh
lsof -u mazhuang
```

`-u` 后面可以跟 uid 或 login name。

#### 查看某个程序占用的文件信息

```sh
lsof -c Vim
```

注意程序名区分大小写。



## 常用分类

### 进程

```shell
# 查看进程详细信息
ps 1777
```

### 端口

```shell
netstat -tlunp #显示出当前主机打开的所有端口
netstat -ntlp #查看当前监听所有端口
netstat -nap #会列出所有正在使用的端口及关联的进程/应用


lsof -i :portnumber #portnumber要用具体的端口号代替，可以直接列出该端口听使用进程/应用
lsof -i tcp:portnumber #同上
netstat -lnp|grep 88 #同上

firewall-cmd --list-ports #查看已经开放的端口

lsof -i|grep pid #查看当前进程tcp连接端口
```

### 防火墙

#### CentOS6

```SHELL
service iptable status #查看防火墙的状态
servcie iptables stop  #临时关闭防火墙
chkconfig iptables off #永久关闭防火墙

#如要开放80，22，8080 端口，输入以下命令即可
/sbin/iptables -I INPUT -p tcp --dport 80 -j ACCEPT
/sbin/iptables -I INPUT -p tcp --dport 22 -j ACCEPT
/sbin/iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
# 保存
/etc/rc.d/init.d/iptables save
#查看打开的端口
/etc/init.d/iptables status

#关闭防火墙
#1） 永久性生效，重启后不会复原
#开启： 
chkconfig iptables on
#关闭： 
chkconfig iptables off
#2）即时生效，重启后复原
#开启： 
service iptables start
#关闭： 
service iptables stop
#查看防火墙状态： 
service iptables status 
```

#### CentOS7

```shell
firewall-cmd --state #查看防火墙的状态

#从centos7开始使用systemctl来管理服务和程序，包括了service和chkconfig
systemctl list-unit-files|grep firewalld.service #防火墙服务状态
systemctl status firewalld.service #防火墙服务运行状态

systemctl stop firewalld.service #停止firewall
systemctl start firewalld.service #启动一个服务
systemctl restart firewalld.service #重启一个服务
systemctl disable firewalld.service #禁止firewall开机启动
systemctl enable firewalld.service #在开机时启用一个服务
systemctl is-enabled firewalld.service;echo $? #查看服务是否开机启动
systemctl list-unit-files|grep enabled #查看已启动的服务列表

iptables-save #查看端口
firewall-cmd --list-ports #查看已经开放的端口
firewall-cmd --zone=public --add-port=80/tcp --permanent #开启端口
#命令含义：
–zone #作用域
–add-port=80/tcp #添加端口，格式为：端口/通讯协议
–permanent #永久生效，没有此参数重启后失效

firewall-cmd --reload #重启firewall
firewall-cmd --state #查看默认防火墙状态（关闭后显示notrunning，开启后显示running）
```

#### 区别

CentOS 7默认使用的是firewall作为防火墙，使用iptables必须重新设置一下

1、直接关闭防火墙

```shell
`systemctl stop firewalld.service ``#停止firewall` `systemctl disable firewalld.service ``#禁止firewall开机启动`
```

2、设置 iptables service

```shell
`yum -y ``install` `iptables-services`
```

如果要修改防火墙配置，如增加防火墙端口3306

```shell
`vi` `/etc/sysconfig/iptables`
```

增加规则

```shell
`-A INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j ACCEPT`
```

保存退出后

```shell
`systemctl restart iptables.service ``#重启防火墙使配置生效` `systemctl ``enable` `iptables.service ``#设置防火墙开机启动`
```

最后重启系统使设置生效即可。

```shell
`systemctl start iptables.service ``#打开防火墙` `systemctl stop iptables.service ``#关闭防火墙`
```

### 安全

#### nmap

```shell
rpm -qa | grep nmap #查看是否安装nmap
yum install nmap #yum安装nmap
nmap ip_address #扫描的主机IP地址对外暴露的端口
```



