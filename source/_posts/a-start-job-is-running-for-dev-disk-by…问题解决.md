---
title: a start job is running for dev-disk-by…问题解决
abbrlink: d633803a
date: 2020-02-17 10:54:49
categories: Linux
tags: 扩容
---

给linux虚拟机扩容后，开机非常慢，主要是卡在了一行“a start job is running for dev-disk-by…”，后来发现是因为在扩容的时候，重新分配了swap分区，导致先swap分区和原swap分区的uuid不同，系统无法加载swap分区。

解决方案如下：

1. 执行

```
sudo blkid
```

记录当前swap分区的uuid。

![](https://pic.downk.cc/item/5e4a018648b86553ee02a551.jpg)

2. 替换/etc/fstab文件中的swap分区uuid即可。

![](https://pic.downk.cc/item/5e4a015448b86553ee029a40.jpg)