---
title: 在Gradle或Maven中切换中央仓库地址为国内镜像源以加速jar包下载
date: 2017-10-22 22:59:19
tags:
- gradle
- maven
- 阿里云Maven仓库
- 国内Maven镜像源
categories:
- 项目构建
---

# 背景

众所周知maven中央仓库位于国外服务器，国内朋友在下载时比较缓慢，常常在构建一个新项目时要等待比较长的时间。如果能够直接从国内服务器下载，那将大幅缩短项目构建时间，下面介绍一下切换源的方法。

# 阿里云代理仓库

`maven.aliyun.com` 代理了很多公共的maven仓库。使用maven.aliyun.com中的仓库地址作为下载源，速度更快更稳定。

## 代理的仓库列表

| 仓库名称         | 代理源地址                                 | 使用地址                                                     |
| :--------------- | :----------------------------------------- | :----------------------------------------------------------- |
| central          | <https://repo1.maven.org/maven2/>          | <https://maven.aliyun.com/repository/central> 或 <https://maven.aliyun.com/nexus/content/repositories/central> |
| jcenter          | <http://jcenter.bintray.com/>              | <https://maven.aliyun.com/repository/jcenter> 或 <https://maven.aliyun.com/nexus/content/repositories/jcenter> |
| public           | central仓和jcenter仓的聚合仓               | <https://maven.aliyun.com/repository/public> 或<https://maven.aliyun.com/nexus/content/groups/public> |
| google           | <https://maven.google.com/>                | <https://maven.aliyun.com/repository/google> 或 <https://maven.aliyun.com/nexus/content/repositories/google> |
| gradle-plugin    | <https://plugins.gradle.org/m2/>           | <https://maven.aliyun.com/repository/gradle-plugin> 或 <https://maven.aliyun.com/nexus/content/repositories/gradle-plugin> |
| spring           | <http://repo.spring.io/libs-milestone/>    | <https://maven.aliyun.com/repository/spring> 或 <https://maven.aliyun.com/nexus/content/repositories/spring> |
| spring-plugin    | <http://repo.spring.io/plugins-release/>   | <https://maven.aliyun.com/repository/spring-plugin> 或 <https://maven.aliyun.com/nexus/content/repositories/spring-plugin> |
| grails-core      | <https://repo.grails.org/grails/core>      | <https://maven.aliyun.com/repository/grails-core> 或 <https://maven.aliyun.com/nexus/content/repositories/grails-core> |
| apache snapshots | <https://repository.apache.org/snapshots/> | <https://maven.aliyun.com/repository/apache-snapshots> 或 <https://maven.aliyun.com/nexus/content/repositories/apache-snapshots> |

<!-- more -->
# 切换阿里云Maven代理仓库

## maven

在这里我们使用阿里云的源，它的速度还是相当快的。在Maven安装目录找到 `settings.xml` （windows机器一般在maven安装目录的conf/settings.xml），在 `<mirrors></mirrors>` 标签中添加mirror子节点：

```xml
<mirror>
    <id>aliyunmaven</id>
    <mirrorOf>*</mirrorOf>
    <name>阿里云公共仓库</name>
    <url>https://maven.aliyun.com/repository/public</url>
</mirror>
```

如果想使用其它代理仓库，可在 `<repositories></repositories>` 节点中加入对应的仓库使用地址。以使用spring代理仓为例：

```xml
<repository>
    <id>spring</id>
    <url>https://maven.aliyun.com/repository/spring</url>
    <releases>
        <enabled>true</enabled>
    </releases>
    <snapshots>
        <enabled>true</enabled>
    </snapshots>
</repository>
```

## gradle

在项目 `build.gradle` 文件中加入以下代码：

```groovy
allprojects {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/public/' }
        mavenLocal()
        mavenCentral()
    }
}
```
如果想使用 `maven.aliyun.com` 提供的其它代理仓，以使用spring仓为例:

```groovy
allProjects {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/public/' }
        maven { url 'https://maven.aliyun.com/repository/spring/'}
        mavenLocal()
        mavenCentral()
    }
}
```