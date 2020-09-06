---
title: Ubuntu通用换源方法
categories: Linux
tags: 换源
abbrlink: 32b1479
date: 2020-06-06 20:28:36
---

一般来说，安装任何linux发行版后，我们做的第一步就是更换它的仓库源，然后才能愉快的下载各种东西。

这里就说一下ubuntu的通用换源方法：

```shell
# 备份源source.list
sudo cp /etc/apt/source.list /etc/apt/source.list.bak
# sed批量替换源地址
sudo sed -i 's/^\(deb\|deb-src\) \([^ ]*\) \(.*\)/\1 http:\/\/mirrors.aliyun.com\/ubuntu \3/' /etc/apt/sources.list
# update 生效
sudo apt update
```

ok， 就这么简单。