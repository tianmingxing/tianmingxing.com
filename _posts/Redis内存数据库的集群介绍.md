---
title: Redis内存数据库的集群介绍
date: 2018-1-15 22:22:14
tags:
- redis
- redis集群
- redis复制集
- redis分片
categories:
- 数据库
- 缓存
---

> 因为单机部署有单点故障隐患，所以在生产环境会采用集群方式部署，以提高系统可用性，也就是在极端条件下服务仍然是可用的。

![](/images/diagram-cluster-architecture.png)

# 复制

> redis支持复制的功能以实现当一台服务器的数据更新后，自动将新的数据同步到其它数据库。

[](/images/redis_replication.png)

把数据库分为主数据库master和从数据库slave，当主数据库可以进行读写操作，从数据库一般是只读的，当主数据库数据变化的时候会自动同步给从数据库。

### 为什么需要复制

1. 可以实现读写分离，从而提高系统业务处理性能。
1. 方便在主数据库奔溃时的数据恢复

### 配置

复制的配置方式非常简单，只需要在从数据库上配置 `slaveof <masterip> <masterport>` 即可。主数据库不用作任何改变。
<!-- more -->
### 测试环境的搭建

1. 方法一：安装三个虚拟机，并分别安装redis。
1. 方法二：在同一台虚拟机上运行多个redis实例，但是要通过配置文件进行区分。
1. 复制一份redis.conf并命名于 `redis_6378.conf`
1. 修改复制出来的配置文件中的内容
    1. `port 6378`
    1. `unixsocket /tmp/redis_6378.sock`
    1. `pidfile /var/run/redis_6378.pid`
    1. `logfile "log_6378.log"`
    1. `dbfilename dump_6378.rdb`
    1. `appendfilename "appendonly_6378.aof"`

假设你和我一样有两个redis实例，主实例端口为6378，从实例端口为6379。你需要在 `redis_6379.conf` 中配置主实例的地址以完成复制的配置。（如果你安装了三台虚拟机，则通过IP就可以做区分。）

#### redis_6379.conf

```bash
# slaveof <masterip> <masterport>
slaveof 127.0.0.1 6378
```

保存配置文件并重启redis实例，在日志中可以看到配置成功的信息。

```bash
5581:S 04 Jul 10:55:40.606 * Connecting to MASTER 127.0.0.1:6378
5581:S 04 Jul 10:55:40.606 * MASTER <-> SLAVE sync started
5581:S 04 Jul 10:55:40.607 * Non blocking connect for SYNC fired the event.
5581:S 04 Jul 10:55:40.607 * Master replied to PING, replication can continue...
5581:S 04 Jul 10:55:40.607 * Partial resynchronization not possible (no cached master)
5581:S 04 Jul 10:55:40.613 * Full resync from master: 268a828c8f410ce4abeda4f8425bd846bb568170:1
5581:S 04 Jul 10:55:40.664 * MASTER <-> SLAVE sync: receiving 76 bytes from master
5581:S 04 Jul 10:55:40.664 * MASTER <-> SLAVE sync: Flushing old data
5581:S 04 Jul 10:55:40.664 * MASTER <-> SLAVE sync: Loading DB in memory
5581:S 04 Jul 10:55:40.665 * MASTER <-> SLAVE sync: Finished with success
```

### 复制的基本操作命令

1. `info replication` 查看复制节点的相关信息
1. `slaveof host port` 可在运行期间修改slave节点的信息，如果该数据库已经是某个主数据库的从数据库，那么会停止和原主数据库的同步关系，转而和新的主数据库同步。
1. `slaveof no one` 使当前数据库停止与其他数据库的同步，转成主数据库

### 复制的基本原理

