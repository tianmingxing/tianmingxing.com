---
title: 自定义shell提示符
date: 2019-08-05 14:31:00
tags:
- shell
- centos
categories:
- linux
---

当你登录到一台linux机器后，屏幕会显示shell提示符，它默认由主机名和当前工作目录组成。我们可以更改显示的字符颜色及它的组成部分，通常前者是有用的，可以方便你肉眼区分自己和系统的输出。

在linux系统中是通过环境变量**PS1**来控制的，只要修改变量对应的值就可以更改上面提到的东西，你可以执行命令 `echo $PS1` 查看当前该变量设置。它的值是由有意义字符所组成的，你必须掌握这些字符的含义才能手动设置，当然这肯定会比较痛苦，下面推荐几个网站可以在线设置。

## 在线生成PS1字符

1. [bashrcgenerator](http://bashrcgenerator.com/)
1. [HalloweenBash](https://xta.github.io/HalloweenBash/)
1. [ezprompt](http://ezprompt.net/)
1. [ps1gen](http://omar.io/ps1gen/)

## 怎样设置

如果只想让当前登录用户生效，可以执行命令 `vim ~/.bashrc` 或 `vim ~/.bash_profile` 编辑环境变量文件，在里面添加下面信息：

```bash
export PS1="\h:\W \u\\$ "
```

保存上面编辑的文件之后再执行命令 `source ~/.bashrc` 或 `source ~/.bash_profile` 就可以看到效果了。如果不执行这个命令你需要退出当前终端，再次进入时也可以看到效果。

如果想让所有用户都应用，则可以把上面的信息放到 `/etc/profile` 文件中即可。

## 优秀样式推荐

在网上找了一些经典样式供大家选用：

```bash
PS1="\[\033[38;5;11m\]\u\[$(tput sgr0)\]\[\033[38;5;15m\]@\h:\[$(tput sgr0)\]\[\033[38;5;6m\][\w]:\[$(tput sgr0)\]\[\033[38;5;15m\] \[$(tput sgr0)\]"

PS1="\[\033[35m\]\t\[\033[m\]-\[\033[36m\]\u\[\033[m\]@\[\033[32m\]\h:\[\033[33;1m\]\w\[\033[m\]\$ "

PS1="[\[\033[32m\]\w]\[\033[0m\]\n\[\033[1;36m\]\u\[\033[1;33m\]-> \[\033[0m\]"
```
