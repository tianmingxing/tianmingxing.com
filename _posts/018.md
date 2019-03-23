---
title: Redis内存数据库介绍及应用
date: 2018-1-2 23:26:39
tags:
- Redis
- nosql
- 缓存
categories:
- 数据库
---

> 详细介绍NoSQL内存型数据库redis，包括安装部署、各种数据结构的命令使用，集群的搭建等内容。

![](/images/redis_logo.png)

# 介绍

Redis（REmote DIctionary Server远程字典服务器）是一个使用c编写、开源、KV型、基于内存运行并支持持久化的NoSQL数据库，它也是当前最热门的NoSQL数据库之一。

### 几种流行数据库对比

|Name |Type |Data storage options |Query types |Additional features |
|-------|-------|-----------|----------|---------------|-------------|
|Redis |In-memory non-relational database |strings,lists,sets,hashes,sorted sets |Commands for each data type for common access patterns, with bulk operations, and partial transaction support |publish/subscribe, master/salve replication, disk persistence, scription(stored procedures) |
|memcached |in-memory key-value cache |mapping of keys to values |commands for create, read, update, delete, and a few others |Multithreaded server for additional performance |
|mysql |relational database |databases of tables of rows, views over tables, spatial and third-party extensions |select, insert, update, delete, functions, stored procedures |acid compliant(with innoDB), master/salve and master/master replication |
<!-- more -->
### redis和memcached的应用场景

1. 如果单纯做kv缓存用而不考虑到其它的需求，比如缓存计算、丰富的数据类型，此时优选memcached。
1. 如果不满足上面的条件则选择redis更适合一些。

# 安装

> 下面所有操作在CentOS7上进行

## 源码安装

