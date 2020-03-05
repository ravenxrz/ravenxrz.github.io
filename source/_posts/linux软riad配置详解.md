---
title: linux软riad配置详解
abbrlink: b85e04d0
date: 2020-03-01 19:59:04
tags:
---
系列文章：

- [linux软riad配置详解](https://www.ravenxrz.ink/archives/b85e04d0.html)

- [Linux配置iSCSI服务](https://www.ravenxrz.ink/archives/e7c30e88.html)
- [Linux配置Multipath访问iSCSI服务](https://www.ravenxrz.ink/archives/f30ddec7.html)

本文以raid5为例说明如何在linux下配置raid。

如果不熟悉raid的基本概念，请查看：[知识扫盲-RAID0，RAID1，RAID5，RAID6，RAID10简介](https://www.ravenxrz.ink/archives/f0834f1f.html)

[软raid？ 硬raid？](https://blog.csdn.net/csdn100861/article/details/51439718)

实验平台：VMware

<!-- more -->
## 1. 新建磁盘

我们知道raid5至少需要3块磁盘，为了测试热备份，我们一共新建4个虚拟磁盘。其中3块（2块工作，1块校验盘）做raid5，另一块做热备份。 

将linux完全关机，选择虚拟机->设置，进入如下界面，选择”添加“：

![](https://pic.downk.cc/item/5e5ba4cb773ff94fc654860a.jpg)

然后选择 硬盘->SCSI->创建新虚拟磁盘，然后给定虚拟盘一个大小即可，我给的是2G。这样就创键好1个虚拟盘了。

重复上述步骤，共得到4个2GB大小的磁盘。如下：

![](https://pic.downk.cc/item/5e5ba570773ff94fc654a251.jpg)

## 2. 虚拟盘分区

现在我们需要对在第一步中创建的4个虚拟盘进行分区，我使用的fdisk工具。[fdisk使用教程](https://www.cnblogs.com/theladyflower/archive/2011/08/07/2130170.html),

以/dev/sdb为例，执行：

```shell
sudo fdisk /dev/sdb
```

- 输入n,创建新分区
- 一路Enter即可
- 最后输入w，将所有修改结果写入。

执行`sudo fdisk -l /dev/sdb`可查看分区结果：

![](https://pic.downk.cc/item/5e5ba663773ff94fc654c072.jpg)

对/`dev/sdc /dev/sdd /dev/sde`重复上述操作。最后对得到`/dev/sdb1 /dev/sdc1  /dev/sdd1 /dev/sde1`四个分区。如下：

![](https://pic.downk.cc/item/5e5ba6d4773ff94fc654cf37.jpg)

## 3. 建立Raid5

建立raid主要使用的是mdadm命令，如果你的系统上找不到该命令，请根据自己的发行版软件管理包进行安装，如：

```shell
apt install mdadm  # debian系
pacman -S mdadm	# arch系
```

接着执行如下命令，创建raid5：

```shell
sudo mdadm --create /dev/md/raid5 --level=5 --raid-devices=3 --spare-devices=1 /dev/sd[b-e]1
```

执行`sudo mdadm -D /dev/md/raid5`查看创建的raid5：

![](https://pic.downk.cc/item/5e5ba836773ff94fc654f2d4.jpg)

采用fdisk对/dev/md/raid5分区，做法同上，这里不赘述。

接着需要对raid5格式化建立文件系统：

执行`sudo mkfs.ext4 /dev/md/raid5`

## 4. 测试raid5

挂在raid5，

```shell
sudo mkdir /mnt/raid5
sudo mount /dev/md/raid5 /mnt/raid5
cd /mnt/raid5
```

执行df -Th 查看挂载结果：

![](https://pic.downk.cc/item/5e5ba914773ff94fc65507a6.jpg)

### 测试热备份效果

我们可以通过 

```
mdadm /path/to/raid_device_file -f /path/to/disk_device_file 
```



来模拟某个盘出现故障，如：

```shell
sudo mdadm /dev/md/raid5 -f /dev/sdb1
```

再次执行

```shell
sudo mdadm -D /dev/md/raid5
```

![](https://pic.downk.cc/item/5e5bab86773ff94fc65548b2.png)

可以看到 `/dev/sde1`盘作为热备起到了rebuild的效果。

如果这里没有`/dev/sde1`作为热备，数据也不会丢失，但是如果再有一个盘down掉，那数据可就找不回来了。

## 5. 卸载raid5

执行:

```shell
sudo umount /dev/md/raid5
sudo mdadm -S /dev/md/raid5 
```

综上！

