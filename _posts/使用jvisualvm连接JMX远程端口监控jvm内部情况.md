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

1. 可以在菜单栏**工具**=>**插件**=>**可用插件**中来安装更多监控功能。

---
参考文献：
1. https://metabase.com/docs/latest/operations-guide/enable-jmx.html
1. https://docs.oracle.com/javase/tutorial/jmx/overview/index.html