1. 前往官网下载最新版本，此刻我以[3.2.9](http://download.redis.io/releases/redis-3.2.9.tar.gz)为例介绍。
    ```bash
    wget -c http://download.redis.io/releases/redis-3.2.9.tar.gz
    ```
1. 解压并进入文件夹
    ```bash
    tar -zxvf redis-3.2.9.tar.gz
    cd redis-3.2.9
    ```
1. 编译并安装
    ```bash
    make
    make install
    ```
1. 安装完成后在目录 `/usr/local/bin` 下多了一些命令，它们的作用分别是：
  * redis-benchmark：性能测试工具
  * redis-check-aof：修复有问题的AOF文件
  * redis-check-rdb：修复有问题的RDB文件
  * redis-cli：命令行操作工具
  * redis-sentinel：集群管理工具
  * redis-server：服务启动程序

## YUM源安装

如果想简化安装过程，可以直接用yum命令安装，不过这样不能保证安装比较新版的软件，你可能需要更新源文件之类的操作，具体这方面的介绍可以参考其它资料。

```bash
yum install -y redis
```

## 启动redis服务

> 如果是yum安装的redis则可以不需要指定配置文件启动，它会默认加载缺省的配置文件，但如果是源码安装则需要进行指定。如果想在同一台机器上启动多个redis实例，也是需要为每个单独指定配置文件的。

1. 可以在安装目录中找到一份配置文件并将其复制到 `/etc` 下面。
    ```bash
    sudo mkdir /etc/redis
    sudo cp redis.conf /etc/redis/6379-redis.conf
    ```
1. 让redis以守护进程执行（后台执行）
    ```bash
    vim /etc/redis/6379-redis.conf
    
    ## 找到 daemonize 参数并修改值为yes
    daemonize yes
    ```
1. 运行程序
    ```bash
    redis-server /etc/redis/6379-redis.conf
    ```

## 命令行客户端

* 使用命令行工具连接redis服务，这是官方提供的一个连接redis服务的命令行工具，在里面可以使用全部redis操作命令，你所能想到的比如切换数据库、CURD数据等等。
    ```bash
    ## 默认主机127.0.0.1和端口6379，所以可以直接执行下面的命令而不必指定任何参数。
    redis-cli

    ## 当然你显式指明参数启动客户端连接也可以
    redis-cli -h 127.0.0.1 -p 6379

    [redis@bogon redis-3.2.9]$ redis-cli
    127.0.0.1:6379> select 1
    OK
    127.0.0.1:6379[1]> quit
    ```
* 关闭redis服务
  1. 建议使用redis的命令关闭服务，这样能够避免数据丢失或破损的情况。
  1. 如果你还在命令行终端内可以输入 `shutdown` 并回车来关闭服务。
  1. 也可以直接使用 `redis-cli shutdown` 来关闭服务，如果不是默认端口则需要另行指定。
  1. 另一种方法是使用 `kill -9` 来强制关闭redis的服务进程，这非常有可能导致丢数据，因为此时缓存中的数据还来不及持久化到文件中。

## 性能测试

* 安装完成后可以使用 `redis-benchmark` 工具对机器性能进行测试，它的作用是测试刚才安装的redis在当前的硬件及系统环境下的读写性能。**注意**在使用本工具前必须先启动redis服务。
    ```bash
    [redis@bogon redis-3.2.9]$ redis-benchmark
    ====== PING_INLINE ======
      100000 requests completed in 1.40 seconds
      50 parallel clients
      3 bytes payload
      keep alive: 1

    95.95% <= 1 milliseconds
    99.90% <= 2 milliseconds
    100.00% <= 2 milliseconds
    71174.38 requests per second

    ......
    ```

# 基础知识

## 单进程

redis程序采用的是单进程模型来处理客户端的请求，对读写等事件的响应是通过对 `epoll` 函数的包装来实现。

redis的实际处理速度完全依靠主进程的执行效率，假如同时有多个客户端并发访问，则服务器处理能力在一定情况下将会下降。假如要提升服务器的并发能力，那可以采用在单台服务器上部署多个redis进程的方式。

## 多数据库

redis每个数据库是用从 `0` 开始递增的数字来命名的，初始化 `16` 个（可以在配置文件中修改）数据库，默认使用 `0` 号数据库，你也可以使用 `select [数字]` 来切换数据库。

* `dbsize` 查看当前数据库KEY的数量
* `move key db` 在多个数据库之间移动数据
* `flushdb` 清除当前数据库的全部数据

redis不支持自定义数据库名字，不支持为每个数据库设置不同的访问密码（只能是统一的密码），多个数据库之间并不完全的隔离，使用 `flushall` 可以清空全部的数据。redis的数据库更像是一个命名空间。

## 数据类型

* KEY（键）
  1. key是字符串类型，如果中间有空格或者转义字符要用引号引起来，当然不建议这么做，这在低版本上可能不被支持。
  1. 命名建议 `对象类型:对象标识(ID):对象属性`，例如 `user:u100001:age`
  1. 多个单词之间用 `.` 分隔，例如 `user:u100001:body.head`
  1. 在可读的情况下key应该尽量有意义并且简短
* VALUE（值）
  1. String：可以存储String/Integer/Float类型的数据，甚至是二进制数据，一个字符串最大容量是512M
  1. List：底层实现上不是数组而是链表，也就是说在头部和尾部插入一个新元素，其时间复杂度是常数级别。当然也有弊端，比如元素定位比数组慢。
  1. Set：无序不可重复列表，是通过HashTable来实现。
  1. Hash：按照K/V方式来存储字符串，类似于java中的Map
  1. ZSet：有序且不可重复，根据Score来排序。使用散列表和跳跃表来实现，所以读取中间部分数据也很快。
  1. Geo：地理空间坐标的存储，很方便的计算两点之间的直线距离。

## 基本操作

### 键操作

1. `keys pattern` 获得符合规则的键名列表，pattern支持glob风格通配符格式
  * `?` 匹配一个字符
  * `*` 匹配任意字符
  * `[]` 匹配中括号内的任一字符，可以用来表示一个范围
  * `\x` 匹配字符x，用于转义符号
    ```bash
    # 查询1号数据库中所有键，返回一个列表
    127.0.0.1:6379[1]> keys *

    # 查询1号数据库中以t开头的键列表
    127.0.0.1:6379[1]> keys t?[a-z]
    ```
1. `exists key` 判断指定键是否存在，如果存在返回1（非0即真）
1. `scan cursor [MATCH pattern] [COUNT count]` 基于游标的迭代器，可以用来遍历键（分页浏览键）。
    ```bash
    127.0.0.1:6379[1]> scan 0 count 3
    1) "5"
    2) 1) "myhash"
       2) "myzset"
       3) "a"
    127.0.0.1:6379[1]> scan 5 count 3
    1) "7"
    2) 1) "myset"
       2) "key2"
       3) "myset2"
       4) "myset3"
    `</pre>
    ```

命令的使用方法是类似的，下面对命令的作用和使用方式做一个说明。

### 删除

1. `del key` 删除指定键值
1. del命令虽不支持通配符，但可以结合linux管道和xargs命令来自定义删除。例如 `redis-cli keys * | xargs redis-cli del` ，如果键名称中有空格可能并不会被删除。

### 类型

1. `type key` 获得键值的数据类型
1. 数据都是以string类型存储，在做计算时会自动进行类型转换。

### 重命名

1. `rename key newkey` 修改指定键名称
1. `renamenx key newkey`  如果新的键名不存在则改名，防止已存在键被覆盖。

### 生存&过期时间

1. `expire key seconds` 修改键的生存时间（单位秒）
1. `pexpireat key milliseconds-timestamp` 修改键的生存时间（单位毫秒）
1. `expireat key timestamp` 修改键的到期时间（时间戳），从1970年1月1日起的秒数
1. `pexpireat key milliseconds-timestamp` 修改键的到期时间（时间戳），从1970年1月1日起的毫秒数
1. `ttl key` 返回键剩余生存时间（单位秒）
1. `pttl key` 返回键剩余生存时间（单位毫秒）
1. `randomkey` 返回随机一个键

### 迁移

1. `migrate host port key| destination-db timeout [COPY] [REPLACE] [KEYS key]` 在两个redis实例之间迁移键
1. `MIGRATE 192.168.1.34 6379 "" 0 5000 KEYS key1 key2 key3`

# Redis数据类型介绍

Redis服务启动之后使用命令行客户端连接，你也可以借助一些Windows下面的客户端（<a href="https://redisdesktop.com/">redisdesktop</a>）来操作。

```bash
# /usr/local/bin/redis-cli
127.0.0.1:6379>
```

在下面示例命令中，参数用 `[]` 包裹起来则表示这个参数是可选的，中间的 `...` 表示参数的个数是多个。

由于下面命令的操作方式在 <a href="https://redis.io/commands">Redis官方文档</a> 上有详细示例，所以我只对命令的作用做了说明，并不逐一进行演示。

## String

1. `append key value`  在值尾部追回字符
1. `bitcount key [start end]` 获取范围内为1的二进制位数。如果值为字母a，则对应的二进制为01100001（ASCII码97表示小字字母a），则里面存在1的位数就是3个。
1. `bitfield key [GET type offset] [SET type offset value] [INCRBY type  offset increment] [OVERFLOW WRAP|SAT|FAIL]` 对字符串执行任意的位域整数运算
1. `bitop operation destkey key [key ...]` 对多个二进制值进行位操作，操作有 `and` `or` `xor` `not`。
1. `bitpos key bit [start] [end]` 将字符串中第一个位的位置返回到1或0
1. `decr key` 递减整数值（步进为1），如果操作的值不是整数则会出错 `(error) ERR value is not an integer or out of range`
1. `decrby key decrement` 递减指定步进的整数值
1. `get key` 获取指定键的值
1. `getbit key offset` 获取指定位置的二进制位的值
1. `getrange key start end` 获取指定索引范围内的值
1. `getset key value` 原子设置键的值并返回key的旧值
1. `incr key`  递增或递减整数值（步进为1），如果操作的值不是整数则会出错 `(error) ERR value is not an integer or out of range`
1. `incrby key increment` `decrby key decrement` 递增或递减指定步进的整数值
1. `incrbyfloat key increment` 递增和递减浮点数值
1. `mget key [key ...]` 同时获得多个键的值
1. `mset key value [key value ...]` 同时设置多个键值
1. `msetnx key value [key value ...]` 同时设置多个键值，如果设置的键已经存在则本次操作全都会失败。
1. `psetex key milliseconds value` 设置键值和到期时间（以毫秒为单位）
1. `set key value [EX seconds] [PX milliseconds] [NX|XX]` 设置键和值。EX设置指定的到期时间（以秒为单位），PX设置指定的到期时间（以毫秒为单位），NX只有在不存在的情况下才设置键，XX只有当它已经存在时才设置键。
1. `setbit key offset value` 设置指定位置的二进制位的值
1. `setex key seconds value` 设置键值和到期时间（以秒为单位）
1. `setnx key value` 设置键值，只有该键不存在时才设置成功。
1. `setrange key offset value` 在指定索引位置设置值（会从索引位置往后覆盖）
1. `strlen key` 获取指定键对应的值的长度

## List

1. `blpop key [key ...] timeout` 从队列左侧弹出值，如果没有值则一直等待并直到过期
1. `brpop key [key ...] timeout` 从队列右侧弹出值，如果没有值则一直等待并直到过期
1. `brpoplpush source destination timeout` 将元素从一个列表转移到另一个列表，可以指定超时时间。
1. `lindex key index` 获取指定索引对应的值
1. `linsert key BEFORE|AFTER pivot value` 在指定值前或后插入元素，不会发生覆盖指定值的情况。
1. `llen key` 获取队列中元素的个数
1. `lpop key` 从队列左边中弹出值，弹出后队列中就没有这个值了。
1. `lpush key value [value ...]` 添加一个或多个值，从列表左侧加入
1. `lpushx key value` 添加一个值，只有当列表存在时才将值添加进去（不会自动创建列表）
1. `lrange key start stop` 按索引范围获取列表值，-1表示最后一个索引
1. `lrem key count value` 删除元素，count可正负，表示从左或右删除，如果值为0表示删除全部与给定值相等的项。
1. `lset key index value` 设置列表中指定索引的值，原来的值会被覆盖
1. `ltrim key start stop` 保留指定索引区间的元素
1. `rpop key` 从队列右边中弹出值，弹出后队列中就没有这个值了。
1. `rpoplpush source destination` 将元素从一个列表转移到另一个列表，目的列表名称可以是自身。
1. `rpush key value [value ...]` 添加一个或多个值，从列表右侧加入
1. `rpushx key value` 只有列表存在时才将值添加到列表中

## Set

1. `sadd key member [member ...]` 往集合中添加元素，重复加的元素会去重
1. `scard key` 统计集合中的元素个数 
1. `sdiff key [key ...]` 差集，返回在第一个set里面而不在后面任何一个set里面的项
1. `sdiffstore destination key [key ...]` 差集并保留结果到指定集合中
1. `sinter key [key ...]` 交集，返回多个set里面都有的项 
1. `sinterstore destination key [key ...]` 交集并保留结果到指定的集合中 
1. `sismember key member` 判断元素是否在集合中 ，如果存在则返回1，反之返回0
1. `smembers key` 获取集合中所有元素 
1. `smove source destination member` 移动元素
1. `spop key [count]` 可以弹出元素，弹出的个数可以选填
1. `srandmember key [count]` 随机获取集合中的元素，数量为正时会随机获取多个不重复的元素；如果数量为负会随机返回多个重复的元素；数量大于集合元素则返回全部。
1. `srem key member [member ...]`  从集合中删除元素
1. `sunion key [key ...]` 并集，合并多个集合中的元素，如果有重复元素则去重 
1. `sunionstore destination key [key ...]` 并集并保留结果到指定集合中
1. `sscan key cursor [MATCH pattern] [COUNT count]` 基于游标遍历集合

## Hash

1. `hdel key field [field ...]` 删除某个项，并不会删除这个键
1. `hexists key field` 判断键里面的值是否存在
1. `hget key field` 获取指定键下面某个元素对应的值
1. `hgetall key` 获取该键下面所有的键值对
1. `hincrby key field increment` 增减整数数值，前提是项的值必须是整数
1. `hincrbyfloat key field increment` 增减浮点数值
1. `hkeys key` 获取所有项的键
1. `hlen key` 获取键里面的键值对数量
1. `hmget key field [field ...]` 同时获取多个值
1. `hmset key field value [field value ...]` 同时设置多个值
1. `hset key field value` 设置值
1. `hsetnx key field value` 设置值，如果field不存在时才设置
1. `hstrlen key field` 获取键值对中值的字符长度
1. `hvals key` 获取所有项的值
1. `hscan key cursor [MATCH pattern] [COUNT count]` 基于游标遍历集合

## ZSet

1. `zadd key [NX|XX] [CH] [INCR] score member [score member ...]` 添加元素，score和项可以是多对，score可以是整数或浮点数，+inf表示正无穷大，-inf表示负无穷大。
1. `zcard key` 统计集合中元素个数
1. `zcount key min max` 获取分数区间内元素个数
1. `zincrby key increment member` 增减元素的分数
1. `zinterstore destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]` 交集
1. `zlexcount key min max` 获取值按字典顺序区间内元素个数
1. `zrange key start stop [WITHSCORES]` 获取索引区间内的元素
1. `zrangebylex key min max [LIMIT offset count]` 获取区间元素根据值的字典顺序
1. `zrevrangebylex key max min [LIMIT offset count]` 获取区间元素根据值的字典顺序，结果是倒序的。
1. `zrangebyscore key min max [WITHSCORES] [LIMIT offset count]` 获取分数区间内的元素，默认包含端点值，如果加上 `(` 表示不包含。可以使用limit限制返回元素个数。
1. `zrank key member` 获取元素在集合中的索引
1. `zrem key member [member ...]` 删除一个或多个元素
1. `zremrangebylex key min max` 删除索引区间内的元素，根据值的字典顺序计算区间
1. `zremrangebyrank key start stop` 删除索引区间内的元素
1. `zremrangebyscore key min max` 删除分数区间内的元素
1. `zrevrange key start stop [WITHSCORES]` 获取索引（倒序）区间内的元素
1. `zrevrangebyscore key max min [WITHSCORES] [LIMIT offset count]` 获取分数（倒序）区间内的元素
1. `zrevrank key member` 获取元素在集合中的倒序的索引，即从右往左计算（或从下往上计算）
1. `zscore key member` 获取元素的分数
1. `zunionstore destination numkeys key [key ...] [WEIGHTS weight] [AGGREGATE SUM|MIN|MAX]` 并集
1. `zscan key cursor [MATCH pattern] [COUNT count]` 基于游标遍历集合

## 排序

1. `sort key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC|DESC] [ALPHA] [STORE destination]` 可以对List、Set和ZSet里面的值排序。
1. `BY` 设置排序的参考键，可以是字符串类型或者Hash类型里面的某个Item键，格式：`Hash键名:*->Item键`。使用此参数后sort不再依据元素的值来排序，它会将每个元素的值替换成参考键中的第一个 `*`，然后获取相应的值并对其排序。如果参考键不存在则默认为0，如果参考键的值一样，再以元素本身的值进行排序。
1. `GET` 指定sort命令返回结果包含的键的值，格式：Hash键名:`*->Item键`，可以指定多个get，其返回的结果一行一个。如果要返回元素的值使用 `GET #`。
1. 对较大数据量排序会严重影响性能，建议：
  1. 尽量减少待排序集合中的数据
  1. 使用limit来限制获取的数据量
  1. 如果要排序的数据量较大，可以考虑使用store参数来缓存结果。

