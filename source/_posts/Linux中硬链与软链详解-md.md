---
title: Linux中硬链与软链详解
date: 2019-08-02 13:58:08
categories: Linux
toc: true
tags:
---

本文整理自《鸟哥的Linux私房菜-基础学习篇》

## 1. 软链

创建软链接的方式如下：

<!-- more -->
```shell
ln -s target link
```

效果如下：

![](https://ae01.alicdn.com/kf/H054a6971f4034628a0f83302edc2a6622.jpg)

软链和Windows的“快捷方式”完全等同。通过软链接我们可以间接访问到源文件，软链接是一个单独的文件，是和源文件完全不同的文件。如何证明这点呢？了解Linux文件系统的朋友肯定知道，在同一个文件系统中，一个文件对应其唯一的inode编号。于是我们只用查看两个文件的inode编号是否相同就知道两个文件是否为同一个文件啦。具体方式如下：

```
ll -i
```

![](https://ae01.alicdn.com/kf/H7970b24cd4184755afdaafc9a2d901fbW.jpg)

第一列即为inode编号，可以到file1和file1_sl两者的inode编号不同，所以两个是不同的文件。

软链文件中存储的内容就是源文件的真实路径，软链接的大小对应源文件的路径长度，如这里软链接为5个字节，对应file1这5个字符，其工作原理如下图：

![](https://ae01.alicdn.com/kf/H11bbe0318b2d479eb0ea67cf2bd4d7d5c.jpg)

## 2. 硬链

我们知道，要访问一个文件，我们首先在目录中根据文件名找到文件的inode，再根据inode找到实际存储文件内容的block，如下图：
![](https://ae01.alicdn.com/kf/Hbf84015e9ed24e1783e33f14c2b0643bb.jpg)

可以通过如下命令创建一个硬链接：

```shell
ln target link
```

效果如下图：

![](https://ae01.alicdn.com/kf/H560e005fb5c641258b7e9644b31bc3e7o.jpg)

我们要关注的有两点：

1. 通过`ll -i`命令，我们发现file1和file1_hl的inode是完全相同的，这说明**两者是完全一样的文件**。我们对其中一个文件名对文件进行写操作，通过另一个文件名也能知道写操作的效果，如下图。

   ![](https://ae01.alicdn.com/kf/H9649cfad4c204f00a991bd45bbdf6f29N.jpg)

2. 可以观察看到` ll -i`打印出来的第三列由1变为了2，这一列的意义是“有多少文件硬链接到本文件”，因为增加了file1_hl这个硬链接，所以file1的第三列变为2，反之file1也是硬链接到file1_hl上的，所以file1_hl也为2。

硬链的原理可见下图：

![](https://ae01.alicdn.com/kf/He3d79d5c5d5e468e8bceda57cac32aefU.jpg)

在[Linux文件系统基础篇](https://www.ravenxrz.ink/archives/basic-knowledge-of-linux-file-system-1.html)我们曾介绍过，inode和block是依托文件系统的，也就是说不同文件系统的相同inode是不同的文件。这个特点**决定了硬链只能在同一文件系统内使用**。除此之外，硬链也**不能链接目录**，原因在于：如果使用 hard link 链接到目录时， 链接的数据需要连同被链接目录下面的所有数据都创建链接，举例来说，如果你要将 /etc 使用实体链接创建一个 /etc_hd 的目录时，那么在 /etc_hd 下面的所有文件名同时都与 /etc 下面的文件名要创建 hard link 的，而不是仅链接到 /etc_hd 与 /etc 而已。 并且，未来如果需要在 /etc_hd 下面创建新文件时，连带的， /etc 下面的数据又得要创建一次 hard link ，因此造成环境相当大的复杂度。

总结来看，硬链的特点包括：

- 不能跨文件系统
- 不能链接目录

## 参考

- 鸟哥的Linux私房菜
- [Linux 软链接与硬链接](http://www.shaoguoji.cn/2018/01/28/linux-hard-soft-link/)
