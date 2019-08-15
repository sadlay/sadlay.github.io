---
layout: post
title: ApacheBench简介
categories: Test
description: ApacheBench 是一个用来衡量http服务器性能的单线程命令行工具。原本针对Apache http服务器，但是也适用于其他http服务器。
keywords: ApacheBench, AB, HTTP 
---

# ApacheBench简介

ApacheBench 是一个用来衡量http服务器性能的单线程命令行工具。原本针对Apache http服务器，但是也适用于其他http服务器。

ab工具与标准 Apache源码一起发布，免费，开源，基于[Apache License](http://en.wikipedia.org/wiki/Apache_License)。

这个工具返回的最有用的信息就是服务器每秒能够处理的请求次数（RPS），不过由于测试的页面不同，RPS相差会很大，静态页面的RPS大于动态页面，页面体积越小，RPS越大。所以，RPS是相对的，在选择主机的时候，可以使用同一个页面进行测试，这样得到的数据相对来说更有可比性。

Tips:在带宽不足的情况下，最好是本机进行测试，建议使用内网的另一台或者多台服务器通过内网进行测试，这样得出的数据，准确度会高很多。远程对web服务器进行压力测试，往往效果不理想（因为网络延时过大或带宽不足）

## 安装

Apache本身会自带ab，如果没有安装Apache，以下方法可以用来便捷的安装ab工具：

- Ubuntu

  `apt-cache install apache2-util`

- CentOS

  `yum install httpd-tools`

- MacOS
   系统自带apache，查看版本信息：

- windows

  windows的话直接在[官网下载](https://www.apachelounge.com/download/)，cd进入bin目录就可以执行命令

Linux安装完成在已添加环境变量的情况下，可以直接使用`ab -v`命令价差是否安装成功。

```shell
$ ab -V
This is ApacheBench, Version 2.3 <$Revision: 1826891 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/
```

## 使用

### GET

对于模拟GET请求进行测试，ab非常简单，就是：ab -n 100 -c 10 'http://testurl.com/xxxx?para1=aaa&para2=bbb'

### POST

对于模拟POST请求进行测试，则稍微复杂些，需要把将要post的数据（一般是json格式）放在文件里。比如建立一个文件post_data.txt，放入：

```json
{"actionType":"collect","appId":1,"contentId":"1770730744","contentType":"musictrack","did":"866479025346031","endType":"mobile","recommendId":"104169490_1_0_1434453099#1770730744#musictrack#USER_TO_SONG_TO_SONGS#gsql_similarity_content2content","tabId":0,"uid":"104169490"}
```

然后用-p参数解析并发送这个json数据：

```shell
ab -n 100 -c 10 -p post_data.txt -T 'application/json' http://testurl.com/xxxx'
```

### 基本命令

```shell
 ab -h
Options are:
 -V              显示版本号并退出
 -n requests     总请求数,默认值为1
 -c concurrency  并发请求数，默认值为1,其值不能超过总请求数。建议是可以被总请求数整除，如 -n 100 –c 10模拟10个并发，每个并发10个请求。
 -t timelimit    测试所用的时间上限（单位为秒），它可以使对服务器的测试限制在一个固定的总时间以内，其内部隐含的默认值等于 -n 50000
 -s timeout      每次响应的最大等待时间（单位为秒），默认值为30s.注意s是小写的
 -p postfile     用于存放发起post请求时的数据的文件地址，不要忘记设置请求头。文件内容的格式要视请求头决定。
 -T content-type POST请求所使用的Content-type头信息，如 -T “application/x-www-form-urlencoded” ，默认值为“text/plain”
 -H attribute  添加头部Header，需要注意custome-header的格式为 key:value，eg. User-Agent: ApacheBench/2.3', 这也是ab的默认值User-Agent
 -C attribute  对请求附加Cookie,格式为 key=value ,注意和-H区分！
 -m method    设置请求方法
 -v verbosity  显示调试信息，verbosity为数字，可以看做是输出信息的级别。3或更大值可以显示响应代码(404, 200等), 2或更大值可以显示警告和其他信息。
 -X proxy:port  设置代理地址以及端口信息
 -w             以HTML表的格式输出结果。
 -r          抛出异常继续执行测试任务 
 -i             使用HEAD请求代替GET
```

### 语法

```shell
ab [ -A auth-username:password ] [ -b windowsize ] [ -c concurrency ] [ -C cookie-name=value ] [ -d ] [ -e csv-file ] [ -f protocol ] [ -g gnuplot-file ] [ -h ] [ -H custom-header ] [ -i ] [ -k ] [ -n requests ] [ -p POST-file ] [ -P proxy-auth-username:password ] [ -q ] [ -r ] [ -s ] [ -S ] [ -t timelimit ] [ -T content-type ] [ -u PUT-file ] [ -v verbosity] [ -V ] [ -w ] [ -x <table>-attributes ] [ -X proxy[:port] ] [ -y <tr>-attributes ] [ -z <td>-attributes ] [ -Z ciphersuite ] [http[s]://]hostname[:port]/path
```

### 详细命令选项

```shell
-A auth-username:password    向服务器提供基本认证信息。用户名和密码之间":"分割，以base64编码形式发送。无论服务器是否需要(即是否发送了401)都发送。 

-b windowsize    TCP发送/接收缓冲区大小，以字节为单位。

-c concurrency    并发数，默认为1。

-C cookie-name=value    添加Cookie。典型形式是name=value对。name参数可以重复。 

-d不显示"percentage served within XX [ms] table"消息(兼容以前的版本)。 

-e csv-file    输出百分率和对应的时间，格式为逗号份额的csv。由于这种格式已经"二进制化"，所以比"gnuplot"格式更有用。

-f protocol    SSL/TLS protocol (SSL2, SSL3, TLS1, 或ALL).

-g gnuplot-file    把所有测试结果写入"gnuplot"或者TSV(以Tab分隔)文件。该文件可以方便地导入到Gnuplot, IDL, Mathematica甚至Excel中，第一行为标题。

-h    显示使用方法。

-H custom-header    附加额外头信息。典型形式有效的头信息行，包含冒号分隔的字段和值(如："Accept-Encoding: zip/zop;8bit")。

-i    执行HEAD请求，而不是GET 。

-k    启用KeepAlive功能，即在HTTP会话中执行多个请求。默认关闭。

-n requests    会话执行的请求数。默认为1。 

-p POST-file    附加包含POST数据的文件。注意和-T一起使用。

-P proxy-auth-username:password    代理认证。用户名和密码之间":"分割，以base64编码形式发送。无论服务器是否需要(即是否发送了407)都发送。

-q    quiet，静默模式。不在stderr输出进度条。

-r    套接字接收错误时不退出。

-s timeout     超时，默认为30秒。

-S    不显示中值和标准偏差值，而且在均值和中值为标准偏差值的1到2倍时，也不显示警告或出错信息。默认显示最小值/均值/最大值。(兼容以前的版本)-t timelimit
    测试进行的最大秒数。内部隐含值是"-n 50000"。默认没有时间限制。

-T content-type    POST/PUT的"Content-type"头信息。比如“application/x-www-form-urlencoded”，默认“text/plain”。

-v verbosity    详细模式，4以上会显示头信息，3以上显示响应代码(404，200等)，2以上显示告警和info。

-V    显示版本号并退出。

-w    以HTML表格形式输出。默认是白色背景的两列。

-x <table>-attributes    设置<table>属性。此属性填入<table 这里 > 。

-X proxy[:port]    使用代理服务器。

-y <tr>-attributes    设置<tr>属性。

-z <td>-attributes    设置<td>属性。 

-Z ciphersuite    设置SSL/TLS加密
```

ab一般常用参数就是 -n， -t ，和 -c。

```shell
-c（concurrency）表示用多少并发来进行测试；

-t表示测试持续多长时间，单位是秒；

-n表示要发送多少次测试请求。
```

一般-t或者-n选一个用。

下面是一些例子：

```shell
> ab -n 10 -c 10 http://cnblogs.com/
​
# Get请求，要注意“&”是ab的保留运算符，使用时要用双引号引起来
> ab -n 10 -c 10 http://httpbin.org/get?name=rethink"&"age=3
​
# Post请求，post_data文件中的内容要和所指定的Content-Type对应（默认text/plain）
> ab -n 10 -c 10 -p E:\post_data1.txt -T application/x-www-form-urlencoded http://httpbin.org/post
> ab -n 10 -c 10 -p E:\post_data2.txt -T application/json http://httpbin.org/post
​
# post_data1中的数据为：name=rethink&age=3&method=post
# post_data1中的数据为：{"name":"Rethink","method":"post","age":3}
​
# 使用`-w`参数将结果以HTML的格式输出，然后将结果写入到本地文件中：
>  ab -n 2 -c 2 -v 2 -w  http://httpbin.org/get?name=rethink"&"age=3 > E:\report.html
```

注意事项：

1. result.html中会打印每次请求的请求头信息，请求总数较大时，重定向结果输出时可以不指定`-v`参数；
2. 使用`-H Content-Type:application/json`不能代替`-T "application/json"`， 使用前者服务器可能会返回400 bad requests;
3. 如果提示`ab: invalid URL`，可能是URL最右边缺少/，例如 *http://www.example.com* 需要改为*http://www.example.com/* ；
4. 不支持发送https请求；
5. postfile注意使用正确的编码格式，否则请求参数服务器端可能无法识别；
6. 调试请求时，对接口返回的中文字符的支持不友好，会显示乱码；

### 结果分析

从上面可以看到ab支持参数很多，但一般来说只有`-c` 和`-n` 参数是必要的，例如:

```shell
>PS > ab -n 10 -c 10  http://httpbin.org/get?name=rethink"&"age=3
This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/
​
Benchmarking cnblogs.com (be patient).....done
​
Server Software:  gunicorn/19.9.0 服务器类型
Server Hostname:        httpbin.org  域名
Server Port:            80  web端口
​
Document Path:          /get?name=rethink&age=3  测试的页面路径
Document Length:        280 bytes  测试的页面大小
​
Concurrency Level:      10  测试的并发数
Time taken for tests:   2.629 seconds  整个测试活动持续的时间
Complete requests:      10  完成的请求数量(只包含状态码==200的请求)
Failed requests:        0  失败的请求数量（包含状态码!=200的请求和超时的请求）
Non-2xx responses:      10     状态码不等于2xx的响应数
Total transferred:      5210 bytes  整个过程中的网络传输量
HTML transferred:       2800 bytes  整个过程中的HTML内容传输量
Requests per second:    3.80 [/sec] (mean)  每秒的响应请求数（QPS），数值等于 请求总数(-n)/Time taken for tests，后面括号中的mean表示这是一个平均值
Time per request:       2629.486 [ms] (mean)    用户平均请求等待时间=concurrency * timetaken * 1000 / done
Time per request:       12.893 [ms] (mean, across all concurrent requests)  每个连接请求实际运行时间的平均值
Transfer rate:          25.15 [Kbytes/sec] received  传输速率
​
Connection Times (ms)
 min  mean[+/-sd] median   max
Connect:      210  216   6.1    216     229
Processing:   229 1022 658.7   1081    2181
Waiting:      229  827 558.2    866    1731
Total:        445 1238 659.4   1291    2399

​
Percentage of the requests served within a certain time (ms)
 50%   1291  整体响应时间的分布比 
 66%   1507
 75%   1731
 80%   1949
 90%   2399  表示90%的请求在1291ms内得到响应
 95%   2399
 98%   2399
 99%   2399
 100%   2399 (longest request)
 
整个场景中所有请求的响应情况。在场景中每个请求都有一个响应时间 
其中 50％ 的用户响应时间小于 1291 毫秒 
80 ％ 的用户响应时间小于 1949 毫秒 
最大的响应时间小于 2399 毫秒
```

### 分析字段

```shell
Server Software
    返回的第一次成功的服务器响应的HTTP头。

Server Hostname
    命令行中给出的域名或IP地址

Server Port
    命令行中给出端口。如果没有80(HTTP)和443(HTTPS)。

SSL/TLS Protocol
    使用SSL打印。

Document Path
    命令行请求的路径。

Document Length
    第一次成功地返回文档的字节大小。后面接受的文档长度变化时，会认为是错误。

Concurrency Level
    并发数

Time taken for tests
    测试耗时

Complete requests
    收到成功响应数

Failed requests
    失败请求数。如果有会打印错误原因

Write errors
   写错误数 (broken pipe)Non-2xx responses
    非2**响应数量。如果有打印。

Keep-Alive requests
    Keep-Alive请求的连接数

Total body sent：
    传输的body的数据量，比如POST的数据。

Total transferred:    总传输数据量

HTML transferred:       
    累计html传输数据量

Time per request:      
    每批平均请求时间

Time per request: 
    每次平均请求时间。计算公式：Time per request/Concurrency Level。

Transfer rate
    数据传输速率。计算公式：otalread / 1024 / timetaken。
```

### 注意事项

1、不要一下子就把并发设置为100，这样的后果类似DDos。并发最大为1024，否则会出现“socket: Too many open files (24)”错误。

2、建议在本地（SSH登录到服务器上，测试在同一台服务器上的网站，或者测试同一个局域网中的网站）进行测试，这样会排除带宽带来的干扰。



详情参考[Apache Bench官方介绍](http://httpd.apache.org/docs/2.4/programs/ab.html)