---
title: 利用redis实现资源分布式锁
date: 2019-08-11 11:19:00
tags:
- redis锁
- 分布式锁
categories:
- redis
- 分布式
---

![](/images/distributed_lock.png)

# 源起

## 为什么需要分布式锁？

要回答这个问题先要搞清楚为什么需要锁？可以试想一下当多个线程同时访问临界资源引起竞争后肯定会导致错误或其它意外行为，这时通过锁这种机制可以避免出现这样的问题，因为锁的存在让多个线程有序的访问临界资源。如果在同一个JVM环境下，本地锁可以满足使用目的，但如果是跨JVM呢？

锁只存在于某一个地方肯定不行，锁本身也需要共享出去，其它JVM才能知道系统整体的锁定情况，否则锁定将失效，因为有的可以锁定了，但有的却没有。这也是为什么需要分布式锁的原因，对于跨JVM、跨物理机器的系统需要能够共享的分布式锁。

## 使用分布式锁的场景有哪些？

在实际项目中需要被锁定的资源有很多，不单单只是某个变量而已，往大了看在分布式（微服务）体系下任何资源都可能有被锁定的需求，因为你肯定不想同样的事情被重复处理，特别是这些处理是昂贵的。到底有哪些业务呢？还是举一些常见的例子：

1. 在电商平台中对于订单的处理由多个内部服务，多个节点处理，而订单的处理状态是有严格方向的，每当处理完当前业务，就应该（也只能）进入下一个状态，出不能被重复处理，并且更重要的是同一时刻应该只能有一个系统在进行处理，它是有这样一个严格的**顺序执行**场景的。你要在内部那么多系统和N多节点中保证前面提到的这个要求，那肯定需要引入分布式锁的支持，当前正在处理的系统或节点在处理某个订单前，要先获取它的分布式锁，自己处理完再释放锁。后续系统在处理时也应该是同样的操作，也就是保证它的顺序执行。
1. 某个后台任务需要对一张表的数据进行处理，为了提高处理效率而启用多线程或多进程（多次启动程序）执行方案，但由于是同张表也就会发现同时处理相同数据的情况，而每条数据的处理是昂贵的，那如何避免这个问题呢？其中一种方案是引入分布式锁的支持来解决，我们可以让程序随机获取表中的数据，这样大大减少获取同条数据的概念，但不可能完全避免，再加上在处理每条数据时要先获取对应的分布式锁，如果获取不到则随机到下一条数据，这样就可以实现前面的要求。

实际需要应用分布式锁的场景还有很多，这样就不再列举了。要判断是否需要分布式锁的一条原则：是否出现跨系统访问临界资源的情况，基本上达到这个要求就需要。

<!-- more -->
## 如何实现分布式锁

锁必须被共享出去（大家都可以访问到），而且能够安全的获取或释放。

![](/images/distributedlocks_01.png)

* 利用MySQL MyISAM引擎表锁功能，当客户端1获取到锁时，其它客户端获取不到，除非前者释放锁。ActiveMQ的主从部署方案之一就是利用的这个方案，当主服务不可用时会释放锁，此时从立即获取表锁并且提供服务。
* 利用Redis STNX操作，当key不存在时可以保存成功（返回1），存在则无操作（返回0），由于redis是单进程原子操作，并发线程同时操作redis也会变成顺序操作，最终肯定有一个唯一线程得到了成功操作的返回。
* 自研开放接口，可以通过HTTP或TCP形式开放，实现上面的要求。

# Redis锁

## 介绍

Redis是一个理想的选择。作为轻量级内存数据库，它具有快速，事务性和一致性等特点，这是我们分布式锁所需的关键特性。

锁本身很容易，因为它只是redis数据库中的一个key。那如何设置锁定状态呢？可以使用**SET**命令，然后判断刚才设置进去的key是否存在来判断是否锁定了该资源，不过如果使用两条命令来达到目的，显然会因为网络延时造成**假锁**的问题，这在线程并发激烈时最容易出现。

好在redis提供了**SETNX**命令，它可以直接满足我们的要求，也就是不需要像刚才那样再发一条**GET**操作进行确认。

### 示例

```bash
redis> SETNX mykey "Hello"
(integer) 1
redis> SETNX mykey "World"
(integer) 0
redis> GET mykey
"Hello"
redis> 
```

### 返回值

命令返回整数：

1. `1` 如果key设置成功
1. `0` 如果key设置失败，可以认为获取锁失败。

## 应用

在实际项目中使用时也不用自己来封装锁的操作，redis官方有推荐java的第三方库[redisson](https://github.com/redisson/redisson)已经实现了锁的操作，我们拿来即用。

在redisson中实现了8种类型的锁，它们都可以用来满足分布式锁的需要，其中**RedLock**实现了[Redlock](https://redis.io/topics/distlock)锁定算法，它将多个RLock对象分组并将它们作为一个锁处理，每个RLock对象可以属于不同的Redisson实例。

```java
RLock lock1 = redissonInstance1.getLock("lock1");
RLock lock2 = redissonInstance2.getLock("lock2");
RLock lock3 = redissonInstance3.getLock("lock3");

RedissonRedLock lock = anyRedissonInstance.getRedLock(lock1, lock2, lock3);
// locks: lock1 lock2 lock3
lock.lock();
...
lock.unlock();
```

至于其它锁根据自己的需要进行选用即可，具体用法可以访问链接的官方文档。

---
参考文献：
1. https://redis.io/commands/setnx
2. https://engineering.gosquared.com/distributed-locks-using-redis
3. https://dzone.com/articles/distributed-java-locks-with-redis
4. https://github.com/redisson/redisson/wiki/8.-Distributed-locks-and-synchronizers
5. https://www.baeldung.com/redis-redisson