---
title: openwrt aria2错误记录
categories: 路由器
tags: aria
abbrlink: 4014c94e
date: 2020-05-02 22:04:30
---

在路由器上使用aria下载文件时，遇到了这样的问题：

### [SocketCore.cc:1015] errorCode=1 SSL/TLS handshake failure: unable to get local issuer certificate

其实就是因为没有证书，所以无法下载https文件。

下载安装`ca-certificates`即可。

![image-20200502220639345](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200502220639345.png)