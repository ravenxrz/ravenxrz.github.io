---
title: Ubuntu18修改网卡接口名
tags: 
	- 网卡接口
abbrlink: e7c30e88
date: 2020-03-05 15:40:28
categories: Linux
---

ubuntu18上的默认网卡名为ens33，添加了网卡后的名字也“没什么规律”，所以这里说一下如何将ens33这类名字更改为原来的eth0,eth1的方法。

方法很简单：

1. 编辑/etc/default/grub

   ```java
   root@ubuntu:~# vi /etc/default/grub
   
   找到GRUB_CMDLINE_LINUX=""
   
   改为GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"
   ```

2. 重新生成GRUB的启动菜单配置文件(/boot/grub/grub.cfg)

```
root@ubuntu:~# update-grub
```

3. reboot即可

接下来添加网卡的名字都会按照ethx的方式来添加。

