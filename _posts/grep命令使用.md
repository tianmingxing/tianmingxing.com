---
title: grep命令使用
date: 2017-11-12 10:58:11
tags:
- grep
- linux grep
categories:
- 工具
---

> grep是一个最初用于Unix操作系统的命令行工具。在给出文件列表或标准输入后，grep会匹配一个或多个正则表达式的文本进行搜索，并只输出匹配的行或文本。它是一个非常实用的强大文本搜索命令，下面介绍它常见的几种应用场景。

# 使用方法

为了方便演示grep的各种常见使用方法，我们在github上随意找一个项目并下载，接着进入根目录开始测试。

```shell
cd /tmp
wget -c https://github.com/WordPress/WordPress/archive/master.zip
unzip master.zip
mv WordPress-master WordPress
cd WordPress
```
<!-- more -->
1. 在某文件中查找字符串 `grep [要搜索的字符串] [具体的文件路径]`
  * 这是最常用最简单的一种应用方式，需要注意的是第二个参数只能是具体的文件路径，传个目录给它是会报错的 `grep: .: Is a directory`。
  * 如果要查找指定目录下的所有文件，可以用这种写法 `grep 'php' wp-admin/*`，避免手动列出所有文件。
  * `grep 'php' index.php` 在 `index.php` 文件中查找 `php` 字符串。
  * 可以同时在多份文件中查找 `grep 'php' index.php readme.html`。
1. 在某目录下递归查找字符串 `grep -r [要搜索的字符串] [具体的文件目录]`
  * 加上 `-r` 参数后可以指定任意文件目录，而不是像上面一样只能是具体的文件。它会在你指定的目录递归查询所有子目录下的所有文件，并在里面逐一查找你指定的字符串。
  * 可以同时指定多个文件目录 `grep -r 'php' wp-admin wp-content`。
1. 显示找到的字符串在文件中的多少行 `grep -n 'php' [具体的文件路径]`
  * `-n` 参数的作用是显示行号
  * 如果你想在某个文件目录中递归查找，只要将选项叠加在一起使用即可 `grep -rn 'php' wp-content`。
1. 统计匹配的字符串在找到的文件中的数量 `grep -rc 'php' wp-admin`
  * `-c` 参数的作用是统计字符串在某份文件中找到的次数
1. 查找指定字符串包含在指定目录的哪些文件中 `grep -rl 'php' wp-content`
  * `-l` 参数的作用是在打印时只显示文件名称
1. 用正则表达式描述被查找的字符串 `grep -rn '^<?php$' wp-content`
  * 直接用正则描述字符串即可，上面是查找以 `<?php` 开头和结尾的字符串。
  * 文件目录同样支持正则匹配
  * `-i` 不区分大小（默认区分大小写）
  * `-L` 输出没有匹配的文件名，它和 `-l` 的作用是相对的。
  * `-v` 反向匹配以选择不匹配的行。
1. 在指定目录下的匹配文件中查找 `grep 'php' --include=index.php  wp-admin/*`
  * `--exclude` 参数表示排除哪些文件不参与查找
