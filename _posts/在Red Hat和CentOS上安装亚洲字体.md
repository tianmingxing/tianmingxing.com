---
title: 在Red Hat和CentOS上安装亚洲字体
date: 2019-4-8 23:08:16
tags:
- 亚洲字体
- CentOS
- Red Hat
categories:
- Linux
---

平台不同版本安装方式有些差异，下面会分别介绍：

* Version 6
* Version 7

### Version 6

在RHEL 6 / CentOS 6版本中语言字体组软件包不再可用。 相反，这些版本具有groupinstall列表，它将多个包捆绑在一起。 要查看可用的所有语言包可以输入以下命令：

| Example            |
| :----------------- |
| `# yum grouplist ` |

这将显示所有组包，其最后一部分是**可用语言组**。 请注意，某些包可能已经安装，在这种情况下，它们将显示在列表顶部的**已安装的语言包**下。 在这种情况下 `chinese-fonts` 现在是 `Chinese Support`。

<!-- more -->

要安装此程序包，请运行以下命令：

| Example                                 |
| :-------------------------------------- |
| `# yum groupinstall "Chinese Support" ` |

要安装帮助文件中列出的所有语言，请运行以下命令：

| Example                                 |
| :-------------------------------------- |
| `# yum groupinstall "Chinese Support"`  |
| `# yum groupinstall "Japanese Support"` |
| `# yum groupinstall "Korean Support"`   |
| `# yum groupinstall "Kannada Support"`  |
| `# yum groupinstall "Hindi Support"`    |

### Version 7 

在RHEL 7 / CentOS7版本中语言字体组软件包不再可用。 相反，这些版本具有groupinstall列表，它将多个包捆绑在一起。 有一个字体包其中包括对这些语言的支持：

| Example                    |
| :------------------------- |
| `# yum groupinstall Fonts` |