## 数据过期机制

1. 当客户端主动访问某个key时，redis服务会对找到的key进行超时判断，过期的key此刻会被立刻删除；
1. redis服务在后台每秒10次执行如下操作：
  1. 随机选取20个key校验是否过期
  1. 删除所有已发现的键
  1. 如果超过25%的键已过期，则重新开始第一步操作。
1. 在过期key不多的情况下redis最多每秒回收200条左右，这种简单的概率算法保证不被访问的数据，在过期之后也会被及时删除。

## 处理过期键

1. `expire key seconds` 设置过期时间，单位秒
1. `expireat key timestamp` 设置过期时间，到秒的时间戳
1. `ttl key` 查看键的生存时间，单位秒，-1表示永不过期，-2表示已过期
1. `persist key` 设置键永不过期；使用set或getset命令为键赋值的时候也会清除原先的过期时间。
1. `pttl key` 查看还有多少毫秒过期
1. `pexpire key milliseconds` 设置过期时间，单位毫秒
1. `pexpireat key milliseconds-timestamp` 设置过期时间，到毫秒的时间戳

# 配置详解

## 动态配置

1. `config get/set 配置名称` 可以在redis-cli里面使用config命令来获取或设置redis配置，可以不用重新启动redis。
1. 并不是所有的配置参数都可以通过config命令在运行期修改，比如：`deamonize` `pidfile` `port` `database` `dir` `slaveof` `rename-command` 等。


