---
title: 两种删除linux内核的方式
categories: Linux
toc: true
abbrlink: a7295ff5
date: 2019-05-03 13:58:26
tags:
---


## 1.前言

在`linux`的世界里，升级、更换内核是很经常的事情，如果你在安装系统的时候给boot分区分小了，偶尔就会出现`boot`分区的占用率爆掉的情况。这里给出两种删除内核的方式
<!-- more -->
## 2. 删除内核

### 2.1 方式一

**step1:**
如果你使用`debian`系发行版本的`linux`的话，可以使用`dpkg`命令(`fedora`系好像是`rpm`)来查看本机装有了哪些内核，具体命令为 :

    dpkg --get-selections|grep linux

![在这里插入图片描述](https://pic3.superbed.cn/item/5cfbb5c4451253d178d9d01c.png)
当然了，我才清理了内核，所以现在只有一个可用的内核版本:`linux-4.15.0-42`。如果你的主机上安装过多个内核的话，这里应该会显示多个才对。

**step2:**
接着查看当前自己使用的是哪个版本:

`uname -r`

![在这里插入图片描述](https://pic1.superbed.cn/item/5cfbb5c6451253d178d9d051.png)

**step3:**
使用 `sudo apt-get purge linux-xxx` 把所有不要的内核版本有关文件全部删了，如这里可以把所有包含4.15.0-42的文件删掉。

**删除除当前你使用的版本的内核以外的其他内核。**

之后在使用`dpkg --get-selections|grep linux`命令查看一下是不是已经删除了呢。



### 2.2 方式二

方式一适合于使用官方包管理器来升级内核版本，当我们通过源码编译来安装内核时，上述方法就不适用了，因为你使用`dpkg --get-selections|grep linux`命令来查看安装了那些内核时，自编译的内核是不会显示出来的，那么该如何删除这样的内核呢？

**step1:**
来到/boot目录:把下面红框内的文件都给删了，注意自己不需要哪些版本哈。XD

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbb5cd451253d178d9d0d0.png)

可以使用通配符来删除：

`sudo rm *4.15.0-42*`

**step2:**
来到`/lib/modules/`目录下，把不要的版本文件删除

![在这里插入图片描述](https://pic2.superbed.cn/item/5cfbb5cf451253d178d9d10c.png)
（可选），查看一下`/usr/src`有没有源码文件，必要的话把源码也删除了

**step3:** 
更新一下启动项: `sudo update-grub`

完。


