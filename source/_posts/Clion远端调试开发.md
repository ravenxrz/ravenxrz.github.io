---
title: Clion远端调试开发环境部署
date: 2020-12-23 11:24:56
categories: 杂文
tags:
---

从来没用过JetBrains家的Remote Development, 今天体验了一波，效果非常好。这里说一下如何配置，以及一些问题的解决方式。

<!--more-->

## 1. 环境说明

我是开了虚拟机来做Linux服务器，实际上如果你有云服务器或者有多余的主机，甚至是树莓派都可以用这个方法。

1. Ubuntu  服务，ip 192.168.18.142.
2. Windows 本地机

## 2. 基本配置

打开设置，如下图添加Remote　Host:

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223112912644.png" alt="image-20201223112912644" style="zoom:50%;" />![image-20201223112935496](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223112935496.png)<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223112912644.png" alt="image-20201223112912644" style="zoom:50%;" />![image-20201223112935496](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223112935496.png)

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223112912644.png" alt="image-20201223112912644" style="zoom:50%;" />![image-20201223112935496](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223112935496.png)<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223112912644.png" alt="image-20201223112912644" style="zoom:50%;" />![image-20201223112935496](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223112935496.png)

如果发现无法连接，多半是Linux没有安装ssh-server:

```shell
sudo apt-get install openssh-server
```

上面配置好后，再按下图配置：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223113115796.png" alt="image-20201223113115796" style="zoom:50%;" />

接着可以在项目主界面右上角选择采用本地编译还是远端编译：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223113152657.png" alt="image-20201223113152657" style="zoom:50%;" />

如果一切顺利，现在就可以运行起来了。

下面在说一下其他配置与问题。

## 3. 其他配置

首先第一个问题，你的项目文件被推送到远端的哪个地方了？

打开：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223113411916.png" alt="image-20201223113411916" style="zoom:50%;" />

可以看到默认的主目录是根目录（当然当前项目肯定不是推送到这儿的）：

<img src="D:\坚果云同步\图库\image-20201223113448453.png" alt="image-20201223113448453" style="zoom:50%;" />

*点击Autodect可以自动变为当前用户的home目录*

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223113638033.png" alt="image-20201223113638033" style="zoom:50%;" />

点击旁边的Mappings：

<img src="D:\坚果云同步\图库\image-20201223113604112.png" alt="image-20201223113604112" style="zoom: 50%;" />

主要关注远端项目位置， 这里的 远端项目位置就是和前面的“Root path”合并的位置。如我这里最终的效果是：

将 本地 `E:\RemoteTest` 推送到 远端 `/home/raven/Projects/RemoteTest/` 下。

还有一个Excluded Paths：用于排除哪些目录是不需要推送到远端的。

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223113809706.png" alt="image-20201223113809706" style="zoom:50%;" />



还有个问题是头文件解析相关, 比如你做linux开发，很多linux特定的头文件在windows下是没有的，这时候怎么办呢？Clion在 **第一次推送文件到远端时，会自动拉取头文件依赖（个人估计是包含了/usr/include和/usr/local/include)**, 后续将不再更新，如果你在开发过程中，更新了库依赖，比如安装了新的库等，需要重新同步头文件依赖。点击下图进行更新：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201223114321453.png" alt="image-20201223114321453" style="zoom:50%;" />



## 4. 问题说明&解决

### 1. 问题1：修改的文件如何实时同步？

> 下面的解决方案需要你学过vim

说完了这些，再说一下优化，如果你仅仅是像上面这样配置，你会发现在本地修改代码后，直接运行代码是没有修改的，可能要到第二次运行才是真的推送到远端的。如何解决这个问题？

根据官网所描述的，远端开发是采用 rsync 命令实现的，所以这里的问题是，你修改的代码没有落盘到文件，也就没有推送到远端。那如何落盘到文件？

首先，开启自动上传：

<img src="D:\坚果云同步\图库\image-20201223114643521.png" alt="image-20201223114643521" style="zoom:50%;" />

接着 **需要你学过vim，打开ideavim插件，添加**

```
" run
nnoremap <Leader>r :action SaveAll<CR>:action RunClass<CR>
```

我的 Leader 键是分号。上面的意思是，按分号+r，自动执行一次保存全部，然后执行运行。 这样当自动保存执行后，clion自动推送修改的代码到远端，然后执行自动运行。就能全自动了。

### 2. 问题2：时间校准

windows和ubuntu的时间尽量保证一致，否则可能出现`warning: modification time in the future`类似的警告。 如何同步请自行百度了，这不影响使用。