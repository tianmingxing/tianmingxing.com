---
title: 使用ESRally压测ElasticSearch性能
date: 2019-07-02 14:12:00
tags:
- ElasticSearch
- ESRally
- 压力测试
categories:
- 搜索引擎
- ElasticSearch
---

```bash
------------------------------------------------------
    _______             __   _____
   / ____(_)___  ____ _/ /  / ___/_________  ________
  / /_  / / __ \/ __ `/ /   \__ \/ ___/ __ \/ ___/ _ \
 / __/ / / / / / /_/ / /   ___/ / /__/ /_/ / /  /  __/
/_/   /_/_/ /_/\__,_/_/   /____/\___/\____/_/   \___/
------------------------------------------------------

|                         Metric |                 Task |     Value |   Unit |
|-------------------------------:|---------------------:|----------:|-------:|
|            Total indexing time |                      |   28.0997 |    min |
|               Total merge time |                      |   6.84378 |    min |
|             Total refresh time |                      |   3.06045 |    min |
|               Total flush time |                      |  0.106517 |    min |
|      Total merge throttle time |                      |   1.28193 |    min |
|               Median CPU usage |                      |     471.6 |      % |
|             Total Young Gen GC |                      |    16.237 |      s |
|               Total Old Gen GC |                      |     1.796 |      s |
|                     Index size |                      |   2.60124 |     GB |
|                  Total written |                      |   11.8144 |     GB |
|         Heap used for segments |                      |   14.7326 |     MB |
|       Heap used for doc values |                      |  0.115917 |     MB |
|            Heap used for terms |                      |   13.3203 |     MB |
|            Heap used for norms |                      | 0.0734253 |     MB |
|           Heap used for points |                      |    0.5793 |     MB |
|    Heap used for stored fields |                      |  0.643608 |     MB |
|                  Segment count |                      |        97 |        |
|                 Min Throughput |         index-append |   31925.2 | docs/s |
|              Median Throughput |         index-append |   39137.5 | docs/s |
|                 Max Throughput |         index-append |   39633.6 | docs/s |
|      50.0th percentile latency |         index-append |   872.513 |     ms |
|      90.0th percentile latency |         index-append |   1457.13 |     ms |
|      99.0th percentile latency |         index-append |   1874.89 |     ms |
|       100th percentile latency |         index-append |   2711.71 |     ms |
| 50.0th percentile service time |         index-append |   872.513 |     ms |
| 90.0th percentile service time |         index-append |   1457.13 |     ms |
| 99.0th percentile service time |         index-append |   1874.89 |     ms |
|  100th percentile service time |         index-append |   2711.71 |     ms |
|                           ...  |                  ... |       ... |    ... |
|                           ...  |                  ... |       ... |    ... |
|                 Min Throughput |     painless_dynamic |   2.53292 |  ops/s |
|              Median Throughput |     painless_dynamic |   2.53813 |  ops/s |
|                 Max Throughput |     painless_dynamic |   2.54401 |  ops/s |
|      50.0th percentile latency |     painless_dynamic |    172208 |     ms |
|      90.0th percentile latency |     painless_dynamic |    310401 |     ms |
|      99.0th percentile latency |     painless_dynamic |    341341 |     ms |
|      99.9th percentile latency |     painless_dynamic |    344404 |     ms |
|       100th percentile latency |     painless_dynamic |    344754 |     ms |
| 50.0th percentile service time |     painless_dynamic |    393.02 |     ms |
| 90.0th percentile service time |     painless_dynamic |   407.579 |     ms |
| 99.0th percentile service time |     painless_dynamic |   430.806 |     ms |
| 99.9th percentile service time |     painless_dynamic |   457.352 |     ms |
|  100th percentile service time |     painless_dynamic |   459.474 |     ms |

----------------------------------
[INFO] SUCCESS (took 2634 seconds)
----------------------------------
```

在部署完一套ES集群之后，我们肯定想知道这套集群性能如何？是否可以支撑未来业务发展？存不存在性能上的瓶颈？要想有依据的回答这些问题，我们需要通过压力测试结果中找答案。

# 介绍

Rally是Elasticsearch的基准测试框架，由官方提供维护。
<!-- more -->

# 安装

1. 安装Python3.5及以上版本，系统默认可能是2.x版本，如果需要升级请参考《[在CentOS7上安装Python3](https://tianmingxing.com/2019/06/20/%E5%9C%A8CentOS7%E4%B8%8A%E5%AE%89%E8%A3%85Python3/)》。
1. 安装git1.9及以上版本
1. 安装esrally `pip3 install esrally`
1. 配置esrally `esrally configure`，执行此命令后会在当前用户根目录下生成 `.rally` 目录，可以 `ll ~/.rally` 这样来确认。

# 使用

## 快速开始

如果想测试当前机器上某个版本单点ES性能，可以像下面这样：

```bash
esrally --distribution-version=6.5.3

# 同样的如果你想测试其它版本
esrally --distribution-version=6.8.1
```

当执行上面的命令之后会自动下载对应es版本软件，并在本地启动，接着执行测试。这个过程在rally中被称为比赛，而赛道是用默认的，即[geonames](https://github.com/elastic/rally-tracks/tree/master/geonames)。

## 测试远程集群

上面的示例不能测试存在的es集群，下面介绍使用方法：

1. 指定跑道和ES集群地址后就可以执行测试。
	```bash
	esrally --pipeline=benchmark-only \
	 --track=http_logs \
	 --target-hosts=192.168.1.100:9200,192.168.1.101:9200,192.168.1.102:9200 \
	 --report-file=/tmp/report_http_logs.md
	```
1. 执行 `esrally list tracks` 命令可以查看可用跑道（--track）
1. 可以使用 `--report-file=/path/to/your/report.md` 将此报告也保存到文件中，并使用 `--report-format=csv` 将其另存为CSV。

## 修改默认跑道参数

如果直接在默认跑道上修改，会被还原，所以只能通过增加跑道的方式。

1. 在 `.rally/benchmarks/tracks` 下面创建新的赛道，比如 `custom`。
1. 在 `custom/http_logs/challenges/default.json` 文件中调整赛道的配置并保存，例如下面修改default对应的操作，由10个客户端发起，每个用户端发出100次操作。
	```json
	{
	  "operation": "default",
	  "clients": 10,
	  "warmup-iterations": 500,
	  "iterations": 100,
	  "target-throughput": 100
	}
	```
1. 在启动时指定跑道：`esrally --track=custom ....`，例如：
	```bash
	esrally --pipeline=benchmark-only \
	 --track=custom \
	 --track=http_logs \
	 --target-hosts=192.168.1.100:9200,192.168.1.101:9200,192.168.1.102:9200 \
	 --report-file=/tmp/report_http_logs.md
	```

# 小结

1. 关于报告解读部分可以参考[官方文档](https://esrally.readthedocs.io/en/stable/summary_report.html)
1. 本文只介绍了基本使用，相信这些东西可以适用大部分人的需求，更多信息可以阅读上面官方文档。

---
参考文献：
1. https://esrally.readthedocs.io/en/stable/index.html
1. https://github.com/elastic/rally
