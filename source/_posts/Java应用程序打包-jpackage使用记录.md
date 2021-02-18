---
title: Java应用程序打包-jpackage使用记录
categories: Java
tags: jpackage
abbrlink: 421e5ad2
date: 2020-05-06 08:16:53
---

jpackage是java 14里面自带的打包工具，jpackage解决了java开发过程中得一个难题：分发自己的程序，需要客户电脑中已安装jre环境。有了jpackage，我们可以直接将java程序打包成安装包，具体来说：

- Windows：exe，msi
- Mac：dmg，pkg
- Linux：deb，rpm

jpackage目前并不成熟，但是也算是可以使用。另外，虽然jpackage可以打包各个系统的安装包，但是在一个系统上只能打对应系统的安装包。比如在windows上，就只能打成exe或msi。

<!--more-->

额外建议：jpackage打包，涉及到java 9中的模块概念以及如何使用jlink自定义jre的知识，请自行学习。推荐阅读：

- [模块-廖雪峰](https://www.liaoxuefeng.com/wiki/1252599548343744/1281795926523938)
- [Java9新特性系列（模块化系统: Jigsaw->Modularity）](https://juejin.im/post/5a7c643a6fb9a0633b20febe)
- [我的Java（定制你的Java/JavaFX Runtime）](https://zhuanlan.zhihu.com/p/45824326)

本文主要介绍如何使用jpackage打包，以及在打包过程中遇到的各种坑。

## 1. 环境准备

要使用jpackage打包，请确保：

1. 安装好jdk 14，配置好JAVA_HOME和PATH
2. windows平台，请安装[wix3](https://github.com/wixtoolset/wix3)

安装好，打开cmd，执行jpackage -h：

![image-20200506083414289](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506083414289.png)

如果有这样的显示，那说明安装成功。注意首行的WARNING，这个不用管，因为jpackage目前还属于一个孵化阶段，所有有这样的提示。

另外，下文中的所有程序都可放在了github，地址：

https://github.com/raven-misc/jpackage-demo

## 2. 非模块程序打包

先以**非模块程序**为例，新建一个java项目，得到如下目录：

![image-20200506085408605](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506085408605.png)

代码：

```java
package com.raven;

import java.awt.*;

/**
 * @author Raven
 * @version 1.0
 * @date 2020/5/6 8:53
 */
public class App {
    public static void main(String[] args) {
        Toolkit.getDefaultToolkit().beep();
    }
}

```

功能很简单，让系统发出beep声。

然后将这个项目打包成jar包，可以通过命令，也可以直接通过idea，自行选择，我这里采用idea：

![image-20200506085602846](https://pic.downk.cc/item/5eb22654c2a9a83be5ae6281.png)

测试一下通过java -jar能否运行：

![image-20200506090006946](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506090006946.png)

好的，现在我们将这个jar包打成安装包：

在根目录下建立一个lib，并把Non-modular-packging-demo.jar复制进lib目录下：

![image-20200506090210470](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506090210470.png)



### 2.1 安装包

在根目录下，打开你的terminal，执行（命令是执行不成功的，会报311错误，但是先执行看看）：

```shell
jpackage --name Non-modular-installer --input lib --main-class com.raven.App --main-jar Non-m
odular-packaging-demo.jar
```

- --name 打包后的安装包名
- --input  要打包的文件目录
- --main-class 这是非必须选项，如果你的jar包META-INF/MANIFEST.MF中已经指定了Main-Class, 则无需此命令。
- --main-jar 主程序所在jar包
- 其余常用：
  - --temp 临时文件所在目录，默认系统temp
  - --dest 打包到哪个目录下,默认当前目录
  - --type 打包成什么类型，exe？msi？deb等等，windows默认是exe

不过在我使用这条命令时(2020/5/5)，**报了311错误**，源头来自wix。解决方案，在命令后指定vendor，如：

```shell
jpackage --name Non-modular-installer --input lib --main-class com.raven.App --main-jar Non-m
odular-packaging-demo.jar --vendor raven
```

![image-20200506091042775](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506091042775.png)

ok，已经打包成功，试试安装吧（提示，又会有想不到的效果）。

相信你通过双击安装，已经发现了问题，那就是安装后没有任何效果，没有windows-menu，没有windows快捷键，程序也没有运行启动。但是**控制面板已经找到安装程序了**。我一度以为是自己电脑环境的问题，最后发现在windows平台上还需加上几条命令才行：

```
jpackage --name Non-modular-installer --input lib --main-class com.raven.App --main-jar Non-modular-packaging-demo.jar --vendor raven --win-dir-chooser --win-shortcut --win-menu --win-menu-group "Non-modular-packaging" 
```

解释：

- --win-dir-chooser, 安装时添加“选择安装路路径”

- --win-shortcut,安装后自动在桌面添加快捷键

- --win-menu-group,启动该应用程序所在的菜单组(实测无效，但是必须有这条命令，没有--win-menu会报311错误)

  **update 2021-2-18： –win-menu-group应该放在–win-menu之后，否则无效。**

- --win-menu，添加到系统菜单中

![image-20200506091723697](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506091723697.png)

![image-20200506091759019](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506091759019.png)

![image-20200506091808728](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506091808728.png)

现在双击启动程序试试，应该就有beep了。再看看安装目录：

![image-20200506092615980](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506092615980.png)

### 2.2 便携版（无需安装）

jpackage提供一个选项，可以用来生成镜像（image），而这个镜像就可以充当便携版执行命令和上面基本一致，添加--type app-image命令,删除所有--win-xxx即可：

```shell
jpackage --name Non-modular-installer --type app-image --input lib --main-class com.raven.App --main-jar Non-modular-packaging-demo.jar --vendor raven
```

![image-20200506092833058](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506092833058.png)

内部目录和安装包安装后的目录完全一致，所以直接压缩这个文件夹就有便携版了。

### 2.3 便携版-->安装包

我们打包成镜像后，还可以将镜像转化为安装包（通常发生在我们想对镜像做一些定制化内容时），执行命令:

```shell
jpackage --name Non-modular-installer --type msi --app-image Non-modular-installer --vendor raven
```

这里打包成msi:

![image-20200506093330774](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506093330774.png)

## 3. 模块程序打包

通过maven新建JavaFX项目，得到如下目录：

![image-20200506085023406](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506085023406.png)

运行一下，看下效果：

![image-20200506093953328](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506093953328.png)



模块化程序打包就略微复杂一些了，首先明确一点，idea中，有maven在，所有的编译输出不在out目录下，而在target目录下：

![image-20200506094133814](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506094133814.png)

除此之外，还需要明确我们当前的模块名：

![image-20200506094239319](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506094239319.png)

当前模块名为app。

### 2.1 字节码打包

首先确定依赖的modules，在module-info.java文件中，

![image-20200506095324701](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506095324701.png)

生成如下的依赖图：

![image-20200506095350078](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506095350078.png)

不用关注java开头和jdk开头的模块，因为它们是系统自带的。也就是所当前app模块直接或间接依赖了：

- javafx.controls
- javafx.graphics
- javafx.base

三个模块，现在我们要找到这个三个模块jar包路径，如何找？

![image-20200506095511115](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506095511115.png)

在这上面右键-->Show in Exploer即可。

![image-20200506095601116](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506095601116.png)

可以看到导航到了我们的maven仓库中，分别记下这三个jar包的路径，如下：

```
C:\Users\Raven\.m2\repository\org\openjfx\javafx-controls\14\javafx-controls-14 win.jar;C:\Users\Raven\.m2\repository\org\openjfx\javafx-base\14\javafx-base-14-win.jar;C:\Users\Raven\.m2\repository\org\openjfx\javafx-graphics\14\javafx-graphics-14-win.jar
```

**现在都是自己手动找的，之后会介绍插件，自动寻找。**

ok，准备工作就绪，开始打包。在target目录下，打开terminal：

执行命令：

```shell
jpackage --name "Modular-installer" --module-path classes;C:\Users\Raven\.m2\repository\org\openjfx\javafx-controls\14\javafx-controls-14-win.jar;C:\Users\Raven\.m2\repository\org\openjfx\javafx-base\14\javafx-base-14-win.jar;C:\Users\Raven\.m2\repository\org\openjfx\javafx-graphics\14\javafx-graphics-14-win.jar -m app/com.raven.App --vendor raven
```

解释：

- --name 应用程序名
- --module-path: 也就是前面让你找的几个依赖模块的jar包路径（以;间隔）+classes（自己的模块）
- -m: 指定主模块及主程序。 app/com.raven.App, "/"前是模块名，"/"后是主程序全路径。

![image-20200506100019571](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506100019571.png)

现在还会遇到的问题，在”非模块程序打包“下都已经作了说明，现在补全所有命令：

```shell
jpackage --name "Modular-installer" --module-path classes;C:\Users\Raven\.m2\repository\org\openjfx\javafx-controls\14\javafx-controls-14-win.jar;C:\Users\Raven\.m2\repository\org\openjfx\javafx-base\14\javafx-base-14-win.jar;C:\Users\Raven\.m2\repository\org\openjfx\javafx-graphics\14\javafx-graphics-14-win.jar -m app/com.raven.App --vendor raven --win-dir-chooser --win-shortcut --win-menu --win-menu-group "Modular-packaging" 
```

### 2.2 通过jar包打包

注意到我们上面module-path中有一项是classes，这就是我们自己的模块，我们也可以先将整个模块打包成jar包，然后替换classes。

有maven，打包就是再简单不过的事了，

![image-20200506100337247](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506100337247.png)

![image-20200506100405088](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506100405088.png)

重新执行命令：

```shell
jpackage --name "Modular-installer" --module-path Modular-packaging-demo-1.0-SNAPSHOT.jar;C:\Users\Raven\.m2\repository\org\openjfx\javafx-controls\14\javafx-controls-14-win.jar;C:\Users\Raven\.m2\repository\org\openjfx\javafx-base\14\javafx-base-14-win.jar;C:\Users\Raven\.m2\repository\org\openjfx\javafx-graphics\14\javafx-graphics-14-win.jar -m app/com.raven.App --vendor raven --win-dir-chooser --win-shortcut --win-menu --win-menu-group "Modular-packaging" 
```

### 2.3 打包成镜像（便携版）

同”非模块程序打包“一样，添加--type,删除--win-xx

```shell
jpackage --name "Modular-installer" --module-path Modular-packaging-demo-1.0-SNAPSHOT.jar;C:\Users\Raven\.m2\repository\org\openjfx\javafx-controls\14\javafx-controls-14-win.jar;C:\Users\Raven\.m2\repository\org\openjfx\javafx-base\14\javafx-base-14-win.jar;C:\Users\Raven\.m2\repository\org\openjfx\javafx-graphics\14\javafx-graphics-14-win.jar -m app/com.raven.App --vendor raven --type app-image
```

### 2.4 定制jre，自定义jlink，插件化（推荐打包方式）

在jpackage打包模块化程序时，其实底层调用了jlink，比如我们的--module-path参数，其实时会传递给jlink的。当我们需要对jlink做一些自定义参数时，可以先调用jlink生成自定义的jre，然后通过这个jre来打包。

前面的打包过程，有一个非常头疼的问题，那就是需要我们自己去寻找依赖，然后找到这些一些的路径。（当然其实下载jmods也是可以的，但是还是麻烦的），还好我们有插件辅助。

打开pom文件，会看到这个插件：

```java
<plugin>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-maven-plugin</artifactId>
    <version>0.0.3</version>
    <configuration>
        <mainClass>com.raven.App</mainClass>
    </configuration>
</plugin>
```

注意版本一定要是0.0.3+。然后点击：

![image-20200506102259548](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506102259548.png)

会得到：

![image-20200506102410359](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506102410359.png)

在image/bin中打开terminal，执行`java --list-modules`命令：

```shell
app
java.base@14.0.1
java.datatransfer@14.0.1
java.desktop@14.0.1
java.prefs@14.0.1
java.xml@14.0.1
javafx.base
javafx.controls
javafx.graphics
jdk.unsupported@14.0.1
```

上面就是我们这个jre所包含的所有模块。可以和我们之前做依赖分析的图做对比，是不是完全相同？
接着我们通过jre来进行打包。

回到target目录：

```shell
jpackage -n Modular-packaging-demo --type msi --runtime-image image --vendor raven
```

老规矩，如果需要menu，shortcut，自己添加。

![image-20200506102742159](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506102742159.png)

细心的同学会发现一个问题，那就是我们手动打包的exe文件对比通过jre打包的文件，体积会小很多。这怎么办呢？

回想一下，为什么我们要通过插件来打包，**因为我们不想手动找各个模块的依赖路径**。换句话说，只要我能够方便的获取模块的依赖路径就好了。

好，那怎么办呢？

其实通过maven+debug就好了。再次在target目录下打开terminal：

```shell
mvn clean -X javafx:jlink
```

-X打开debug。最后会得到：

![image-20200506103403094](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200506103403094.png)

这就是我们的依赖路径了，提取出来，再手动执行jpackage就ok了。

## 4. 参考

- [JEP 343: Packaging Tool (Incubator)](https://openjdk.java.net/jeps/343)
- [JavaFX环境搭建&部署打包](https://www.bilibili.com/video/BV1tz411q7Q1?p=4)
- [Brief Example Using the Early Access jpackage Utility](https://dzone.com/articles/a-brief-example-using-the-early-access-jpackage-ut)

总算是写完了，从学习modular到遇到各种坑，花了差不多整整一天，就为了打包javafx。总体来说，jpackage还不算成熟，仍有些bug，比如为什么一定要添加vendor，又如为什么--win-menu一定要和--win-menu-group配套。但总体来说能使用。

