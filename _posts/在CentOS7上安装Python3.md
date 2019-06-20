---
title: 在CentOS7上安装Python3
date: 2019-6-20 15:24:00
tags: 
- centos7
- python3
categories: 
- linux
---

![](/images/python_logo.jpg)

## 启用软件集（SCL）

[软件集合](https://www.softwarecollections.org/en/)也称为SCL，是一个社区项目，允许你在同一系统上构建、安装和使用多个版本的软件，而不会影响系统默认软件包。通过启用软件集，你将可以访问核心存储库中不可用的较新版本的编程语言和服务。

CentOS 7默认安装Python 2.7.5，SCL将允许你安装较新版本的python 3.x以及默认的python v2.7.5。
<!-- more -->
为了启用SCL，我们需要安装CentOS SCL版本文件。它是CentOS extras存储库的一部分，可以通过运行以下命令来安装：

```bash
sudo yum install centos-release-scl
```

## 在CentOS 7上安装Python 3

现在我们可以访问SCL存储库可以安装需要的任何Python 3.x版本。目前，可以使用以下Python 3集合：

- Python 3.3
- Python 3.4
- Python 3.5
- Python 3.6

在本示例中我们将安装Python 3.6，这是撰写本文时可用的最新版本。为此，请在CentOS 7终端上键入以下命令：

```bash
sudo yum install rh-python36
```

## 使用Python 3

`rh-python36` 安装软件包后，键入以下命令检查Python版本：

```bash
python --version
```

```output
Python 2.7.5
```

你会注意到Python 2.7是当前shell中的默认Python版本。

要访问Python 3.6，你需要使用Software Collection `scl`工具启动新的shell实例：

```bash
scl enable rh-python36 bash
```

上面的命令会调用 `/opt/rh/rh-python36/enable` 更改shell环境变量的脚本。

如果再次检查Python版本，你会发现Python 3.6现在是当前shell中的默认版本。

```bash
python --version
```

```output
Python 3.6.3
```

需要指出的是，Python 3.6仅在此shell会话中设置为默认的Python版本。如果退出会话或从另一个终端打开一个新会话，Python 2.7将是默认的Python版本。

## 安装开发工具

构建Python模块需要开发工具，你可以通过键入以下内容来安装必要的工具和库：

```bash
sudo yum groupinstall 'Development Tools'
```

## 创建虚拟环境

Python `Virtual Environments`允许你在特定项目的隔离位置安装Python模块，而不是全局安装。这样你就不必担心影响其他Python项目。

在Python 3中创建新虚拟环境的首选方法是执行 `venv` 命令。

假设我们想 `my_new_project` 在我们的用户主目录和匹配的虚拟环境中创建一个新的Python 3项目。

首先，创建项目目录并[切换](https://linuxize.com/post/linux-cd-command/)到它：

```bash
mkdir ~/my_new_projectcd ~/my_new_project
```

使用该 `scl` 工具激活Python 3.6 ：

```bash
scl enable rh-python36 bash
```

从项目根目录内部运行以下命令以创建名为的虚拟环境 `my_project_venv`：

```bash
python -m venv my_project_venv
```

要首先使用虚拟环境，我们需要输入以下命令来激活它：

```bash
source my_project_venv/bin/activate
```

激活环境后，shell提示符将以环境名称作为前缀：

```sh
(my_project_venv) user@host:~/my_new_project$
```

从Python 3.4开始，在创建虚拟环境[pip时，](https://linuxize.com/post/how-to-install-pip-on-centos-7/)默认情况下会安装Python [的包管理器](https://linuxize.com/post/how-to-install-pip-on-centos-7/)。
