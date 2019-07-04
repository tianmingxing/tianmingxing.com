---
title: 介绍如何使用Apache JMeter测试系统性能
date: 2019-07-02 15:54:00
tags:
- JMeter
- 压力测试
categories:
- 测试
---

![](/images/apache_jmeter_logo.png)

# 概述

JMeter™是Apache基金会的一款纯Java开源软件，它最初是为测试Web应用程序而设计的，但后来扩展到其他测试功能。它可用于测试静态和动态资源以及Web动态应用程序的性能。

## 环境要求

1. JMeter4.0与Java8或Java9兼容。出于安全性和性能原因，官方强烈建议你安装这些版本的最新版本。
1. 虽然你可以使用JRE，但最好安装JDK以测试HTTPS，因为JMeter需要JDK的keytool程序。

## 安装

1. 建议下载[最新版本](http://jmeter.apache.org/download_jmeter.cgi)，当前最新版本为 `5.1.1`。
1. 下载zip或tar包后解压就算安装了，记住安装路径上不能包含任何空格。
1. 可以重命名父目录（apache-jmeter-5.1.1），但不要更改里面子目录的名字。
1. 运行 `bin/jmeter.bat` 可以打开GUI界面。

<!-- more -->
## 可选配置

* GUI界面语言切换到中文
  1. 打开 `jmeter.properties` 文件，找到 `language` 和 `locales.add` 属性，并修改其值为 `zh_CN`。
  1. 关掉并重新打开JMeter
* 设置JMeter环境变量
  1. 在安装目录中的 `bin` 目录下创建名为 `setenv.bat` 的文件。
  ```bat
  rem This is the content of bin\setenv.bat,
  rem it will be called by bin\jmeter.bat

  set JVM_ARGS="-Xms1024m -Xmx1024m -Dpropname=value"
  ```
  1. 如果是在Linux系统上应该创建名为 `setenv.sh` 的文件。
  ```bash
  # This is the file bin/setenv.sh,
  # it will be sourced in by bin/jmeter

  # Use a bigger heap, but a smaller metaspace, than the default
  export HEAP="-Xms1G -Xmx1G -XX:MaxMetaspaceSize=192m"

  # Try to guess the locale from the OS. The space as value is on purpose!
  export JMETER_LANGUAGE=" "
  ```
  1. 也可以在执行测试同时指定环境变量，例如 `JVM_ARGS="-Xms1024m -Xmx1024m" jmeter -t test.jmx`。
* 类加载路径设置
  1. JMeter自动从 `JMETER_HOME/lib` 和 `JMETER_HOME/lib/ext` 目录中的jar包中查找类。如果需要添加其它jar包请放到第一个目录中，因为第二个目录是用来放JMeter组件和插件的。
  1. 可以在 `jmeter.properties` 中定义属性 `search_paths` 来设置JMeter插件加载路径，多个路径之间用 `;` 分隔。如果是其它jar包建议把路径配置在 `user.classpath` 属性上，这里的其它包指的是非JMeter组件和插件包，例如JDBC驱动包等。

## 使用流程

不管怎样基于什么场景来测试，在使用JMeter时基本遵循下面的流程：
1. 构建测试计划
  * 建议在GUI模式下运行JMeter，可以通过两种途径构建测试计划，一是手动构建，二是[模板录制](https://jmeter.apache.org/usermanual/get-started.html#template)。
  * 过程中可以通过菜单对计划进行调试。
    1. `Run` → `Start no pauses`
    1. `Run` → `Start`
    1. `Validate` on `Thread Group`
1. 运行计划
  * 一旦测试计划准备就绪就可以开始运行测试。第一步是配置将运行JMeter的injectors，这与任何其他负载测试工具一样，包括：
    1. 根据CPU、内存和网络调整机器大小
    1. 操作系统调整
    1. Java设置：确保安装JMeter支持的最新Java版本
    1. 增加Java堆大小。默认情况下JMeter以1GB的堆运行，可以根据你的测试计划和要运行的线程数进行调整。
  * 一切准备就绪后使用CLI模式运行它以进行负载测试。记住不要使用GUI模式运行负载测试！
  * 使用CLI模式可以生成包含结果的CSV（或XML）文件，并让JMeter 在Load Test结束时生成HTML报告。默认情况下JMeter会在运行时提供负载测试的摘要。
  * 你还可以在测试期间使用Backend Listener获得实时结果。
1. 分析测试结果
  * 运行测试完成后可以使用HTML报告分析。 


# 实战

以下是在JMeter单点环境下试验得出的结论，后续我会再单独介绍《如何使用JMeter分布式压测》。

## 压测WEB应用

需求：压测网站 `http://example.com/` 首页查看吞吐量多少。

### 1. 构建测试计划

在计划下添加 `Thread Group` 以设置请求客户端数量等信息，下面的配置表示将有5个客户端在1秒内初始化完，循环发起20次请求，那总的请求量：5*20=100个请求。

![](/images/jmeter_create_thread_group.gif)

在线程组下面添加 `HTTP Request Defaults` ，也就是对本次请求的公共参数进行设置，这样后续的请求不用重复设置相同的参数。

![](/images/jmeter_create_http_request_defaults.gif)

浏览器在访问某个网站时会启用cookie管理功能，我们也尝试着添加 `HTTP Cookie Manager` 以启动cookie。并不需要作特别的配置只要添加在线程组下面就可以生效。

![](/images/jmeter_create_http_cookie.gif)

接下来配置你要请求的链接，在这里应该是网站首页，即 `http://example.com/`，由于我们在 `HTTP Request Defaults` 中添加了域名信息，此时只需要在Path参数处填写 `/`。

![](/images/jmeter_create_http_request.gif)

压测之后肯定想看看结果怎么样，所以我们接着添加Listener，在这里我添加的是 `Graph Results` 和 `Aggregate Report`。

![](/images/jmeter_create_listener.gif)

### 2. 运行计划

执行当前测试计划的方式有三种：

1. 单击菜单 Run => Start 或者 Start on pauses
1. 也可以单击工具栏绿色“播放”按钮
1. 使用快捷键 `ctrl + R`

### 3. 分析测试结果

由于我们在测试计划中添加了Listener，所以可以在那两个计划中单击查看报告详情，报告是可以保存到指定文件中的。

![](/images/jmeter_result.gif)

在测试过程中可以看到实时报告，最终报告上表明该网站QPS在13左右。

#### 生成可视化HTML报告

![](/images/jmeter_report_html.jpg)

要生成上面更加友好的报告也很简单，思路是先生成csv文件，然后通过对这份文件的分析处理，最后自动生成HTML呈现的报告。

1. 在测试计划中添加两个Listener `Summary Report`， `Simple Data Writer`。
1. 分别设置csv文件的保存位置（设置相同的文件），例如 `F:\jmx\jmeter_report.csv`。
1. 打开 `bin/reportgenerator.properties` 文件复制全部内容追加到 `bin/user.properties` 文件中。
1. 运行测试计划。
1. 可以检查你设置的csv文件是否有生成。
1. 执行命令 `jmeter -g F:\jmx\jmeter_report.csv -o F:\jmx\report`，按照你的实际情况修改文件路径。
1. 检查在 `F:\jmx\report` 中是否生成了一堆文件，找到 `index.html` 并打开它。

### 本部分小结

事实上在实际项目中，可能一次压测计划会包含多个请求，因为仅压单一页面不能很准确的反映系统真实性能，一般会以某个业务场景为单位来实施。比如压测注册功能，那包含访问到注册页面、填写相关信息，调用接口实时校验这些信息，最后再提交表单。通过这样一个场景的压测，可以很真实的反映出这个系统最大可以并发支持多少用户注册。

在上面的案例中不再是简单的请求某个页面，还需要提交数据，这些数据的来源可以随机也可以来自某份文件。那这些配置都可以在官方文档上找到，本文没有对这些进行详细的介绍。

有一些有用的官方示例介绍：
1. [怎样登录一个网站](https://jmeter.apache.org/usermanual/build-web-test-plan.html#logging_in)
1. [使用URL重写处理用户会话](https://jmeter.apache.org/usermanual/build-adv-web-test-plan.html#session_url_rewriting)

## 在Linux环境下执行测试计划

JMeter可以在Linux环境下以命令行（CLI）模式运行，将上面的测试计划保存成 `test_plan.jmx` 文件，并将这个文件上传到服务器上。

像下面这样启动JMeter并执行指定测试计划：

```bash
./bin/jmeter -n -t /tmp/test_plan.jmx

# 选项n表示不以GUI模式启动，t表示指定测试计划文件
```

如果你想看到测试报告，可以在添加每个Listener时指定写入文件，像下面这样：

![](/images/jmeter_write_file.jpg)

这样当计划执行完成后会把报表写入你指定的文件中，之后可以用上面介绍的方法基于csv文件生成HTML形式的报告。

## 压测ElasticSearch性能

测试目标：压测某机器配置下ES集群的读写性能。

### 思路

基准测试应当以最简单的操作进行，尽量减少由于外部因素干扰。下面规划了两种基本操作：

1. 写
```bash
POST test/_doc
{
  "name": 1
}
```
1. 精确搜索
```bash
POST test/_search
{
  "query": {
    "term": {
      "name": {
        "value": 1
      }
    }
  }
}
```

### 构建测试计划

1. 添加 `Add => Config Element => HTTP Header Manager` 并设置Content-Type为application/json，这一步很重要。
1. 添加 `Add => Config Element => HTTP Request Defaults` 
1. 之后再添加两个请求以及报告，方法同上面，这里不再赘述。

### 运行计划

执行计划方法同上。在压测过程中可以不断调整压力，直到达到被测系统性能瓶颈，注意同时还要关注CPU、内存、磁盘以及带宽的使用情况。


---
参考文献：
1. https://jmeter.apache.org/usermanual/get-started.html
