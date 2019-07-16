---
title: 从源码JAR包中生成javadocHTML页面
date: 2019-07-16 14:55:00
tags:
- javadoc
- jar
categories:
- 工具
---

![](/images/javadocEx1.jpg)

有时为了搞清楚某个jar包该怎么使用，里面有哪些方法，我们会**点**进去查看类和方法上面的文档注释，这通常是通过IDE导航完成的。的确可以查看到这些信息，不过看上去不直观，特别是里面有一些代码示例时，当然你可以使用IDE自带的功能渲染某段注释，不过这些东西总是零散的出现在你的视野，我们是不是可以本地生成HTML页面来更方便的查看呢？答案是肯定的。
<!-- more -->
# 步骤

以生成junit的javadoc为例。

## 获取源码

你可以在本地maven仓库或gradle缓存目录中找到对应jar包的源码，它们通常位于 `C:\Users\Administrator\.m2\repository` 和 `C:\Users\Administrator\.gradle\caches` 目录。

1. 在本地仓库中找到 `junit-4.12-sources.jar` 文件并复制到其它目录。
1. 注意要判断是否带有 `sources` 单词，否则是编译后的jar包，里面已经没有注释内容了，也就不可能再生成文档。

## 解决依赖

由于junit依赖hamcrest，所以我们还需要把 `hamcrest-core-1.3-sources.jar` 一并复制出来，如果不这样做在生成时会报错。在为其它第三方jar包生成javadoc时，可以根据错误提示找到它所依赖的jar包，把它们一并复制出来即可。

**注：**依赖包也必须是源码jar包。

## 解压源码

1. 在当前目录（源码所在目录，例如： `E:\tmp`）中创建新目录 `source`。
1. 打开CMD命令，并进入 `source` 目录中执行jar包解压操作。
    ```bat
    E:\tmp\source> jar vxf ../junit-4.12-sources.jar
    E:\tmp\source> jar vxf ../hamcrest-core-1.3-sources.jar
    ```
1. 可以在 `source` 目录中看到源码文件

## 生成javadoc

回到源码所在目录并创建 `javadoc` 新目录，使用下面的命令生成javadoc。

```bat
E:\tmp> javadoc -html5 -d E:\tmp\javadoc -sourcepath E:\tmp\source -subpackages org.junit

正在构建所有程序包和类的索引...
正在生成E:\tmp\javadoc\overview-tree.html...
正在生成E:\tmp\javadoc\deprecated-list.html...
正在构建所有类的索引...
正在生成E:\tmp\javadoc\index.html...
正在生成E:\tmp\javadoc\index-all.html...
正在构建所有类的索引...
正在生成E:\tmp\javadoc\allclasses-index.html...
正在生成E:\tmp\javadoc\allpackages-index.html...
正在生成E:\tmp\javadoc\overview-summary.html...
正在生成E:\tmp\javadoc\help-doc.html...
11 个错误
100 个警告
```

上面的错误是来自于javadoc写的不规范导致的，可以不用理会。

## 最后

1. 在 `javadoc` 目录中可以看到生成的HTML文件，单击 `index.html` 文件可以看到javadoc首页。

---
参考文献：
1. https://www.reddit.com/r/javahelp/comments/64so9p/how_do_i_generate_javadoc_html_from_a_source_jar/
1. https://docs.oracle.com/javase/8/docs/technotes/tools/windows/javadoc.html#CHDJBGFC