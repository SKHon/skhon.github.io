---
title: jmeter压测
date: 2020-02-25 03:25:24
categories: 前端
tags: [node]
comments: true
copyright: true
---

## 背景

这两天我负责的一个 node 中间层框架 hobber，在业务方的那边的项目压测中，当 qps 达到 1500 左右时，出现大量 504。由于压测线上服务在半夜，当时我也不在现场，所以，我决定自己压一下，目的有两个，一个是压我们自己的服<!--more-->务，看 hobber 是不是能够压到当初承诺的 qps 2500，另一个是想复现一下那天业务方压测出现的 504 问题。

## 步骤

### 下载

我用的是 jmeter，[下载地址](http://jmeter.apache.org/download_jmeter.cgi)

## 启动

进入 jmeter 目录下对 bin 文件，执行命令：sh jmeter

### 脚本

jmeter 压测的时候，一般不用图像化工具压，一般是执行脚本压，如果图像化工具直接压，太耗性能了，所以机器对资源尽量留给发压力。
但是脚本的话，一般是通过图形化界面配，然后把脚本导出来。

1. 创建线程组
   在“测试计划”上右键 【添加】-->【Threads(Users)】-->【线程组】。
   ![](/images/jmeter/1.png)

设置线程数和循环次数。我这里设置线程数为 500，循环一次。
![](/images/jmeter/2.png)

2. 配置元件
   在我们刚刚创建的线程组上右键 【添加】-->【配置元件】-->【HTTP 请求默认值】。
   ![](/images/jmeter/3.png)
   配置我们需要进行测试的程序协议、地址和端口:
   ![](/images/jmeter/4.png)
   `当所有的接口测试的访问域名和端口都一样时，可以使用该元件，一旦服务器地址变更，只需要修改请求默认值即可。`
3. 构造 HTTP 请求
   在“线程组”右键 【添加-】->【samlper】-->【HTTP 请求】设置我们需要测试的 API 的请求路径和数据。我这里是用的 json
   ![](/images/jmeter/5.png)

4. 添加 HTTP 请求头
   在我们刚刚创建的线程组上右键 【添加】-->【配置元件】-->【HTTP 信息头管理器】。

因为我要传输的数据为 json，所以设置一个 Content-Type:application/json
![](/images/jmeter/6.png)

5. 添加断言
   在我们刚刚创建的线程组上右键 【添加】-->【断言】-->【响应断言】。

根据响应的数据来判断请求是否正常。我在这里只判断的响应代码是否为 200。还可以配置错误信息
![](/images/jmeter/7.png)

6. 添加察看结果树
   在我们刚刚创建的线程组上右键 【添加】-->【监听器】-->【察看结果树】。

直接添加，然后点击运行按钮就可以看到结果了。
![](/images/jmeter/8.png)

7. 添加 Summary Report
   在我们刚刚创建的线程组上右键 【添加】-->【监听器】-->【Summary Report】。

直接添加，然后点击运行按钮就可以看到结果了。
![](/images/jmeter/9.png)

8. 测试计划创建完成
   文件 -> 保存测试计划

## 执行测试计划

前面我们说过，执行测试计划不能用 GUI，需要用命令行来执行。
![](/images/jmeter/10.png)

我这里执行的命令为：

jmeter -n -t testplan/RedisLock.jmx -l testplan/result/result.txt -e -o testplan/webreport

说明：
testplan/RedisLock.jmx 为测试计划文件路径
testplan/result/result.txt 为测试结果文件路径
testplan/webreport 为 web 报告保存路径。

## 问题

如果发压机端口不够用，可能会出现这样的问题：
<font color=#DC143C face="黑体">Non HTTP response code: java.net.NoRouteToHostException/Non HTTP response message: Cannot assign requested address (Address not available)</font>

解决办法：

1. 调低端口释放后的等待时间， 默认为 60s， 修改为 15~30s
   echo 30 > /proc/sys/net/ipv4/tcp_fin_timeout
2. 修改 tcp/ip 协议配置， 通过配置/proc/sys/net/ipv4/tcp_tw_resue, 默认为 0， 修改为 1， 释放 TIME_WAIT 端口给新连接使用。
   echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse
3. 修改 tcp/ip 协议配置，快速回收 socket 资源， 默认为 0， 修改为 1.
   echo 1 > /proc/sys/net/ipv4/tcp_tw_recycle
4. 执行：sysctl -p ，使设置立即生效。
