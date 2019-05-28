---
title: 在Gradle或Maven中切换中央仓库地址为国内镜像源以加速jar包下载
date: 2017-10-22 22:59:19
tags:
- gradle
- maven
- 阿里云
- 国内镜像源
categories:
- 项目构建
---

# 背景

众所周知maven中央仓库位于国外服务器，国内朋友在下载时比较缓慢，常常在构建一个新项目时要等待比较长的时间。如果能够直接从国内服务器下载，那将大幅缩短项目构建时间，下面介绍一下切换源的方法。

<!-- more -->

# 切换国内镜像

## maven

在Maven安装目录找到 `settings.xml` 并配置如下，在这里我们使用阿里云的源，它的速度还是相当快的。

```xml
<mirror>
      <id>alimaven</id>
      <name>aliyun maven</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
</mirror>
```

## gradle

在 `$USER_HOME/.gradle/` 下面创建新文件 `init.gradle` 并将下面的内容输入保存。

```groovy
allprojects{
    repositories {
        def REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/groups/public/'
        all { ArtifactRepository repo -&gt;
            if(repo instanceof MavenArtifactRepository){
                def url = repo.url.toString()
                if (url.startsWith('https://repo1.maven.org/maven2') || url.startsWith('https://jcenter.bintray.com/')) {
                    project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                    remove repo
                }
            }
        }
        maven {
            url REPOSITORY_URL
        }
    }
}
```
