---
title: Trac安装和使用
date: 2017-12-14 15:19:21
tags:
- Trac
- Trac安装
categories:
- 项目管理
---

> 在团队中使用Trac管理研发任务已两年左右，是时候总结一下使用体会了。

# 介绍

[Trac](https://trac.edgewall.org/)  是一款Web应用形式的项目管理和缺陷&事务追踪系统，使用Python语言编写，开源并且免费使用。

想必你也听说过一些项目管理软件，比如禅道、TAPD或自研的类似系统等，它们实现的需求类似，主要目的是帮助团队管理开发任务，记录缺陷便于后续跟踪。

通常比较小的技术团队较少去使用到类似的管理软件，可能更多的是用Excel表格记录一下，甚至开发计划存在于脑海中，接到需求之后拿起键盘就是干，它们是决体会不到任务管理的好处的。

## 当初为什么选择使用它？
<!-- more -->
一直想在项目管理上流程化，希望有款软件能够解决这样的问题，最好这款软件是免费的，有评估过禅道，最终发现并不能很好的满足我们的要求。首先它的基本版本虽然免费，但是一些特别实用的功能是收费的，以插件的形式购买并安装，大家都知道领导是希望尽量使用开源（免费）的东西，说白了不愿掏钱（比较穷）。其实它的设计理论我还是比较认同的，敏捷开发的精髓在它上面均有体现，老实讲还挺不容易的，只是真正会使用的人太少了，因为使用的前提是掌握敏捷开发的理论，并且要去实践才能体会。

产品、研发和测试应该是三权分立的，三方都对需求负责，产品平时记录需求到需求池，依据内部的计划选定一些需求放到迭代计划中，并分配给技术部门。研发项目组拿到需求后拆解成开发任务，并为技术人员分配开发任务，研发工作结束后通知测试部门。测试部门前期在产品出发布计划时，已经对这些需求拆解了测试用例，此时按用例进行测试，确认研发的最终成果是否满足所有需求。如果有发现缺陷则记录在案，并及时分配给技术人员解决，最终完成所有缺陷后创建发布版本计划，进行下一阶段的软件发现作业。由此可见三个部门的工作在系统上均有提前，而且是环环相扣，可以追溯。

我是通过一位之前在雅虎中国待过的大神介绍而知晓Trac的，他当时在部门内推动使用Trac，我在使用过程中也存在一些疑惑，不过有大神在还好问题都得到及时的解决，最终我发现这确实是个好东西，所以在之后的工作中力推它。

## 遇到的问题

在整个使用过程中遇到很多问题且是方方面面的，但最终都逐一解决了，下面罗列一些：

1. 安装部署是个大问题
  * 因为之前使用时已经由别人部署完成，所以这方面的问题并没有显露出来，等到后面自己在新的公司推动时才暴露出来。
  * 当时让运维配合安装部署就花了两个星期，在安装过程中出现各种各样的问题，不过还好最终总算是完成了。总之遇到问题就网上搜索相关的解决方案，实在不行可以到谷歌论坛中向作者提问，我有好多问题是通过这种方式最终解决的。
1. 推动不起来
  * 由于技术人员之前没有使用过类似的软件，对敏捷开发的理论也没有深入了解，刚接触时是非常排斥的，觉得这会影响自己的开发效率。
  * 一方面先让技术总监切实感受到Trac的优势，有了他的支持就比较好办了，再举办几次分享会议，全面论述Trac的工作流程，介绍项目管理的必要性及优势。
1. 优势
  * 先有任务再有行动
  * 我们在团队中使用时强制必须见单作业，在做开发时必须先创建任务单，任务单上准确完整地描述要做的事情。
1. 先设计后计划最后开发
  * 有了任务单你还不能马上就开发做，根据任务的复杂程序，你要先在任务上设计出实现方案，对于复杂的有风险的任务，要对你设计的方案作验证。只要有这些都完成了，你才能真正的开发动代码。
  * Trac通过安装插件可以完美直接UML图的绘制，也就是说可以直接通过简单的文字描述生成图片，可以说非常的方便。
1. 历史工作可还原
  * 这有一个前提要求，那就是在当时工作时一定要按照详细的流程来走，做好开发时的工作留痕，包括但不限于方案的设计和验证结果，同事间的讨论，接口的定义，数据库结构设计等等。  * 一旦出现员工离职，或接手其它同事的项目时，除了代码之外还能够有当初开发的设计文档，我们在这方面有尝试到甜头，深知它的作用非常巨大。

# 安装

主要分成两个部分，一是Python环境的安装，二是Trac软件的安装及它的配置。

## 准备好环境

> 演示环境 centos7

1. Python（≥ 2.6 and < 3.0）
    ```bash
    # 建议使用yum直接安装，方便且快捷。
    yum -y install python

    # 如果不小心安装到Python3版本了，那可以去下载Python2的源码包安装。https://www.python.org/downloads/
    # 解压下载的文件并进入其目录逐条执行下面的语句
    ./configure
    make
    make install
    ```
1. 安装pip和setuptools（≥ 0.6）
    ```bash
    # 通过这个脚本能够自动帮我们安装好这两个软件，确保下载的get-pip.py有执行权限。
    wget -c https://bootstrap.pypa.io/get-pip.py
    python get-pip.py

    # （备选方案）如果安装之后使用上有问题，就再安装一次，这次指定一下安装目录。
    python get-pip.py --prefix=/usr/local/
    ```
1. Genshi（≥ 0.6）
    ```bash
    pip install Genshi

    # （备选方案）源码下载地址https://genshi.edgewall.org/wiki/Download
    ```
1. 任意一种数据库（SQLite、PostgreSQL和MySQL）,为了方便测试暂时使用SQLite。
    ```bash
    # sqlite3模块通常已经包含在Python中了，可以通过下面的方式检查一下，不报错即可为成功。
    [root@localhost ~]# python
    Python 2.7.5 (default, Nov  6 2016, 00:28:07)
    [GCC 4.8.5 20150623 (Red Hat 4.8.5-11)] on linux2
    Type "help", "copyright", "credits" or "license" for more information.
    >>>
    >>> import sqlite3
    >>> quit()
    ```

## 安装Trac

* 通过PIP工具安装比较简单，参照下面的示例。
    ```bash
    pip install trac
    # 下面两个也可以一并安装上
    pip install trac psycopg2
    pip install trac mysql-python
    ```

## Trac的配置

### 小试牛刀

* 创建一个全新的Trac项目
    ```bash
    adduser trac
    su - trac
    trac-admin example_project initenv
    # 输入项目的名称
    # 回车默认使用sqlite数据库
    # 稍等几秒可创建完
    ```
* 启动项目`tracd --port 8000 /home/trac/example_project`
* 浏览器访问 `http://[IP]:8000/example_project`

### 继续配置

* 中文支持
    ```bash
    # Trac已经支持国际化，可以安装中文语言包。
    pip install Babel
    ```
    * 安装完重启一下服务则可以看到汉化之后的效果。
    * 用户管理
    ```bash
    # 我们需要为系统默认一位管理员，用来进行系统设置等操作。
    cd example_project/conf/
    htpasswd -c .htpasswd admin # 回车并输入admin用户的密码

    # 将admin用户在Trac系统中的权限设置为最高
    cd ../../
    trac-admin example_project permission add admin TRAC_ADMIN

    # 如果你想添加其它普通用户，可以在.htpasswd文件中追回新的用户。
    htpasswd example_project/conf/.htpasswd zhangshan

    # 为了让Trac采用我们创建的账号文件，在启动服务的时候要进行明确指定。
    tracd --port 8000 --basic-auth="example_project,example_project/conf/.htpasswd,测试环境"  example_project
    ```
* 用上面创建的admin管理员账号进行登录，这时会发现在导航栏上多出管理菜单，单击进入可以对系统进行相应的配置。

# 使用

软件已经安装完成，如何将它与自己的开发相结合呢？答案就是基于任务单的驱动，将任务单涵盖到整个开发过程中，不管是开发计划的制定，技术方案的设计及验证，以及具体的某个功能或需求点的开发。可以理解为不管做什么事情，都得先创建一个任务单，在单上有明确的所属项目名称、迭代版本号、任务的简要和详述以及任务的完成时间。

## 任务单管理

创建任务单非常简单，如果你已经按照上面中文化Trac之后，可以在导航栏上找到 `新建任务单`，单击会打开新页面以填写任务单的信息。
  1. 类型` 标识任务单的用途，你可以在管理后台中自由配置，比如缺陷、开发、设计、事务和测试等。
  1. 优先级` 可以自定义任务单的紧急程度，以便于程序员在领取任务时知道什么单应该排前面做，不太紧急的任务可以排到后面做。
  1. 里程碑` 当你在做一件耗时比较长的工作时特别有用，比如项目有一次较大的功能迭代或有一个可以划分成多阶段来做的事情。可以想像一下马路上的路碑，间隔一定距离就有一块标识当前坐标距离前后地点多少km的路碑，在一条路上的里程碑肯定是固定的，当你走过所有的路碑时也意味着你走完了这条路。在项目开发中也可以应用这样的思想，比如你要开发一个项目，但是这个项目开发周期比较长，要开发很长一段时间才能看到效果，不能对开发人员起到一个激励的作用，总感觉事情做也做不完。如果在项目开发你就把这个大项目分成多个阶段，每个阶段树立一块里程碑，那么只要这些里程碑都越过时，即意味着项目开发的结束。这是一个很好用的项目管理方法，当你有看得见的目标时，你才会朝着它不断前进。
  1. 版本` 每一次的迭代过程即代表未来会有一个对应的版本发布，在当前创建的任务单应该也必须有所版本，版本和里程碑可以相关包含，这在使用上是非常灵活的，就看你的实际情况。

谁创建任务单谁就是任务单的负责人，他应该关注任务的状态，自己创建的任务单不一定非要自己处理，可以指派给他人处理，也可以由他人自己领取。任务的所属人即是任务的执行者，负责实现任务所描述的内容。同时也可以二次指派给其他人处理，但报告人始终是同一个人，不可改变。

如果用户信息中有配置邮箱地址，并且启用了当任务单状态变更时通知功能，则任务单状态变更时会收到邮件提醒，当然还有一个前提那就是需要在Trac中配置邮箱信息，可以配置腾讯企业邮箱，当然公司内部邮箱也是可以的。

我们在任务单上除了创建时填写的描述信息，也可以为任务单添加评论，在这些评论中可以记录完成当前任务所做的所有故事，团队成员可以参与到任务单中讨论，相当于工作留痕，方便后面可逆源。

## 插件安装

插件是一个为Trac增强功能的东西，可以让你在管理任务过程中得心应手，下面介绍几款插件：

1. <a href="https://trac-hacks.org/intertrac/FullBlogPlugin" title="FullBlogPlugin">FullBlogPlugin</a> 博客
1. <a href="https://github.com/microsoft-mobile/trac-multiproject" title="MultiProject">MultiProject</a> 支持多项目
1. <a href="https://trac-hacks.org/wiki/PlantUmlMacro" title="PlantUmlMacro">PlantUmlMacro</a> UML图生成插件，其实更准确的叫宏。它的实现比插件还小，一般用在编辑器增强。

更多的插件列表见官方插件中心 <code>https://trac-hacks.org/wiki/HackIndex</code>

* 下面介绍插件的安装方法
  1. 找到插件的介绍页面，在插件列表中很容易找到
  1. 找到svn源码地址，检出在某个目录下面
  1. 进入插件的安装目录，也就量有setup.py文件的那一级，输入 `python setup.py bdist_egg` 并回车，在当前目录下生成了一个dist目录，复制此目录内的egg文件到example_project/plugins/下面，然后重启一下服务即可。
  1. 具体的安装过程在插件介绍页面都有写，有些插件安装之后还需要配置，你在安装某个插件时具体的参考文档即可。

我们在实践中尝试过不少插件，经过试用觉得以下还不错，可以供大家参考：

|名称 |版本 |
|-----|----|
|BackLinksMacro |7.0.dev0|
|BlackMagicTicketTweaks |0.12.2 |
|BreadCrumbsNav |0.3dev-r0 |
|cc-selector |0.0.4 |
|CodeExampleMacro |1.2.post0 |
|Color |r11892 |
|DataSaverPlugin |2.0.dev0 |
|GroupTicketFields |0.0.1dev-r0 |
|NoteBox |1.0.post0 |
|PlantUML |2.1.dev0 |
|SimpleMultiProject |0.5.2.dev0 |
|TablePlugin |0.2.post0 |
|TicketCalendarPlugin |0.12.0.2 |
|TracAccountManager |0.5.dev0 |
|TracCodeReviewer |1.0.0.dev0 |
|TracCustomFieldAdmin |0.2.12 |
|TracDynamicFields |2.2.0 |
|TracExtractUrl |0.3 |
|TracHtmlNotificationPlugin |0.12.0.1 |
|TracMarkdownMacro |0.11.4.post0 |
|TracMermaid |0.4.1 |
|TracMindMapMacro |1.2.0.dev0 |
|TracNumberedHeadlinesPlugin |0.4.post0 |
|TracPermRedirect |3.0 |
|TracSectionEditPlugin |1.2.0.dev0 |
|TracTags |0.9.dev0 |
|TracTicketTemplate |1.0.dev0 |
|TracWikiExtras |1.0.dev0 |
|TracWorkflowAdmin |0.12.0.3 |
