---
title: leveldb源码阅读记录-leveldbutil工具
categories: leveldb
abbrlink: 2938ff32
date: 2020-10-12 09:52:34
tags:
---

从本文之后，我们将正式分析leveldb内部组件，如memtable,sstable, log, manifest等。

<!--more-->

 如果你已经搭建了leveldb的源码debug环境并做过简单测试，你会发现leveldb中存在以下几类文件：

- sstable文件，sst或ldb结尾
- log文件，保证写入的持久性
- MANIFEST, 记录每次修改操作的增量，记录sstable的元数据
- CURRENT， 指向当前该使用哪个MANIFEST
- LOCK 锁相关
- LOG.old 日志

一般情况下，除了CURRENT和LOG.old文件可以双击查看，其它文件都是没办法查看的，因为它们都不是txt文件，而是经过leveldb按照自身设计而存储的。但是有没有办法查看这些文件的内容呢？有，leveldb为我们提供了一个工具---**leveldb/db/leveldbutil.cc**

我们将它编译成可执行文件后，执行 ./levelutil dump [要查看的文件] 即可打印相关文件的信息，如，我要查看log文件：

![](https://pic.downk.cc/item/5f81b4da1cd1bbb86b526c02.jpg)

知道了这个工具，在后续查看文件布局时就会比较方便了。

## 总结

这篇文章介绍了查看 log、sstable、manifest等文件的工具–leveldbutil. 文本是”前置知识“的最后一篇，从下文开始，我们将分析leveldb中log文件。