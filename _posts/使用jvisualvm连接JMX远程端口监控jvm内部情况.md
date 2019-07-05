---
title: 使用jvisualvm连接JMX远程端口监控jvm内部情况
date: 2019-07-05 10:36:00
tags:
- jvisualvm
- jmx
- 远程调试
categories:
- 监控
- java
---

![](/images/jvisualvm.jpg)

对于应用程序的监控是必要的，特别是在生产环境上，我们应该始终能够提前发现潜在的问题，并在问题产生后要第一时间就知道。在真实的业务运营中监控是多维度的，几乎涵盖了软硬件以及各项业务指标。就java应用程序本身来讲，我们最关注的是jvm的内部情况，比如CPU和堆内存使用率、线程状态和垃圾回收等情况。

那如何得到这些监控数据呢？
好在JDK里面自带了一些工具帮我们分析这些数据并呈现报告，接下来介绍**Java VisualVM**。
<!-- more -->

# 对于本地java程序

1. 确保被监控程序已经启动
1. 在JDK安装目录的 `bin` 子目录找到 `jvisualvm.exe` 并双击打开。
1. 在打开后的界面左边菜单可以看到**本地**根节点，继续展开后可以看到本机上所运行的java程序，选择自己要查看的那个应用并双击。

![](/images/visualvm_local.gif)

# 对于远程java程序

如果你要监控的程序不在本地，而是处于其它机器上，那远端的监控数据肯定是要通过网络传输过来的，所以相对于本地应用多了一步，即建立数据传输通道。

## 开启JMX端口

在java应用启动时添加开启JMX端口的参数，其中 `<hostname>` 填写远程机器的外网IP：

```bash
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=1099 \
     -Dcom.sun.management.jmxremote.rmi.port=1099 \
     -Dcom.sun.management.jmxremote.authenticate=false \
     -Dcom.sun.management.jmxremote.ssl=false \
     -Dcom.sun.management.jmxremote.local.only=false \
     -Djava.rmi.server.hostname=<hostname> \
     -jar test-visualvm-1.0-SNAPSHOT.jar
```

执行完上面的命令后检查端口 `1099` 是否已经开启：

```bash
[root@localhost ~]# netstat -ntpl|grep 1099
tcp6       0      0 :::1099                 :::*                    LISTEN      6902/java
```

## 添加远程主机

1. 在打开后的**Java VisulaVM**界面左边菜单可以看到**远程**根节点，右击选择**添加远程主机**，下面用动画演示。

![](/images/visualvm_remote.gif)


# 其它更多

## 插件

1. 可以在菜单栏**工具**=>**插件**=>**可用插件**中来安装更多监控功能。

## 工具使用心得

当程序出现问题情况会很多，要综合各种数据来判断。

1. 观察cpu使用率情况，如果打满肯定会造成程序卡顿响应非常慢。
1. 堆内存过大甚至超过最大值会OOM造成程序挂掉，此时可以根据实际情况进行调整。
1. 线程等待过多是否CPU、内存、带宽或磁盘IO瓶颈造成，同时也要检查程序设计是否合理。可以查看每个线程的调用栈情况，也可以通过安装插件检查是否发生死锁，每个线程到底锁了什么资源。
1. 安装插件后可以观察每个内存各个区的详细情况，查看是否有哪个区块遇到了空间不足的问题，通过jvm参数进行相应调整。

## 后话

推荐Alibaba开源的Java诊断工具[Arthas](https://alibaba.github.io/arthas/index.html)，可以结合多种工具来综合分析，尽早定位到问题。

当你遇到以下类似问题而束手无策时，Arthas可以帮助你解决：

1. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
1. 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
1. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
1. 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
1. 是否有一个全局视角来查看系统的运行状况？
1. 有什么办法可以监控到JVM的实时运行状态？

Arthas支持JDK 6+，支持Linux/Mac/Winodws，采用命令行交互模式，同时提供丰富的 Tab 自动补全功能，进一步方便进行问题的定位和诊断。

---
参考文献：
1. https://metabase.com/docs/latest/operations-guide/enable-jmx.html
1. https://docs.oracle.com/javase/tutorial/jmx/overview/index.html