## 静态配置

> 参数配置在redis.conf文件中，你可以在安装目录中找到它，我将对里面每部分作一个介绍。

### 通用部分

1. 度量单位
    ```bash
    1k => 1000 bytes
    1kb => 1024 bytes
    1m => 1000000 bytes
    1mb => 1024*1024 bytes
    1g => 1000000000 bytes
    1gb => 1024*1024*1024 bytes
    ```
    单位不区分大小写，所以 `1GB 1Gb 1gB` 都一样。基础的单位不支持bit只支持bytes。
1. 子配置文件
    ```bash
    include /path/to/local.conf
    include /path/to/other.conf
    ```
    可以把一些通用的配置移到外面，供多个实例共享，而不必重复去配置。
1. `daemonize` 是否以后台守护进程方式启动redis服务
1. `pidfile` pid文件位置，默认会生成在 `/var/run/redis.pid`
1. `bind` 指定要绑定的IP，默认会响应梧桐所有可用网卡的连接请求。
1. `port` 监听的端口号，默认服务端口6379，0表示不监听端口；如果redis不监听端口可以通过unix socket文件来接收请求。
1. `tcp-backlog` 设置tcp的backlog，它其实是一个连接队列，backlog队列总和 = 未完成三次握手队列 + 已经完成三次握手队列。在高并发环境下需要一个高的backlog值来避免慢客户端连接问题。
1. `timeout` 连接空闲超时时间，0表示永不关闭
1. `tcp-keepalive` 单位为秒，如果设置为0则不会进行在线检测。
1. `loglevel` log信息级别，共分四个级别：debug、verbose、notice和warning
1. `logfile` log文件位置，如果设置为空字符串，则redis会将日志输出到标准输出，即控制台。
1. `syslog-enabled` 是否把日志输出到syslog中
1. `syslog-ident` 指定日志标志
1. `syslog-facility` 指定syslog设置
1. `databases` 开启数据库的数量，编号从0开始。
1. `maxclients` 设置redis同时可以与多少个客户端连接，默认为1万个客户端。当连接超过此限制时会报错 `max number of clients reached`
1. `maxmemory` 设置redis可以使用的内存量。一旦达到内存使用上限，redis将试图移除内部数据，移除规则可以通过 `maxmemory-policy` 指定。
1. `maxmemory-policy` 设置内存移除规则，redis提供了多达6种规则。
  1. `volatile-lru` 使用LRU算法移除key，只对设置了过期时间的键操作
  1. `allkeys-lru` 使用LRU算法移除key
  1. `volatile-random`  在过期集合中移除随机的key，只对设置了过期时间的键
  1. `allkeys-random` 移除随机的key
  1. `volatile-ttl` 移除那些ttl值最小的key，即那些最近要过期的key
  1. `noeviction`  不进行移除，针对写操作返回错误信息。
