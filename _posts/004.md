---
title: ActiveMQ消息队列主从方案
date: 2017-10-24 21:35:17
tags:
- ActiveMQ
- 消息队列
- 主从
categories:
- 中间件
---

# 背景

消息队列是实际项目中经常用到的中间件，目前也有很多开源并广泛应用的消息队列，今天拿ActiveMQ来聊一聊，怎样保证它的高可用。

# 现状

目前ActiveMQ提供了三种主从方案，分别是共享文件系统（Shared File System Master Slave）、数据库（JDBC Master Slave）和LevelDB（Replicated LevelDB Store）。需要注意的是LevelDB存储已被弃用，官方不再支持或建议使用，推荐使用KahaDB来代替。

* 第一种方式需要直接共享文件系统，在实际操作上比较少见。
* 而第三种方案已经被弃用
* 关于第二个方案的性能官方有一段描述：共享数据库不会太快，因为它不能使用高性能日志。但即便如此这也是目前最可靠、可行的一种方案。所以下面主要对基于JDBC的主从配置方式展开介绍。
<!-- more -->
# 方案

## 介绍

1. 在mq进程启动时一个主（某一台mq进程）抓取数据库中表的独占锁，另一台mq进程会由于争抢不到表锁而阻塞，这样其它的进程都变成了从。（只允许有一个主，可以有多个从）
  ![](/images/jdbc-master-slave-Startup.png)
  从上面的示意图上可以看到，三个Broker启动了但只有一个主，其它两个都是从。此时客户端1、2只能连接主。
1. 如果主的服务断开或者服务中断（比如进程中断、机器宕机等），它（原来的主）所占用的数据库表锁就会被释放，此时其他从开始竞争锁，谁先抢到谁就是新的主。
  ![](/images/jdbc-master-slave-MasterFailed.png)
  上面的示意图很好的说明了当主服务中断时，其它从服务竞争主的情况。</li>
1. 如果原来的主重新启动，但是已经有了新主，此时会怎么办呢？
  ![](/images/jdbc-master-slave-MasterRestarted.png)
  从上面的示意图不难看出，原来的主加回来之后变成了从，原因很简单，因为它争取不到锁（锁给新主了）。

## 客户端连接

通过上面的主从配置部署之后，可以保证只有一个服务是可用的，也就是说当主服务工作时，其它从服务都不会工作，如果你连接从服务会返回连接失败的错误。那么问题来了，客户端不确定哪个服务可用，哪个不可用，在连接时要“写死”连接吗？

* 目前有三种方案可以供大家来解决这个问题
  1. 第一种：在客户端连接mq服务的地址上来解决，我们可以使用含故障转移的连接地址，例如：
      ```java
      failover:(tcp://192.168.1.100:61616,tcp://192.168.1.101:61616,tcp://192.168.1.102:61616)
      ```
  1. 第二种：在项目代码上实现故障转移功能，用配置的方式维护一个服务列表，每次连接时从列表上逐一尝试，当前连接超时后就顺序尝试下一个，如果某个服务成功就保存下来，防止每次都走这个重试流程。
  1. 第三种：使用负载均衡软件（例如Keeplived、Nginx、LVS和HAProxy等），这样程序连接的地址是唯一的，故障转移由上述软件来完成即可。

# 安装

> 讲述在CentOS上安装、部署的方法

1. 从官网下载二进制包
  ```shell
  cd /usr/local/src
  wget http://mirrors.cnnic.cn/apache//activemq/5.14.3/apache-activemq-5.14.3-bin.tar.gz
  ```
1. 解压到任意目录
  ```shell
  tar -zxvf apache-activemq-5.14.3-bin.tar.gz
  ```
1. 启动mq
  ```shell
  cd apache-activemq-5.14.3
  ./bin/activemq start
  ```

* 此时安装就完成了，仅需要解压出来放到某个位置，没有实际的安装过程。（比如编译、安装等操作）
* 将这个过程在另外两台机器上操作一次，如果你是在一台机器上部署，那就把解压出来的文件再复制出来两个即可。（记得要修改配置文件中端口）
* 你可以在浏览器上访问mq的服务来判断是否都安装成功，访问地址 `http://192.168.1.100:8161` 你应该可以看到类似下面的WEB页面。
  ![](/images/activemq-web_console.png)

## 配置

* 修改web控制台凭证，重启服务后可以使用新的密码来登入。
  ```shell
    vim conf/jetty-realm.properties
    #将里面的内容清空，重新添加一行
    admin: Admin!123, admin

    vim conf/jetty.xml
    #修改第30行变成如下
    <property name="roles value="admin" />
  ```