1. slave启动时会向master发送sync命令（2.8版后发送psync以实现增量复制）
1. 主数据库接到sync请求后在后台保存快照，也就是实现RDB持久化，并将保
存快照期间接收到的命令缓存起来。
1. 快照完成后主数据库会将快照文件和所有缓存的命令发送给从数据库
1. 从数据库接收后会载入快照文件并执行缓存的命令，从而完成复制的初始化。
1. 在数据库使用阶段主数据库会自动把每次收到的写命令同步到从服务器。

### 配置文件中复制部分

1. `slaveof` 指定某一个redis作为另一个redis的从服务器，通过指定IP和端口来设置主redis。建议为从redis设置一个不同频率的快照持久化的周期，或者为从redis配置一个不同的服务端口。
1. `masterauth` 如果主redis设置了验证密码的话（使用requirepass来设置），则在从redis的配置中要使用masterauth来设置校验密码，否则主redis会拒绝从redis的访问请求。
1. `slave-serve-stale-data` 设置当从redis失去了与主redis的连接，或者主从同步正在进行中时，redis该如何处理外部发来的访问请求。
    1. 如果设置为yes（默认）则从redis仍会继续响应客户端的读写请求。
    1. 如果设置为no，则从redis会对客户端的请求返回 `SYNC with master in progress`，当然也有例外，当客户端发来INFO请求和SLAVEOF请求，从redis还是会进行处理。
    1. 从redis2.6版本之后，默认从redis为只读。
1. `slave-read-only` 设置从Redis为只读
1. `repl-ping-slave-period` 设置从redis会向主redis发出PING包的周期，默认是10秒。
1. `repl-timeout` 设置主从同步的超时时间，要确保这个时限比 `repl-ping-slave-period` 的值要大，否则每次主redis都会认为从redis超时。
1. `repl-disable-tcp-nodelay` 设置在主从同步时是否禁用TCP_NODELAY，如果开启，那么主redis会使用更少的TCP包和更少的带宽来向从redis传输数据。但是这可能会增加一些同步的延迟，大概会达到40毫秒左右。如果关闭，那么数据同步的延迟时间会降低，但是会消耗更多的带宽。
1. `repl-backlog-size` 设置同步队列长度。队列长度（backlog）是主redis中的一个缓冲区，在与从redis断开连接期间，主redis会用这个缓冲区来缓存应该发给从redis的数据。这样的话，当从redis重新连接上之后就不必重新全量同步数据，只需要同步这部分增量数据即可。
1. `repl-backlog-ttl` 设置主redis要等待的时间长度，如果主redis等了这么长时间之后，还是无法连接到从redis，那么缓冲队列中的数据将被清理掉。设置为0，则表示永远不清理。默认是1个小时。
1. `slave-priority` 设置从redis优先级。在主redis持续工作不正常的情况，优先级高的从redis将会升级为主redis。而编号越小优先级越高。当优先级被设置为0时，这个从redis将永远也不会被选中。默认的优先级为100。
1. `min-slaves-to-write` 设置执行写操作所需的最少从服务器数量，如果至少有这么多个从服务器， 并且这些服务器的延迟值都少于 `min-slaves-max-lag` 秒， 那么主服务器就会执行客户端请求的写操作。
1. `min-slaves-max-lag` 设置最大连接延迟的时间。min-slaves-to-write和min-slaves-max-lag中有一个被置为0，则这个特性将被关闭。默认情况下min-slaves-to-write为0，而min-slavesmax-lag为10。

### 乐观复制策略

Redis采用乐观复制的策略，容忍在一定时间内主从数据库的内容不同，保存最终的数据会是一样的。这个策略保证了性能，在复制的时候，主数据库并不阻塞，照样处理客户端的请求。

Redis提供了配置来限制只有当数据库至少同步给指定数量的从数据库时，主数据库才可写，否则返回错误。配置是：min-slaves-to-write、min-slavesmax-lag。

### 无硬盘复制

当复制发生时主数据库会在后台保存RDB快照，即使你关闭了RDB它也会这么做，这样就会导致一些问题。