1. `maxmemory-samples` 设置样本数量，LRU算法和最小TTL算法都并非精确的算法，而是估算值。

## 持久化

> redis持久化分成两种方式 `RDB（Redis Data Base）` 和 `AOF（Append Only File）`。

1. `RDB` 在不同的时间点，将redis某一时刻的数据生成快照并存储到磁盘上。
1. `AOF` 只允许追回不允许改写的文件，将redis执行过的所有写指令记录下来，在下次redis重新启动时，把这些指令从前到后再重复执行一次。
1. `RDB` 和 `AOF` 可以同时使用，在这种情况下如果redis重启的话，则会优先采用AOF方式来进行数据恢复，这是因为AOF方式的数据恢复完整度更高。
1. 可以关闭执久化，即redis变成一个纯内存数据库，类似于Memcache一样。
1. 在配置文件 `redis.conf` 中可以开启AOF功能 `appendonly yes`

### RDB

redis会单独创建（fork）一个子进程来进行持久化，会先将内存中的数据写入到一个临时文件中，当持久化过程结束完再用这个临时文件替换上次持久化的文件。整个过程中主进程不进行任何IO操作，从而确保极高的性能。

#### 配置

1. `save * *` 保存快照的频率。第一个 `*` 表示多长时间（单位秒），第二个 `*` 表示至少执行写操作的次数；在一定时间内至少执行一定数量的写操作时，就自动保存快照；可以设置多个条件。如果想禁用RDB持久化策略，只需要不设置快照频率或给save传空字符串即可。
    ```bash
    save 900 1
    save 300 10
    save 60 10000
    ```
