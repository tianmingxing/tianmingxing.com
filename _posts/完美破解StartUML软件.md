---
title: 完美破解StartUML软件
date: 2019-12-26 11:10:00
tags:
- StartUML
categories:
- 工具
---

> 本文介绍的方法仅供学习使用，如果要正式使用请购买许可证。

[StartUML](http://staruml.io/)是一款非常棒的UML图绘制工具，官方提供了试用版本，但它在使用时会有所限制，在每次保存时有可能弹出让你购买许可证的提示，以及在将设计图导出成图片时会自动添加上水印。

那如何屏蔽掉这些限制呢？请看下面的方法：
<!-- more -->

## 安装Nodejs

在官方https://nodejs.org/zh-cn/上下载并安装

## 安装asar

待nodejs安装完成后，打开命令行，输入 `npm install -g asar` 全局安装asar。

## 修改app.asar中的源码

1. 进入StartUML安装目录 `C:\Program Files\StarUML\resources` 找到 `app.asar` 压缩文件。
1. 解析找到的文件并输出到某个目录 `asar e app.asar C:\Users\mxtia\Documents\tmp\app`。
1. 进入解析出来的目录 `C:\Users\mxtia\Documents\tmp\app\src\engine`，打开里面的 `license-manager.js` 文件。
1. 将里面的 `checkLicenseValidity` 方法修改成下面这样
```javascript
checkLicenseValidity () {
    this.validate().then(() => {
      setStatus(this, true)
    }, () => {
      //下面两行注释掉
      //setStatus(this, false)
      //UnregisteredDialog.showDialog()
	  
	    //修改后的代码，这行是新添加的
      setStatus(this, true)
    })
}
```
1. 修改成上面这样后保存当前文件。

## 将修改后的源码重新压缩

在输出目录 `C:\Users\mxtia\Documents\tmp\app` 执行命令 `asar pack app app.asar` 将修改后的源码压缩打包。命令执行完后可以看到目录下面新生成了**app.asar**文件。

## 覆盖官方的asar文件

将上一步生成的新app.asar文件覆盖StartUML安装目录 `C:\Program Files\StarUML\resources` 中的app.asar文件。

## 最后

重新启动StartUML则发现已经完美破解。