1. 如果主数据库关闭了RDB，现在强行生成了RDB，那么下次主数据库启动的时候，可能会从RDB来恢复数据，这可能是旧的数据。
1. 由于要生成RDB文件，在硬盘性能不高的时候会对性能造成一定影响，因此从2.8.18版本引入了无硬盘复制选项 `repl-diskless-sync`。

## 哨兵（sentinel）

Redis提供了哨兵工具来实现监控系统的运行情况，主要实现：
  1. 监控主从数据库运行是否正常
  1. 当主数据库出现故障时，自动将从数据库转换成为主数据库
  1. 使用Redis-sentinel，redis实例必须在非集群模式下运行

### 开启哨兵功能

1. 找到 `sentinel.conf` 文件并设置被监控主数据库
1. `sentinel monitor (监控的主数据库的名字) 127.0.0.1 6378 1`
1. `1` 表示选举主数据库的最低票数
1. 这个文件的内容在运行期间会被sentinel动态修改，一般追回到文件的最后面。
1. 可以同时监控多个主数据库，一行一个配置即可。
1. 如果监控的主数据库故障，则哨兵将从slave中选举master，后续原来的master服务恢复后会作为slave处理。

**注意**开启哨兵模式后，在应用程序中连接redis时会有一些变化，具体就是在配置连接信息时增加哨兵的地址、端口和密码信息。

## 复制的问题

由于复制中每个数据库都是拥有完整的数据，因此复制的总数据存储量，受限于内存最小的数据库节点，如果数据量过大复制就无能为力了。

# 分片

分片（Partitioning）就是将你的数据拆分到多个Redis实例的过程，这样每个Redis实例只包含完整数据的一部分。

![](/images/redis-partitioning.jpg)

* 常见的分片方式
 * 按照范围分片
 * 哈希分片，比如一致性哈希

### 常见的分片实现

* 在客户端进行分片
* 通过代理来进行分片，比如：Twemproxy
* 查询路由
 * 发送查询到一个随机实例，这个实例会保证转发你的查询到正确的节点。
 * Redis集群在客户端的帮助下，实现了查询路由的一种混合形式，请求不是直接从Redis实例转发到另一个，而是客户端收到重定向到正确的节点。
* 在服务器端进行分片
 * Redis采用哈希槽（hash slot）的方式在服务器端进行分片
 * Redis集群有16384个哈希槽，使用键的CRC16编码对16384取模来计算一个键所属的哈希槽。

### 分片的问题

* 不支持涉及多键的操作，如mget。如果所操作的键都在同一个节点就正常执行，否则会提示错误。
* 分片的粒度是键，因此每个键对应的值不要太大。
* 数据备份会比较麻烦，备份数据时你需要聚合多个实例和主机的持久化文件
* 扩容的处理比较麻烦。
* 故障恢复的处理会比较麻烦，可能需要重新梳理Master和Slave的关系，并调整每个复制集里面的数据。

# 集群

> 由于数据量过大单个复制集难以承担，因此需要对多个复制集进行集群，形成水平扩展，每个复制集只负责存储整个数据集的一部分，这就是Redis的集群。

* 在以前版本中Redis的集群是依靠客户端分片来完成，但是这会有很多缺点，比如维护成本高，需要客户端编码解决；增加、移出节点都比较繁琐等。
* Redis3.0新增的一大特性就是支持集群，在不降低性能的情况下还提供了网络分区后的可访问性和支持对主数据库故障的恢复。
* 使用集群后只能使用默认的0号数据库。
* 每个Redis集群节点需要两个TCP连接打开，正常的TCP端口用来服务客户端，例如6379，加10000的端口用作数据端口，必须保证防火墙打开这两个端口。
* Redis集群不保证强一致性，这意味着在特定的条件下Redis集群可能会丢掉一些被系统收到的写入请求命令。


# 集群架构

![](/images/redis_cluster_1.jpg)