1. 如果用户开启了RDB快照功能，那么在redis持久化数据到磁盘时如果出现失败，默认情况下redis会停止接收所有的写请求。这样做是为了让用户很明确的知道内存中的数据和磁盘上的数据已经不一致了，如果下一次RDB持久化成功，redis会自动恢复接收写请求。
1. `stop-writes-on-bgsave-error yes` 如果配置成 `no` 表示你不在乎数据不一致或者有其它手段发现和控制这种不一致，那么在快照写入失败时也能确保redis继续接收新的写入请求。
1. `rdbcompression yes` 对于存储到磁盘的快照进行压缩，redis会采用LZF算法进行压缩，如果你不想消耗CPU来进行压缩的话可以关闭此功能。
1. `rdbchecksum yes` 在存储快照后可以让redis使用CRC64算法进行数据校验，但是这样会增加大约10%的性能消耗，如果希望获取最大的性能提升可以关闭此功能。
1. `dbfilename dump.rdb` 数据快照文件名
1. `dir ./` 数据快照保存目录，默认当前路径

如果需要进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那RDB方式要比AOF方式更加的高效。它的缺点是最后一次持久化后的数据可能丢失。

#### 问题

1. fork一个进程时内存的数据也被复制了，即内存会是原来的两倍。
1. 每次快照持久化都是将内存数据完整写入到磁盘一次，并不是增量的只同步脏数据。如果数据量大而且写操作比较多，必然会引起大量的磁盘IO操作，可能会严重影响性能。
1. 由于快照方式是在一定间隔时间做一次，所以当redis意外down掉时就会丢失最后一次快照后的所有修改。