* 服务的凭证，关闭不需要的协议
  ```shell
    vim conf/activemq.xml
    #定位到120行找到 `<transportConnectors>` 节点注释不需要开放的协议，仅保留下面两种协议即可。
    <transportConnectors>
        <transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnections=1000&wireFormat.maxFrameSize=104857600"/>
        <transportConnector name="stomp" uri="stomp://0.0.0.0:61613?maximumConnections=1000&wireFormat.maxFrameSize=104857600"/>
    </transportConnectors>
  ```
  为了保证安全，上面这两个端口可以配置为仅内网访问。（具体以实际项目情况来决定）
* 配置主从队列
  1. 修改配置文件
    ```shell
    vim conf/activemq.xml

    #在第40行修改 <borker> 节点的 brokerName 属性，新加入 persistent 属性。
    <broker xmlns="http://activemq.apache.org/schema/core" brokerName="127.0.0.1" persistent="true" dataDirectory="${activemq.data}">

    #在第81行 <persistenceAdapter> 节点下面配置如下代码
    <persistenceAdapter>
        <jdbcPersistenceAdapter dataDirectory="${activemq.base}/activemq-data" dataSource="#mysql-ds"/>
    </persistenceAdapter>

    #在第125行 <broker> 节点外面新增如下内容
    <!-- MySql DataSource Master Slave -->
    <bean id="mysql-ds" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="url value="jdbc:mysql://localhost/activemq?relaxAutoCommit=true"/>
        <property name="username" value="activemq"/>
        <property name="password" value="activemq"/>
        <property name="poolPreparedStatements" value="true"/>
    </bean>
    ```
  1. 安装java `yum -y install java-1.8.0-openjdk`
  1. 下载 [DBCP](http://central.maven.org/maven2/org/apache/commons/commons-dbcp2/2.1.1/commons-dbcp2-2.1.1.jar) 连接池包放到 `lib` 里面。
  1. 下载 [java-mysql-jdbc](http://central.maven.org/maven2/mysql/mysql-connector-java/5.1.38/mysql-connector-java-5.1.38.jar) 驱动包放到 `[activemq_install_dir]/lib` 里面。
  1. 创建数据库结构（表结构通常会在mq进程启动时自动创建，如果没有自动创建那你可以手动执行）
    ```sql
    CREATE DATABASE `activemq`;

    USE `activemq`;

    DROP TABLE IF EXISTS `activemq_acks`;

    CREATE TABLE `activemq_acks` (
      `LAST_ACKED_ID` int(11) DEFAULT NULL,
      `CONTAINER` varchar(250) DEFAULT NULL,
      `PRIORITY` bigint(20) NOT NULL DEFAULT '5',
      `XID` varchar(250) DEFAULT NULL,
      `CLIENT_ID` varchar(250) DEFAULT NULL,
      `SUB_DEST` varchar(250) DEFAULT NULL,
      `SUB_NAME` varchar(250) DEFAULT NULL,
      `SELECTOR` varchar(250) DEFAULT NULL,
      KEY `ACTIVEMQ_ACKS_XIDX` (`XID`)
    ) ENGINE=MyISAM DEFAULT CHARSET=utf8;

    DROP TABLE IF EXISTS `activemq_lock`;

    CREATE TABLE `activemq_lock` (
      `ID` bigint(20) NOT NULL,
      `TIME` bigint(20) DEFAULT NULL,
      `BROKER_NAME` varchar(250) DEFAULT NULL,
      PRIMARY KEY (`ID`)
    ) ENGINE=MyISAM DEFAULT CHARSET=utf8;

    DROP TABLE IF EXISTS `activemq_msgs`;

    CREATE TABLE `activemq_msgs` (
      `ID` bigint(20) NOT NULL,
      `CONTAINER` varchar(250) DEFAULT NULL,
      `MSGID_PROD` varchar(250) DEFAULT NULL,
      `MSGID_SEQ` bigint(20) DEFAULT NULL,
      `EXPIRATION` bigint(20) DEFAULT NULL,
      `MSG` longblob,
      `PRIORITY` bigint(20) DEFAULT NULL,
      `XID` varchar(250) DEFAULT NULL,
      PRIMARY KEY (`ID`),
      KEY `ACTIVEMQ_MSGS_MIDX` (`MSGID_PROD`,`MSGID_SEQ`),
      KEY `ACTIVEMQ_MSGS_CIDX` (`CONTAINER`),
      KEY `ACTIVEMQ_MSGS_EIDX` (`EXPIRATION`),
      KEY `ACTIVEMQ_MSGS_PIDX` (`PRIORITY`),
      KEY `ACTIVEMQ_MSGS_XIDX` (`XID`)
    ) ENGINE=MyISAM DEFAULT CHARSET=utf8;
    ```

最后你可以尝试中断某个服务，用客户端连接过去测试是否能够实现故障转移。
