---
title: 从零开始搭建属于自己的博客系列教程二-域名绑定
date: 2019-05-04 13:58:32
categories: 博客搭建
toc: true
tags:
---

## 1.  写在最前面

> 折腾了整整一个五一，总算是完成了自己的博客系统。为了给想要搭建自己的博客，但是又和我一样是技术小白的人铺点路，所以准备开个博客搭建的系列教程，避免后来人像我一样走了一大堆弯路。

先贴上自己的博客网站：https://www.ravenxrz.ink   
<!-- more -->
系列教程目录：

- [从零开始搭建属于自己的博客系列教程零 -- 绪论](https://www.ravenxrz.ink/archives/ck27a6coa001t08vm7dom4ddk/)
- [从零开始搭建属于自己的博客系列教程一 -- 博客系统搭建](https://www.ravenxrz.ink/archives/ck27a6co2001d08vm37fp2jrz/)
- [从零开始搭建属于自己的博客系列教程二 -- 域名绑定](https://www.ravenxrz.ink/archives/ck27a6co5001i08vmexcp0gm3/)
- [从零开始搭建属于自己的博客系列教程三 -- 主题选择与CommentToMail插件介绍](https://www.ravenxrz.ink/archives/ck27a6cp5003u08vm39sqe1zg/)
- [从零开始搭建属于自己的博客系列教程四 -- 去掉烦人的index.php后缀与常用插件介绍](https://www.ravenxrz.ink/archives/ck27a6co9001r08vm7zd936xx/)
- [从零开始搭建属于自己的博客系列教程五 -- 图床选择与博客迁移](https://www.ravenxrz.ink/archives/ck27a6co7001l08vm695m3jwl/)
- [从零开始搭建属于自己的博客系列教程六 -- 博客系统备份](https://www.ravenxrz.ink/archives/ck27a6co8001o08vme47fc50a/)
- [从零开始搭建属于自己的博客系列教程七 -- 开启全站HTTPS](https://www.ravenxrz.ink/archives/ck27a6co4001g08vm38779qtt/)

## 2. 域名绑定

本篇讲解如何购买域名与绑定。

### 2.1 域名购买

我是在阿里上购买：https://wanwang.aliyun.com/ 

购买的操作这里就不写了，选定合适域名，支付即可，需要实名认证。

### 2.2 域名绑定

然后就到了域名和ip绑定的阶段。

![](https://pic.superbed.cn/item/5cfbb5b1451253d178d9cd55.png)

![](https://pic.superbed.cn/item/5cfbb5b2451253d178d9cd8d.png)

需要添加两条记录：

![](https://pic.superbed.cn/item/5cfbb5b9451253d178d9ce06.png)

仿照上图，再添加一个www的记录。这样域名解析时，既可以使用`ravenxrz.ink`也可以使用`www.ravenxrz.ink`。

填完后也不是立即能够解析的，需要一定的时间才能提交到DNS服务器，据官网介绍需要48小时才能全球同步，我是在5小时后发现能够使用域名登陆的。