#### 触发快照的情况

1. 根据配置规则自动快照
1. 用户执行 `save` 或 `bgsave` 命令
  1. 执行save命令时redis会阻塞所有客户端的请求， 然后再同步进行快照操作。
  1. 执行bgsave命令时redis在后台异步进行快照操作，同时还可以响应客户端请求。可以通过 `lastsave` 命令获取最后一次成功执行快照的时间。
1. 执行 `flushall` 命令
1. 执行复制 `replication` 命令

### AOF

默认的AOF持久化策略是每秒钟fsync一次，fsync是指把缓存中的写指令记录到磁盘中，在这种情况下redis仍可以保持很高的性能。

由于OS会在内核中缓存write做的修改，所以可能不是立即写到磁盘上。这样AOF方式的持久化也还是有可能会丢失部分修改。可以通过配置文件告诉redis，通过fsync函数强制OS写入指令到磁盘的时机。

#### 配置

1. `appendonly no` 是否开启AOF
1. `appendfilename "appendonly.aof"` 设置AOF日志文件名
1. `appendfsync everysec` 设置AOF日志同步磁盘的策略，fsync()调用用来告诉操作系统立即将缓存的指令写入磁盘，有三个选项：
  1. `always` 每次写都强制调用fsync，在这种模式下redis会相对较慢，但数据最安全。
  1. `everysec` 每秒调用一次fsync
  1. `no` 不调用fsync()。而是让操作系统自行决定sync的时间。在这种模式下redis性能会更快。
1. `no-appendfsync-on-rewrite no` 设置当redis在rewite的时候是否允许appendsync。因为redis进程在进行AOF重写的时候，fsync()在主进程中的调用会被阻止，也就是redis的持久化功能暂时失效，默认为no这样能保证数据安全。
1. `auto-aof-rewrite-percentage 100` 设置自动进行AOF重写的基准值。也就是重写启动时的AOF文件大小，假如redis自启动至今还没有进行过重写，那么启动时AOF文件的大小会被作为基准值，这个基准值会和当前的AOF大小进行比较。如果当前AOF大小超出所设置的增长比例，则会触发重写。如果设置的值为0则会关闭重写功能。
1. `auto-aof-rewrite-min-size 64mb` 设置一个最小值是为了防止在AOF很小时就触发重写。

AOF方式在同等数据规模的情况下，AOF文件要比RDB文件体积大，因为AOF方式的恢复速度也要慢于RDB方式。

#### 日志恢复

如果在追回日志时恰好遇到磁盘空间满或断电等情况，导致日志写入不完整也没有关系，redis提供了redis-check-aof工具用来进行日志修复

1. 备份受损的aof文件
1. 运行 `redis-check-aof -fix` 进行修复
1. 用diff -u查看两份文件差异以确认问题点
1. 重启redis加载修复后的aof文件

#### 重写

1. 由于AOF采用文件追回方式，这会导致AOF文件起来越大，为此redis提供了AOF文件重写（rewrite）机制，当AOF文件的大小超过所设定的阈值，redis就会启动aof文件的内容压缩，只保留可以恢复数据的最小指令集。
1. 也可以手动使用命令 `bgrewriteaof` 强制要求进行上面的操作。

#### 触发机制

redis会记录上次重写时的AOF大小，假如自启动至今还没有进行过重写，那么启动时AOF文件的大小会被作为基准值，拿它和当前的AOF大小进行比较，如果当前AOF大小超出所设置的增长比例 `auto-aof-rewrite-percentage 100` 则会触发重写。另外你还需要设置一个最小值 `auto-aof-rewrite-min-size 64mb` 防止在AOF很小时就触发重写。

#### 基本原理

1. 在重写开始前redis会创建一个重写子进程，这个进程会读取再有的AOF文件，并将其包含的指令进行分析压缩并写入到一个临时文件中。
1. 与此同时主进程会将新接收到的写指令一边累积到内存缓冲区中，一边继续写入到原有的AOF文件中，这样做是保证原有的AOF文件可用性，避免在重写过程中出现意外。
1. 当重写子进程完成工作后会给父进程发一个信号，父进程收到信号后就会将内存中缓存的写指令追回到新AOF文件中。
1. 当追回结束后redis会用新AOF文件代替旧AOF文件，之后再有新的写指令都会追回到新AOF文件中。
1. 重写AOF文件的操作，并没有读取旧的AOF文件，而是将整个内存中的数据库内容用户命令的方式重写了一个新的AOF文件，这点和快照有点类似。