* 所有的Redis节点彼此互联，内部使用二进制协议优化传输速度和带宽。
* 节点的fail是通过集群中超过半数的节点检测失效时才生效。
* 客户端与Redis节点直连，不需要中间proxy层。客户端不需要连接集群所有节点，只要连接集群中任何一个可用节点即可。
* 集群把所有的物理节点映射到[0-16383]插槽上，集群负责维护 `节点-插槽-值` 的关系。

# 相关操作命令

1. `CLUSTER INFO` 获取集群的信息
1. `CLUSTER NODES` 获取集群当前已知的所有节点，以及这些节点的相关信息
1. `CLUSTER MEET <ip> <port>` 将ip和port所指定的节点添加到集群当中
1. `CLUSTER FORGET <node_id>` 从集群中移除 node_id 指定的节点
1. `CLUSTER REPLICATE <node_id>` 将当前节点设置为 node_id 指定的节点的从节点
1. `CLUSTER SAVECONFIG` 将节点的配置文件保存到硬盘里面
1. `CLUSTER ADDSLOTS <slot> [slot ...]` 将一个或多个槽分配给当前节点
1. `CLUSTER DELSLOTS <slot> [slot ...]` 从当前节点移除一个或多个槽
1. `CLUSTER FLUSHSLOTS` 移除分配给当前节点的所有槽
1. `CLUSTER SETSLOT <slot> NODE <node_id>` 将槽分配给 node_id 指定的节点，如果槽已经分配给另一个节点，那么先让另一个节点删除该槽>，然后再进行分配
1. `CLUSTER SETSLOT <slot> MIGRATING <node_id>` 将本节点的槽迁移到指定的节点中
1. `CLUSTER SETSLOT <slot> IMPORTING <node_id>` 从指定节点导入槽到本节点
1. `CLUSTER SETSLOT <slot> STABLE` 取消对槽的导入（import）或迁移（migrate）
1. `CLUSTER KEYSLOT <key>` 计算键 key 应该被放置在哪个槽
1. `CLUSTER COUNTKEYSINSLOT <slot>` 返回槽目前包含的键值对数量
1. `CLUSTER GETKEYSINSLOT <slot> <count>` 返回 count 个槽中的键
1. `MIGRATE 目的节点ip 目的节点port 键名 数据库号码 超时时间 [copy] [replace]` 迁移某个键值对

# 手动创建插槽

* 准备6个实例来组成一个集群，你可以在同一台虚拟机上复制6份配置文件 `redis.conf`，通过里面的端口来进行区分，除端口外还有一些其它的配置也要改，具体可参考之前的文章。
 * 假设你准备了6个redis实例，端口范围从 `6374` 至 `6379`。
 * 保证没有之前文章中讲述的复制集的内容，如果有请恢复到原始状态。

  ```
  [redis@bogon redis-3.2.9]$ ll redis*
-rw-rw-r--. 1 redis redis 46716 7月   4 15:38 redis6374.conf
-rw-rw-r--. 1 redis redis 46716 7月   4 15:39 redis6375.conf
-rw-rw-r--. 1 redis redis 46716 7月   4 15:41 redis6376.conf
-rw-rw-r--. 1 redis redis 46716 7月   4 15:41 redis6377.conf
-rw-rw-r--. 1 redis redis 46750 7月   4 15:41 redis6378.conf
-rw-rw-r--. 1 redis redis 46740 7月   4 15:42 redis6379.conf
  ```
* 修改这6个配置文件的内容
 ```
 cluster-enabled yes
 cluster-config-file nodes-6374.conf
 ```

 * 请再一下检查这些值是否做了区分 `pidfile` `port` `logfile` `dbfilename` `unixsocket`（每份配置文件都不能完全相同，建议用端口来作区分）

* 分别启动这些redis数据库 `redis-server redis6374.conf`，使用 `redis-cli -p 6374` 登录对应的实例并使用 `info cluster` 查看信息。
* 使用 `cluster meet` 连接各个节点，这样可以把所有的数据库都放到一个集群中。 

 ```
 [redis@bogon redis-3.2.9]$ redis-cli -p 6374
127.0.0.1:6374> cluster nodes
052100ccbf0d912fc3f9ecbac63bfa7ff5256d96 127.0.0.1:6375 master - 0 1499156448710 1 connected
b537b00eed7a2271f52c4b9de3dce0a0b4b3c798 127.0.0.1:6379 slave 5ad7c308c8e9cb41f5c6d8f46b0f7a283a353826 0 1499156450747 4 connected
5ad7c308c8e9cb41f5c6d8f46b0f7a283a353826 127.0.0.1:6378 master - 0 1499156444637 4 connected 12706
5f5238a05fce1f2f75a9f487813c4b883d1c03ac 127.0.0.1:6376 master - 0 1499156451772 2 connected
af95d2c2bb83bb7c5a923655c472d9b3956a51b7 127.0.0.1:6377 slave 5ad7c308c8e9cb41f5c6d8f46b0f7a283a353826 0 1499156449726 4 connected
88d4c64307564d3d2d515db929e42e70647909d0 127.0.0.1:6374 myself,master - 0 0 3 connected
 ```

* 使用 `cluster replicate (节点编号)` 设置部分数据库为slave，这个命令要在你计划它为从的实例上执行。我演示的是3主3从（6374->6375，6376->6377，6378->6379）
 ```
 127.0.0.1:6375> cluster replicate 88d4c64307564d3d2d515db929e42e70647909d0
OK
 ```

至此手动创建集群已经结束。

# 什么是插槽

> 插槽是Redis对Key进行分片的单元。在Redis的集群实现中内置了数据自动分片机制，集群内部会将所有的key映射到 `16384` 个插槽中，集群中的每个数据库实例负责其中部分的插槽的读写。

## 键与插槽的关系

* Redis会将key的有效部分使用 `CRC16` 算法计算出散列值，然后对 `16384` 取余数，从而把key分配到插槽中。
* 键名的有效部分规则
 1. 如果键名包含 `{}` 那么有效部分就是 `{}` 中的值
 1. 否则就是取整个键名

## 移动已分配的插槽

* 假设要迁移 `10023` 号插槽从 `实例A` 到 `实例B`
 1. 在B上执行 `cluster setslot 10023 importing A`
 1. 在A上执行 `cluster setslot 10023 migrating B`
 1. 在A上执行 `cluster getkeysinslot 10023` 要返回的数量
 1. 对上一步获取的每个键执行migrate命令将其从A迁移到B
 1. 在集群中每个服务器上执行 `cluster setslot 10023 node B`

## 防止在移动已分配插槽过程中键的临时丢失
上面迁移方案中的前两步就是用来避免在移动已分配插槽过程中，键的临
时丢失问题的，大致思路如下：

1. 当前两步执行完成后，如果客户端向A请求插槽10023中的键时，如果键还未被转移，A将继续处理请求。
2. 如果键已经转移则返回新的地址给客户端，由客户端发起新的请求以获取数据。

## 获取插槽对应的节点

* 当客户端向某个数据库发起请求时，如果键不在这个数据库里面，将会返回一个move重定向的请求，里面包含新的地址，客户端收到这个信息后，需要重新发起请求到新的地址去获取数据。
* 大部分的Redis客户端都会自动去重定向，也就是这个过程对开发人员是透明的。
* redis-cli也支持自动重定向，只需要在启动时加入 -c 的参数。

## 故障判定

