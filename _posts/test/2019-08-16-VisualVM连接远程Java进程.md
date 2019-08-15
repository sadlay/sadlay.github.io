---
layout: post
title: VisualVM连接远程Java进程
categories: Test
description: VisualVM 通过检测 JVM 中加载的类和对象信息等帮助我们分析内存使用情况，我们可以通过 VisualVM 的监视标签对应用程序进行内存分析。
keywords: VisualVM, Java, Analyzer 
---

# VisualVM连接远程Java进程

## JMX方式

### 配置脚本文件

在`$CATALINA_HOME/bin/startup.sh` 倒数第二行（也就是`exec "$PRGDIR"/"$EXECUTABLE" start "$@"`一行上边）写入下面的内容：

```shell
export CATALINA_OPTS="$CATALINA_OPTS
-Dcom.sun.management.jmxremote
-Djava.rmi.server.hostname=*.*.*.* YOUR SERVER IP
-Dcom.sun.management.jmxremote.port=7003
-Dcom.sun.management.jmxremote.ssl=false
-Dcom.sun.management.jmxremote.authenticate=true
-Dcom.sun.management.jmxremote.password.file=../conf/jmxremote.password
-Dcom.sun.management.jmxremote.access.file=../conf/jmxremote.access"
```

或者编辑Tomact里bin目录的catalina.sh . 在其头部加入（推荐）

```shell
JAVA_OPTS="
   -Dcom.sun.management.jmxremote
   -Dcom.sun.management.jmxremote.port=8998  这个端口可以改
   -Dcom.sun.management.jmxremote.rmi.port=8998
   -Dcom.sun.management.jmxremote.ssl=false
   -Dcom.sun.management.jmxremote.authenticate=ture 需要鉴权 若为false则不需要下两行的配置
   -Dcom.sun.management.jmxremote.password.file=../conf/jmxremote.password
-Dcom.sun.management.jmxremote.access.file=../conf/jmxremote.access
-Djava.rmi.server.hostname=xxx.xxx.xxx.xxx" 服务器ip地址 
```

参数说明

```shell
-Dcom.sun.management.jmxremote 启用JMX远程监控
-Djava.rmi.server.hostname=*.*.*.* 你的tomcat服务器IP地址
-Dcom.sun.management.jmxremote.port=7003  jmx连接端口
-Dcom.sun.management.jmxremote.ssl=false  是否ssl加密
-Dcom.sun.management.jmxremote.authenticate=true  远程连接需要密码认证
-Dcom.sun.management.jmxremote.password.file=../conf/jmxremote.password  指定连接的用户名和密码配置文件
-Dcom.sun.management.jmxremote.access.file=../conf/jmxremote.access  指定连接的用户所拥有权限的配置文件
```

编辑Tomcat里conf目录的server.xml 加入监听器:

```xml
<Listener className="org.apache.catalina.mbeans.JmxRemoteLifecycleListener" rmiRegistryPortPlatform="8998" rmiServerPortPlatform="8999" />

```

- 其中8998可以改为你想要的端口

### 创建账号密码和权限配置文件

#### jmxremote.access 

进入在 `$CATALINA_HOME/conf/` 目录下

创建`touch jmxremote.access`里添加可以连接监控的用户名以及权限： 
文件内容如下：

```shell
monitorRole readonly
controlRole readwrite
```

#### jmxremote.password

`touch jmxremote.password` 创建存储账号密码的文件 
写入下面内容：

```shell
monitorRole 111111
controlRole 222222
```

### 修改访问权限

修改jmxremote.access和jmxremote.password的权限：

```shell
sudo chmod 600 jmx*
```

然后， 开放端口，重启Tomcat使之生效。

## jstatd方式

### 新建配置文件

使用cd $JAVA_HOME 到jdk的根目录，bin文件夹里面新建jstatd.all.policy文件。内容如下:

```shell
grant codebase "file:${java.home}/../lib/tools.jar" { permission java.security.AllPermission; };
```

### 启动配置文件

