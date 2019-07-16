---
title: 在CentOS上使用CGroups隔离硬件资源
date: 2019-07-03 16:30:00
tags:
- CGroups
- docker
- 资源隔离
categories:
- Linux
---

![](/images/cgroups.png)

CGroups提供了进程级别的资源（CPU、内存、网络和磁盘等）分配机制，也就是可以限制某一个或者某一组进程的资源使用。

**为什么需要这么一种技术呢？**

如果你了解过docker，那应该知道容器之间是相互安全隔离的，它的底层实现就是采用的CGroups技术。资源隔离是必要的，在同一台机器上运行着非常多的进程，如果这台机器资源是共享给多个用户在使用，你肯定不想因为某个用户的程序负载过大而影响到所有其它用户，这就需要资源安全隔离，避免发生级联错误。再一个也为了保持大家都公平的使用资源，而不会出现一方过多或另一方过少的情况。

本文不介绍更深层次的实现原因，会把重点放在CGroups使用层面，下面会举个例子详细介绍。
<!-- more -->

# 限制CPU使用率

## 准备一段程序

将下面c代码保存到文件 `cputime.c` 之中。

```c
void main() {
  unsigned int i, end;
  end = 1024 * 1024 * 1024; 
  for(i = 0; i < end; ) {
    i ++;
  }
}
```

用gcc命令进行文件编译，如果没有这个命令则使用 `yum install -y gcc*` 进行安装即可。

```bash
gcc cputtime.c -o cputime
```

执行完上面的命令后会在当前目录生成新可执行文件 `cputime`，到这里我们的测试程序就准备完了。

## 创建控制组

我们将创建一个名称为 `cpu_cputime` 的控制组，在里面会自动生成cpu控制文件，修改里面的参数就可以达到限制cpu资源的目的了。

```bash
sudo cgcreate -t tianmingxing:tianmingxing -g cpu:/cpu_cputime
```

命令执行之后可以在目录 `/sys/fs/cgroup/cpu/cpu_cputime` 下看到上面创建的控制组。

```bash
$ ll /sys/fs/cgroup/cpu/cpu_cputime/
total 0
-rw-rw-r-- 1 root         root         0 Jul  3 05:08 cgroup.clone_children
--w--w---- 1 root         root         0 Jul  3 05:08 cgroup.event_control
-rw-rw-r-- 1 root         root         0 Jul  3 05:08 cgroup.procs
-r--r--r-- 1 root         root         0 Jul  3 05:08 cpuacct.stat
-rw-rw-r-- 1 root         root         0 Jul  3 05:08 cpuacct.usage
-r--r--r-- 1 root         root         0 Jul  3 05:08 cpuacct.usage_percpu
-rw-rw-r-- 1 root         root         0 Jul  3 05:08 cpu.cfs_period_us
-rw-rw-r-- 1 root         root         0 Jul  3 05:08 cpu.cfs_quota_us
-rw-rw-r-- 1 root         root         0 Jul  3 05:08 cpu.rt_period_us
-rw-rw-r-- 1 root         root         0 Jul  3 05:08 cpu.rt_runtime_us
-rw-rw-r-- 1 root         root         0 Jul  3 05:08 cpu.shares
-r--r--r-- 1 root         root         0 Jul  3 05:08 cpu.stat
-rw-rw-r-- 1 root         root         0 Jul  3 05:08 notify_on_release
-rw-rw-r-- 1 tianmingxing tianmingxing 0 Jul  3 05:08 tasks
```

## CPU参数配置

不建议直接修改上面列出的文件内容，请使用下面的命令进行修改。

```bash
sudo cgset -r cpu.cfs_quota_us=10000000 cpu_cputime
```

如果要查询某个控制组下面的配置，可以采用下面的命令：

```bash
$ cgget -g cpu:cpu_cputime
cpu_cputime:
cpu.rt_period_us: 1000000
cpu.rt_runtime_us: 0
cpu.stat: nr_periods 686341
	nr_throttled 7886
	throttled_time 702560123598
cpu.cfs_period_us: 1000000
cpu.cfs_quota_us: 10000000
cpu.shares: 1024
......
```

### 参数介绍

`cfs_quota_us` 和 `cfs_period_us` 参数单位是微秒，其中前者是指一个周期内总的可用运行时间，后者是指一个周期的长度，它们可以配合起来限制CPU的运行时间，下面列举几组例子让大家更容易理解。

1. `cpu.cfs_quota_us=250000`、 `cpu.cfs_period_us=250000` 如果period为250ms且quota为250ms，那么该控制组每250ms将获得1个CPU运行时间。
1. `cpu.cfs_quota_us=1000000`、 `cpu.cfs_period_us=500000` 如果period为500ms且quota为1000ms，那么该控制组每500ms将获得2个CPU运行时间。
1. `cpu.cfs_quota_us=10000`、 `cpu.cfs_period_us=50000` 如果period为50ms且quota为10ms，那么该控制组每50ms将获得1个CPU的20%运行时间。