1. 集群中每个节点都会定期向其他节点发出ping命令，如果没有收到回复，就认为该节点为疑似下线，然后在集群中传播该信息。
1. 当集群中的某个节点，收到半数以上认为某节点已下线的信息，就会真的标记该节点为已下线，并在集群中传播该信息。
1. 如果已下线的节点是master节点，那就意味着一部分插槽无法写入了。
1. 如果集群任意master挂掉，且当前master没有slave，集群进入fail状态。
1. 如果集群超过半数以上master挂掉，无论是否有slave，集群进入fail状态。
1. 当集群不可用时，所有对集群的操作做都不可用，收到 `CLUSTERDOWN The cluster is down` 错误信息。

## 故障恢复

发现某个master下线后，集群会进行故障恢复操作，来将一个slave变成master（基于Raft算法），大致步骤如下：

1. 某个slave向集群中每个节点发送请求，要求选举自己为master。
1. 如果收到请求的节点没有选举过其他slave会同意
1. 当集群中有超过节点数一半的节点同意该slave的请求，则该Slave选举成功。
1. 如果有多个slave同时参选，可能会出现没有任何slave当选的情况，将会等待一个随机时间，再次发出选举请求。
1. 选举成功后，slave会通过 `slaveof no one` 命令把自己变成master。如果故障后还想集群继续工作，可设置 `cluster-require-full-coverage no`。

## 对于集群故障恢复的说明

1. master挂掉了重启还可以加入集群；但挂掉的slave重启，如果对应的master变化了是不能加入集群的，除非修改它们的配置文件将其master指向新master。
1. 只要主从关系建立就会触发主和该从采用save方式持久化数据，不论你是否禁止save。
1. 在集群中如果默认主从关系的主挂了并立即重启，如果主没有做持久化，数据会完全丢失，从而从的数据也被清空。

# 使用redis-trib.rb来操作集群

* redis-trib.rb是Redis源码中提供的一个辅助工具，可以非常方便的来操作集群，它是用ruby写的，因此需要在服务器上安装相应环境。
* 下面两种方式选择其中一种即可

### 使用yum安装

```
yum install ruby -y
```

### 源码安装

1. 安装Ruby
 1. 下载安装包，地址https://www.ruby-lang.org/en/downloads/
 1. 然后分别configure、make、make install
1. 还需要安装rubygems
 1. 下载安装包，地址https://rubygems.org/pages/download
 1. 解压后进入解压文件夹运行 `ruby setup.rb`。

### 安装redis的ruby library

1. 由于连接国外源不太稳定，建议安装国内源 `gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/`。
1. 可以通过 `gem sources -l` 查看源并确保只有gems.ruby-china.org。
1. 运行 `gem install redis`

## 使用redis-trib.rb来初始化集群

```sh
ruby redis-trib.rb create --replicas 1 127.0.0.1:6374
127.0.0.1:6375 127.0.0.1:6376 127.0.0.1:6377 127.0.0.1:6378
127.0.0.1:6379
```

`create` 表示要初始化集群，`--replicas 1` 表示每个驻数据库拥有的从数据
库为1个。

## 使用redis-trib.rb来迁移插槽

1. 执行 `ruby redis-trib.rb reshard ip:port`，这就告诉Redis要重新分片，
`ip:port` 可以是集群中任何一个节点。
1. 然后按照提示去做
1. 这种方式不能指定要迁移的插槽号

# 预分区

> 为了实现在线动态扩容和数据分区，Redis的作者提出了预分区的方案，实
际就是在同一台机器上部署多个Redis实例，当容量不够时将多个实例拆分到不同的机器上，这样就达到了扩容的效果。

## 拆分过程

1. 在新机器上启动好对应端口的Redis实例
1. 配置新端口为待迁移端口的从库
1. 待复制完成并与主库完成同步后，切换所有客户端配置到新的从库的端口
1. 配置从库为新的主库
1. 移除老的端口实例
1. 重复上述过程把要迁移的数据库转移到指定服务器上

以上拆分流程是Redis作者提出的一个平滑迁移的过程，不过该拆分方法还
是很依赖Redis本身的复制功能的，如果主库快照数据文件过大，这个复制的过程也会很久，同时会给主库带来压力。