在服务器上jstatd.all.policy所在目录下执行下面的命令 
其中 /usr/local/java/bin/jstatd为jstatd所在路径，为${JAVA_HOME}/bin/jstatd，在bin目录下启动Jstatd。

```shell
./jstatd -J-Djava.security.policy=jstatd.all.policy -J-Djava.rmi.server.hostname=192.168.0.23
```

hostname为外网ip。

默认使用1099端口，如果想指定端口使用如下命令：

```shell
jstatd -J-Djava.security.policy=jstatd.policy -p 1099
```

### jstatd 的安全问题

jstatd服务只能监视具有适当的本地访问权限的JVM，因此jstatd进程与被监控的JVM必须运行在相同的用户权限中。但是有一些特殊的用户权限，如基于UNIX（TM）为系统的root用户，它有权限访问系统中所有JVM的资源，如果jstatd进程运行在这种权限中，那么它可以监视系统中的所有JVM，但是这也带来了额外的安全问题。

jstatd服务不会对客户端进行任何的验证，因此运行了jstatd服务的JVMs，网络上的任何用户的都具有访问权限，这种暴露不是我们所希望的，因此在启动jstatd之前本地安全策略必须要加以考虑，特别是在生产环境中或者是在不安全的网络环境中。

如果没有其他安全管理器被安装，jstatd服务将会安装一个RMISecurityPolicy的实例，因此需要在一个安全策略文件中指定，该策略文件必须符合的默认策略实施的策略文件语法。

下面的这个示例策略将允许jstatd服务具有JVM全部的访问权限：

```shell
grant codebase "file:${java.home}/../lib/tools.jar" {  
   permission java.security.AllPermission;  
}; 
```

注：此处策略中的java.home，和JAVA_HOME不是一个概念，不要搞错了，此处的java.home指的是JRE的路径，这个是Java的系统属性，不需要手工指定，通常是这个jdk下面的jre路径,即可以认为${java.home}和${JAVA_HOME}/jre是等价，如果想查看这个变量的值，可以任意找一个运行着的Java应用，找到它的PID，然后通过如下jinfo命令查看就可以查看到java.home的值：

```shell
jinfo ${PID}|grep java.home
```

## 其他

### 无法访问

这个时候理论上可以开启visualVM然后添加远程主机监控了，但是由于JMX和jstatd还需要监听一到两个随机端口。会有报错信息：**无法使用service:jmx:rmi连接**，通常解决方法有两种：

-   是用 jps 得到pid，然后使用 lsof -i|grep {pid} 命令得到监听的其他端口然后将在iptables开放。  
- 关闭防火墙

### Jar包监控

执行jar启动命令

```shell
java -Dcom.sun.management.jmxremote.port=12345 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -jar foo.jar
```

说明

- 12345为需要监控的端口，远程机器需要开启

- foo.jar为程序名称

出现VisualVM 无法使用 service:jmx:rmi:///jndi/rmi:///jmxrmi 连接情况，关闭远程机器的防火墙即可

原因是除了JMX server指定的监听端口号外，JMXserver还会监听一到两个随机端口号，
可以通过命grep <pid> 来查看当前java进程需要监听的随机端口号

远程连接启动authenticate、ssl参数

以authenticate设置为例

1. jmx连接使用安全凭证，这里的凭证不是linux的登录账号密码，需要单独设置

2. jmxremote.access内容

   ```shell
   admin readwrite
   ```

3. jmxremote.password内容

   ```shell
   admin 123456
   ```

4. 两个文件授权（必须按下面方式授权，chmod777都不行）

   ```shell
   chmod 600 jmxremote.access
   chmod 600 jmxremote.password
   
   chown root:root jmxremote.access
   chown root:root jmxremote.password
   ```

5. jar程序启动命令

   ```shell
   java -Dcom.sun.management.jmxremote.port=12345 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=true -Dcom.sun.management.jmxremote.authenticate=true -Dcom.sun.management.jmxremote.access.file=/usr/local/jmxremote.access -Dcom.sun.management.jmxremote.password.file=/usr/local/jmxremote.password -jar foo.jar
   ```

   


  