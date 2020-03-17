---
title: Linux配置iSCSI服务
abbrlink: 47bf0456
date: 2020-03-05 17:05:45
tags: Linux
---
系列文章：

- [linux软riad配置详解](https://www.ravenxrz.ink/archives/b85e04d0.html)
- [Linux配置iSCSI服务](https://www.ravenxrz.ink/archives/47bf0456.html)
- [Linux配置Multipath访问iSCSI服务](https://www.ravenxrz.ink/archives/f30ddec7.html)


在[《linux软riad配置详解》](https://www.ravenxrz.ink/archives/b85e04d0.html)中讲解了如何在Linux中配置软raid，本文中我们将讲解如何让这个raid5可被客户端通过ISCSI服务访问。

实验平台：

Server端： Manjaro 内核4.9.214

Client端： Ubuntu18 内核 5.3.40

当然了，这里强烈推荐Server和Client端都是用**CentOs**，个人是因为笔记本已经装了Manjaro和Ubuntu不方便更换，所以就将就使用了。

<!-- more -->

## 0. 基础知识

iSCSI 的全称是: Internet 小型计算机系统接口，是一个基于 TCP/IP 的协议，主要用于通过 IP 网络仿真 SCSI，从而为远程块存储设备提供数据传输和管理。说白了，就是通过网络由专门的服务器提供存储管理，以实现数据的远程储存，便于数据的集中管理，从而简化了数据复制、迁移和容灾。

### **常用的基本概念**

|       名词        |                             解释                             |
| :---------------: | :----------------------------------------------------------: |
|        ACL        | 访问权限控制列表，用来验证客户端启动器的访问，通常是客户端   iSCSI 启动器的 IQN 名称 |
|        IQN        |    用于标识单个 iSCSI   目标和启动器的唯一名称(全部小写)     |
|        WWN        |           用于标识单个光纤通道端口和节点的唯一编号           |
|      TARGET       |                   iSCSI 服务器上的存储资源                   |
|        LUN        |                    iSCSI 服务器上的块设备                    |
| initiator(启动器) |               以软件或硬件实施的 iSCSI 客户端                |
|       NODE        |                  单个 iSCSI 启动器或者目标                   |
|        TPG        |              启动器或者目标上的单个 IP 连接地址              |
|      Portal       |                        网络接口及端口                        |

*IQN 的格式为：iqn.年份-月份.com|cn|net|org.域名:自定义标识，如：iqn.2018-05.com.test:desktop;其中的字母均应为小写，即使输入时包含大写，命令执行后，系统会自动转换成小写。*



## 1. Server配置

 **安装 iSCSI 服务端程序以及其配置命令工具**

```shell
yay -S targetd targetcli # 这里用了aur，其余平台按各自的包管理器即可
```

**启动targtd服务，并将其添加到开机自启中。**

```sh3ell
systemctl start targetd
systemctl enable targetd
```

**使用 targetcli 配置命令**

targetcli 是用于管理 iSCSI 服务端存储资源的专用配置命令，它能够提供类似于 fdisk 命令的交互式配置功能，将 iSCSI 共享资源的配置内容抽象成“目录”的形式，我们只需将各类配置信息填入到相应的“目录”中即可。

这里的难点主要在于认识每个“参数目录”的作用。当把配置参数正确地填写到“目录”中后，iSCSI 服务端就可以提供共享资源服务了。

**配置块设配资源池**

/backstores/block 是 iSCSI服务端配置共享设备的位置。在 targetcli 交互命令环境下，使用 create 命令把上述所创建的 RAID 5 磁盘阵列 /dev/md.raid5 文件加入到配置共享设备的“资源池”中，并将该文件重新命名为 disk0。

额外补充，/backstores下的四个目录的含义：

|  目录   |                             解释                             |
| :-----: | :----------------------------------------------------------: |
|  block  | 块设备，磁盘驱动器、磁盘分区、逻辑卷、以及服务器上定义的任何 b 类型的设备文件 |
| fileio  | 在服务器上生成的一个指定大小的文件，类似于虚拟机中的虚拟磁盘 |
|  pscsi  |                    物理 SCSI，通常不使用                     |
| ramdisk |        内存盘，其中存储的数据在服务器重启后将全部丢失        |

具体操作：

切换到root用户，执行targetcli命令,进入targetcli管理界面，help命令可显示帮助

```shell
cd /backstores/block
create disk0 /dev/md/raid05 # create 资源名 raid设备路径(不限定是raid，只是这里以为例)
```

**创建IQN：**

```
cd /iscsi/
create iqn.2020-03.com.ravenxrz:server
```

**设置ACL访问列表**：

```shell
cd /iscsi/iqn.2020-02.com.ravenxrz:server/tpg1/acls
create iqn.2020-03.com.ravenxrz:client		# 这个要记住，后面的客户端通过配置这个来访问资源
```

**创建LUN:**

```shell
cd ../luns
create /backstores/block/disk0
```

**设置监听IP和端口：**

如果默认可通过多个网卡访问的话，可以不修改（后文配置Multipath访问时，就可以采用默认的配置），否则：

```
# 删除默认0.0.0.0 3260端口 
cd iscsi/iqn.2020-03.com.ravenxrz:server/tpg1/portals/
delete 0.0.0.0 3260
# 配置你想用于被访问的网卡的ip，可通过ip addr命令查看，如下图
create 192.168.18.130 3260
```

![](https://pic.downk.cc/item/5e60ca5198271cb2b8a214fc.jpg)

最后，贴出**完整配置**：

![](https://pic.downk.cc/item/5e60cf6298271cb2b8a5942d.png)

退出，重启服务，更新配置

```shell
exit
# 这里应该回到了普通termial
systemctl restart targetd
```

**设置防火墙，放行3260端口**

如果是在个人的PC上的话，这一步应该没有。

```
firewall-cmd --permanent --add-port=3260/tcp
firewall-cmd --reload
```

## 2. Client端配置

**安装initiator工具包：**

- CenteOs：`yum install iscsi-initiator-utils`
- Ubuntu: `apt install open-iscsi`

**修改initiatorname**，这一步很重要，就是把Server中配置的acl名字复制下来，然后修改这个文件：

vim /etc/iscsi/initiatorname.iscsi

![](https://pic.downk.cc/item/5e60cc1d98271cb2b8a35c7b.jpg)

*我这里是2020-02，上文中配的是2020-03，因为我之前已经配置过了，所以没有修改*，如果你一直按照之前的教程配置到这里的话，把2020-02替换为2020-03才行。

**查找iSCSItargets主机的targetname**

```shell
iscsiadm --mode discovery --type sendtargets --portal 192.168.18.130	# 替换ip
```

**登录iscsi**

```shell
iscsiadm --mode node --targetname iqn.2020-03.com.ravenxrz:server --portal 192.168.18.130:3260 --login
```



*如果是widnows端登录的话，将会非常简单，这里不作介绍，可参考这篇文章的末尾介绍。* <https://www.jianshu.com/p/648b8be1499f>

## 3. 验证及使用

```
fdisk -l
```

![](https://pic.downk.cc/item/5e60cd2898271cb2b8a41232.jpg)

接下来可以分区，格式化，make文件系统，挂载等，就像和本地磁盘一样。

## 4. 退出及清除

**Client端退出**

取消挂在，然后退出：

```shell
umount xxx
iscsiadm -m node -T iqn.2020-03.com.ravenxrz:server -u
```

**Server清除所有配置**

清楚配置和配置过程就是一个逆过程，分别删除portals、acls、luns、iSCSI Targets和后端存储即可。



## 5. 参考

- [[iSCSI配置与卸载](https://www.cnblogs.com/iouwenbo/p/10229055.html)](https://www.cnblogs.com/iouwenbo/p/10229055.html)

- [使用 iSCSI 服务部署网络存储](https://www.jianshu.com/p/648b8be1499f)

- [Linux就该这么学 | 第17章 部署 iSCSI 网络存储]( https://www.jianshu.com/p/78bedd69bd9d)

