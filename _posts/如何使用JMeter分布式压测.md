---
title: 如何使用JMeter分布式压测
date: 2019-07-03 14:40:00
tags:
- JMeter
- 压力测试
- 分布式压测
categories:
- 测试
---

![](/images/distributed-jmeter.png)

施加压力的机器被称为压力源，如果没有一个稳定且强大的源将严重阻碍压测工作的开展，压测结果得不到保证，无法真正达到压测目的，也压不出系统的性能瓶颈。单机的能力是受限的，达到瓶颈后就无能为力了，所以能不能把多台机器有机的组合起来，形成一个分布式压力源呢？答案是肯定的，这也是JMeter另一个强大的功能。

<!-- more -->
# 方案

![](/images/distributed-names.png)

* Master：控制单元，通常以GUI界面方式运行。
* Slaves：压力源，它从Master处获取命令并将请求发送到目标系统，一般由多台机器组成。
* Target：被测系统

# 部署

从上面方案可以看出，需要规划好哪些机器用来充当压力源，哪台机器用来作为控制单元。

这里假设有三台CentOS7和一台个人PC：

1. 在三台CentOS7机器上安装上JMeter，以这种方式启动 `./bin/jmeter-server -Jserver.rmi.ssl.disable=true -Djava.rmi.server.hostname=192.168.1.100`，hostname应该填写你自己的真实IP。
1. 个人PC作为控制台的主系统，在文件 `bin/jmeter.properties` 中的 `remote_hosts` 属性上修改 `remote_hosts=192.168.1.101:1099,192.168.1.102:1099,192.168.1.103:1099`，也就是把你自己的这三台机器ip加进去，这样才能组成一个分布式压力源。另外还需要禁用SSL，在相同的文件中找到属性 `server.rmi.ssl.disable` 修改成 `server.rmi.ssl.disable=true`。最后以GUI模式启动JMeter。

在控制侧JMeter的GUI界面上，通常菜单Run => Remote start可以看到添加进来的Slaves。

# 运行

1. 加载已有的测试计划
1. 通常菜单Run => Remote start All

---
参考文献：
1. https://jmeter.apache.org/usermanual/jmeter_distributed_testing_step_by_step.html