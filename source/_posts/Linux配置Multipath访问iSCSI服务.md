---
title: Linux配置Multipath访问iSCSI服务
abbrlink: f30ddec7
date: 2020-03-05 18:15:00
tags: 
	- Multipath
	- iSCSI
category: Linux
---

系列文章：

- [linux软riad配置详解](https://www.ravenxrz.ink/archives/b85e04d0.html)
- [Linux配置iSCSI服务](https://www.ravenxrz.ink/archives/47bf0456.html)
- [Linux配置Multipath访问iSCSI服务](https://www.ravenxrz.ink/archives/f30ddec7.html)
本文架构参考自：https://blog.csdn.net/Dream_ya/article/details/88674952

### 1. Multipath简介

普通的电脑主机都是一个硬盘挂接到一个总线上，这里是一对一的关系。而到了有光纤组成的SAN环境，由于主机和存储通过了光纤交换机连接，这样的话，就构成了多对多的关系。也就是说，主机到存储可以有多条路径可以选择。主机到存储之间的IO由多条路径可以选择。

**实现功能**

1. 故障的切换和恢复
2. IO流量的负载均衡磁盘的虚拟化
3. 磁盘的虚拟化

<!--more -->

### 实验平台

|   主机   |                   IP                    | 操作系统 | 已有磁盘         |          安装服务           |
| :------: | :-------------------------------------: | :------: | ---------------- | :-------------------------: |
| Server端 | 10.10.10.1（eth1）、192.168.1.1（eth2） | Manjaro  | sda，制作的raid5 |     targetd，targetcli      |
| Client端 | 10.10.10.2（eth1）、192.168.1.2（eth2） |  ubuntu  | sda              | open-iscsi、multipath-tools |

这里你可能会遇到的问题：

- 如何添加多余的虚拟网卡？ -- 虚拟机->设置->添加即可

- 如何修改网卡名？ -- [点这里](<https://www.ravenxrz.ink/archives/e7c30e88.html>)

- 如何设置Server和Client的各个网卡ip及默认网关？

  这里简单说一下我是怎么配置的，首先我两个主机各有三张网卡，其中一张用于和外网通信，另外2张是Server和Client之间的互联。

  如，Server的网卡eth1，设置ip为10.10.10.1/24，则其默认网关设置为10.10.10.2，Client则反过来。

## 2. iscsi配置（客户端）

**Server端**如何配置iscsi已经在上一篇文章中写过了：https://www.ravenxrz.ink/archives/47bf0456.html

唯一需要注意的是，在配置portal的时候，采用默认即可，要不就将多个ip和port都create上,以及防火墙的放行。

现在来说说Client端：

**发现Server的iSCSI：**

![](https://pic.downk.cc/item/5e60d4f898271cb2b8a9aef1.jpg)

**登录：**

```
iscsiadm -m node -T iqn.2020-02.com.ravenxrz:server -l 
```

此时执行`fdisk -l`，你会发现被添加了很多设备（具体来说是3个，因为3张网卡）：

![](https://pic.downk.cc/item/5e60d5c198271cb2b8aa1e8a.jpg)

为了解决这个问题，就要使用到multipath了。

## 3. Multipath配置

**安装multipath工具包**

- CentOs：` yum -y install device-mapper-multipath`
- Ubntu: `apt install -y multipath-tools` 

**配置/etc/multipath.conf**

/etc/multipath.conf这个文件默认是没有得，需要手动创建：

```
defaults {
        user_friendly_names yes	# 如果是集群，则不该开启这个
        find_multipaths yes
}
multipaths {
    multipath {
        wwid    360014056393309b846f47bcae82517a0
        alias   mpatho
  }
}
```

需要修改的是wwid，执行`cat /etc/multipath/wwids`命令查看。

**启动multipathd守护进程**

`systemctl start multipathd`

**查看multipath信息**

```shell
multipath -ll # 或者multipath -rr
```

![](https://pic.downk.cc/item/5e60db6a98271cb2b8adae36.jpg)

可以看到mapthn这个设备有三条路径。

mapthn这个设备在 /dev/mapper下，具体而言为 `/dev/mapper/mapthn`，现在对这个设备操作就是唯一的了。
同时也仅能对/dev/mapper/mapthn做mount操作，如果mount其余"两个设备",会显示busy。这样就能保证不管是对从哪个接口获取的lun，
都可能够正确得进行操作。

### 参考

- [Linux Multipath+iscsi实现多路径配置](https://blog.csdn.net/Dream_ya/article/details/88674952)
- [Ubuntu系统下的多路径软件 DM Multipath 配置](https://www.cnblogs.com/pipci/p/9247858.html)