# 事务

redis的事务就是一组命令的集合，它们被顺序的执行，如果取消事务的执行那么事务中的命令全部都不会执行。

![](/images/redis_transactions2.png)

1. redis的事务仅仅是保证事务里的操作会被连接独占的执行，因为是单线程架构所以在执行完事务内的所有指令前是不可能再去同时执行其他客户端的请求。
1. redis的事务没有隔离级别的概念，因为事务提交前任何指令都不会被实际执行，也就不存在“事务内的查询要看到事务里的更新，在事务外查询不能看到”这种问题。
1. redis的事务不保证原子性，也就是不保证所有指令同时成功或同时失败，只能决定是否开始执行全部指令的能力，没有执行到一半进行回滚的能力。

### 事务的基本过程

![](/images/redis_transactions.png)

1. 发送一个事务的命令 `multi` 给redis；
1. 依次发送要执行的命令给redis，redis接收到这些命令后并不会立即执行，而是放到等待执行的事务队列里面；
1. 发送执行事务的命令 `exec` 给redis。
1. redis保证一个事务内的命令依次执行，而不会被其它命令插入到中间；
1. 如果想放弃事务使用命令 `discard` 。

### 事务过程中的错误处理

1. 如果任何一个命令语法有错，redis会直接返回错误，所有的命令都不会执行。
1. 如果某个命令执行错误，那么其它的命令仍然会正常执行，然后在执行后返回错误信息。
1. redis不提供事务回滚的能力，开发者必须在事务执行出错后自行恢复数据库状态。

### 事务操作的基本命令

1. `multi` 开启事务
1. `exec` 执行事务
1. `discard` 放弃事务
1. `watch` 监控键值。如果键值被修改或删除，后面的一个事务就不会执行。
1. `unwatch` 取消watch

#### Watch说明

1. redis使用watch来提供乐观锁定，类似于CAS（check-and-set）
1. watch可以被调用多次
1. 当exec被调用后所有之前被监视的键值会被取消监视，不管事务是否被取消或者执行。并且当客户端连接丢失的时候所有东西都会被取消监视。

## 示例

```bash
127.0.0.1:6379[2]> set key1 a
OK
127.0.0.1:6379[2]> set key2 b
OK
127.0.0.1:6379[2]> keys *
1) "key1"
2) "key2"
127.0.0.1:6379[2]> multi
OK
127.0.0.1:6379[2]> set key1 aaa
QUEUED
127.0.0.1:6379[2]> set key2 bbb
QUEUED
127.0.0.1:6379[2]> exec
1) OK
2) OK
127.0.0.1:6379[2]> get key1
"aaa"
127.0.0.1:6379[2]> get key2
"bbb"
```

# 发布订阅

> redis的发布订阅模式可以实现进程间的消息传递

![](/images/redis_transactions3.jpg)

### 命令

1. `publish channel message` 往指定频道发送消息，如果频道不存在则发送失败，一定要有客户端订阅你发送的频道。
1. `subscribe channel [channel ...]` 订阅一个或多个频道，只有先订阅才能发送消息
1. `psubscribe pattern [pattern ...]` 访问一个或多个频道，支持glob风格的通配符
1. `unsubscribe [channel [channel ...]]` 取消一个或多个订阅，如果不指定频道表示取消所有subscribe命令的订阅
1. `punsubscribe [pattern [pattern ...]]` 取消一个或多个订阅，如果不指定频道表示取消所有psubscribe命令的订阅。

### 示例

开启两个客户端，分别模拟订阅者和发布者。

#### subscribe

```bash
127.0.0.1:6379> subscribe s1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "s1"
3) (integer) 1
```

此时订阅这边会一直等待

#### publish

我们订阅s1频道并往里面发送一条消息

```bash
127.0.0.1:6379> publish s1 hello
(integer) 1
```

如果返回1表示成功，返回0表示失败

#### 结果

当上面的命令发送成功之后，在订阅者这边可以收到消息。

```bash
127.0.0.1:6379> subscribe s1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "s1"
3) (integer) 1
1) "message"
2) "s1"
3) "hello"
```
