---
title: JavaFX+IDEA+Maven环境搭建
categories:
  - JavaFX
tags: 环境搭建
abbrlink: dba39b2f
date: 2020-05-06 11:09:00
---

本文讲解如何在Idea上搭建JavaFX开发环境，对应视频：https://www.bilibili.com/video/BV1tz411q7Q1/

如果想了解如何打包JavaFX程序，参见：https://www.ravenxrz.ink/archives/421e5ad2.html

<!--more-->

## Start

要搭建环境，最直接的肯定是参看官方文档了，https://openjfx.io/openjfx-docs/

![image-20200506111205459](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506111205459.png)

这里再复述一遍：

### step1: 添加archetype（只用第一次添加，后面创建项目就不用了）

![image-20200506111939997](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506111939997.png)

![image-20200506112020694](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506112020694.png)

- the groupId (org.openjfx),

- the artifactId (javafx-maven-archetypes)
- the version (0.0.1)

### step2 创建项目

![image-20200506112115982](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506112115982.png)

![image-20200506112145436](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506112145436.png)

![image-20200506112203726](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506112203726.png)

value中填写：

- javafx-archetype-fxml， javafx项目包含fxml
- javafx-archetype-simple， javafx项目不包含fxml

二者选一，我以javafx-archetype-simple为例

然后点击右上角的+：

![image-20200506112319277](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506112319277.png)

指定javafx的版本，目前最新为14。

Finish即可。第一次创建可能有点慢，因为maven需要下载一些依赖包。

打开pom文件，修改javafx插件版本为0.03：

```java
<plugin>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-maven-plugin</artifactId>
    <version>0.0.3</version>
    <configuration>
        <mainClass>org.example.App</mainClass>
    </configuration>
</plugin>
```

看一下创建好的目录：

![image-20200506112428268](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506112428268.png)

打开App，运行。

![image-20200506112458825](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506112458825.png)