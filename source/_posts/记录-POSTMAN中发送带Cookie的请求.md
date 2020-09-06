---
title: 记录-POSTMAN中发送带Cookie的请求
categories: 杂文
toc: true
abbrlink: '41155787'
date: 2019-08-21 13:59:17
tags:
	- POSTMAN
	- Cooklie
---

postman是个很出名的接口调试助手，虽然他已经出了独立的应用，但是我依然坚持在使用chrome插件版本。下面说下怎么在postman中调试带cookie的请求。

![](https://ae01.alicdn.com/kf/H8a407e9405014962862e42a32298927b1.jpg)

<!-- more -->
chrome出于安全考虑，使得postman不能直接调试带cookie的请求，于是我们需要多安装一个插件，叫postman-interceptor：

![](https://ae01.alicdn.com/kf/Hedf0972cae804f5eb52255ac8c25894cD.jpg)

安装并打开后，输入你要调试的url，在Headers栏下填写你的Cookie信息：

![](https://ae01.alicdn.com/kf/H719d4790a49344ffa128ada8be9742aaI.jpg)

**postman中的cookie有两种填写方式：**

1. 将所有cookie的键值对都写在一栏上：key填写cookie，value填写真正的cookie键值对，格式为cookie_key1=cookie_value1; cookie_key2=cookie_value2多个cookie对用分号隔开。

2. 一栏写一个键值对，每栏左侧key都填写cooike，右侧value依然填写真正的cookie键值对(cookie_key=cookie_value)，如:

   ![](https://ae01.alicdn.com/kf/Ha5d4125075e1450aa4e12c5d89a14a0cR.jpg)