设想一下在32核CPU下，如果要限制某个进程占用8个CPU运行时间，该怎么配置呢？通过上面的例子举一反三，思考一下相信可以得出答案。

| 核数 | 配置                                                         |
| ---- | ------------------------------------------------------------ |
| 2核  | `cgset -r cpu.cfs_quota_us=1000000 cpu_2c ` `cgset -r cpu.cfs_period_us=500000 cpu_2c` |
| 4核  | `cgset -r cpu.cfs_quota_us=2000000 cpu_4c` `cgset -r cpu.cfs_period_us=500000 cpu_4c` |
| 8核  | `cgset -r cpu.cfs_quota_us=4000000 cpu_8c` `cgset -r cpu.cfs_period_us=500000 cpu_8c` |
| 16核 | `cgset -r cpu.cfs_quota_us=8000000 cpu_16c` `cgset -r cpu.cfs_period_us=500000 cpu_16c` |

## 测试程序

### 不限制资源的情况下

使用 `time` 命令可以为我们报告程序执行消耗的时间，其中的 `real` 就是我们真实感受到的时间。

```bash
$ time ./cputime

real	0m3.153s
user	0m3.152s
sys	0m0.001s
```

### 限制资源的情况下

将控制组限制为1个CPU的20％。

```bash
sudo cgset -r cpu.cfs_quota_us=10000 cpu_cputime
sudo cgset -r cpu.cfs_period_us=50000 cpu_cputime
```

然后我们再执行程序，不过本次要在cgroup中运行应用程序：

```bash
# 其实很简单，把要执行的命令整体放到后面即可
sudo cgexec -g cpu:/cpu_cputime time ./cputime

real	0m10.681s
user	0m10.680s
sys	0m0.001s
```

如果以普通用户执行上面命令，可能会提示权限不足等一些问题，此时可以对控制组目录进行授权。

```bash
sudo chown -R tianmingxing:tianmingxing /sys/fs/cgroup/cpu/cpu_cputime
```

可以使用下面的命令检查程序是否在所需的cgroup中运行，把pid替换成真实进程编号：

```bash
$ ps -o cgroup [pid]
CGROUP
11:pids:/user.slice,10:devices:/user.slice,8:blkio:/user.slice,6:memory:/memory_4g,3:cpuacct,cpu:/cputime,1:name=systemd:/user.slice/user-1007.slice/session-7250.scope
```

从上面 `cpu:/cputime` 中可以看到，这个进程确实是在我们定义的cgroup中运行。

# 限制内存

同样的像下面这样设置内存最大使用量11G，这个值是由 `1024 * 1024 * 1024 * 11` 计算出来的，你可以根据需求自己调整。


```bash
sudo cgcreate -t tianmingxing:tianmingxing -g memory:/memory_cputime
sudo cgset -r memory.limit_in_bytes=11811160064 memory_cputime

# 如果要查询内存所有参数设置，可以使用下面的命令：
$ cgget -g memory:memory_cputime
memory_cputime:
memory.kmem.tcp.max_usage_in_bytes: 0
memory.kmem.tcp.failcnt: 0
memory.kmem.tcp.usage_in_bytes: 0
memory.kmem.tcp.limit_in_bytes: 9223372036854771712
memory.memsw.failcnt: 0
memory.limit_in_bytes: 11811160064
memory.memsw.max_usage_in_bytes: 11811291136
memory.memsw.usage_in_bytes: 11809599488
memory.kmem.max_usage_in_bytes: 0
memory.kmem.failcnt: 0
memory.kmem.usage_in_bytes: 0
......
```

# 小结

1. 可以通过 `top` 系统命令并查看 `%CPU` 和 `%MEM` 列确认目标进程资源是否真的被限制住了，这个百分比只表示单核，如果机器是多核就乘以核数。
1. 如果某个控制组已经应用在一个进程上了，那再次使用该控制组将中断原先的进程，你可以通过创建多个控制组来解决这个问题。
1. 其它资源的限制方法和上面是类似的，大家可以尝试一下，本文不再赘述。

---
参考文献：
1. http://www.manongjc.com/article/7475.html
1. https://www.linuxjournal.com/content/everything-you-need-know-about-linux-containers-part-i-linux-control-groups-and-process
1. https://tech.meituan.com/2015/03/31/cgroups.html
1. https://www.ibm.com/developerworks/cn/linux/1506_cgroup/index.html
1. https://www.cnblogs.com/sparkdev/p/8052522.html
1. https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt
