---
title: 使用find指令查找文件遇见的一点小坑
categories: Linux
toc: true
abbrlink: dc78766f
date: 2019-05-05 13:58:38
tags:
---


## 1. 前言

最近在学习`linux`内核数据结构，准备找找队列的头文件`kfifo.h`，但是通过`/lib/modules/内核版本/build`下面使用`find`指令找，怎么都找不到。。。
<!-- more -->
## 2. 问题

如图，使用`find`指令找不到

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbad0f451253d178d94b6e.png)

但是使用`locate`指令却找得到:

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbad10451253d178d94bc0.png)

这就让我很纳闷了，最后读了一下`find`的`manual page`，发现了问题所在，

![在这里插入图片描述](https://pic2.superbed.cn/item/5cfbad12451253d178d94c17.png)

咦，好像这里出现了符号链接，而且我的当前目录下的文件目录颜色也怪怪的。

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbad15451253d178d94c7b.png)

好吧，真的是因为符号链接的问题造成了无法搜索到文件。加上“-L”指令，搞定。

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbad19451253d178d94cc8.png)



## 3. 总结

`find` 指令默认不开启软链接的搜索

