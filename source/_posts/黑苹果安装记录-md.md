---
title: 黑苹果安装记录
categories: 杂文
toc: true
abbrlink: b8634fa6
date: 2019-05-24 13:59:33
tags:
---

## 1. 前言

> 又是一次闲的无聊的折腾，想着去年有一次心血来潮安装黑苹果却没成功，最后在淘宝费了100多才安装的经历。最近正好忙完毕设，就来作作死，安装一波。**这次文章也不是什么详细教程，主要是不想重搞一次来截图**，只是简单记一下自己安装的经历吧，不过有想安装黑苹果的小伙伴也可以看看大体流程，特别是第4节说的安装后应该做的事。

<!-- more -->
手上有Intel的笔记本和AMD的台式机，所以两台电脑都尝试过了，并且也都成功安装，下面分别简单说一下。

[Meting]
[Music server="netease" id="528484544" type="song"/]
[/Meting]


## 2. Intel笔记本篇

### 2.1 材料准备

Intel笔记本是几年前买的了，配置不行，所以闲着没怎么用，都说Intel的CPU好安装一点（实际从我的安装体验来看，用AMD的懒人包简单多了），那就从Intel笔记本入手了。准备了以下材料:

1. Mac OS 10.13.4 系统镜像
2. TransMac刻录工具
3. DiskGenius磁盘管理工具
4. U盘8G以上
5. **EFI** （这个是最重要的了，安装不了都是这个问题）
6. 非必须，各种驱动（最好能备好有线网卡驱动，无线基本没法驱动的，所以无线就去淘宝花个20来软妹币买个无线网卡就ok，有网才能做后面的各种事情嘛）

### 2.2 刻录镜像

安装好TransMac工具，并插入你的U盘。然后做以下两部：

step1: 格式化U盘（所以备份好你的U盘），根据提示写个U盘名字就ok~

![](https://pic.superbed.cn/item/5cfbacbd451253d178d93fcf.png)

step2: 刻录mac系统，选择Restore with Disk Image，然后选中的镜像就好。

![](https://pic.superbed.cn/item/5cfbacbe451253d178d9400b.png)

### 2.3 划分Mac系统分区

刻录好系统后，咱们就需要分区了。一般来说有两种方案，一个是直接拿一整块干净硬盘来做黑苹果系统，另一个就是在目前已经有的磁盘分区上划分一部分出来做系统。两种方式都很简单，但是**后期安装好后设置启动项略有区别。**

这里就不说明具体的划分分区的操作了，简单说来就是一整块硬盘直接新建一个卷不用格式化，划分新分区也是同理。建议分区大于30G。

### 2.4 最重要的一步，替换EFI

要替换EFI，首先要找到适合自己的EFI，我也是小白并不会自己配，所以就在网上找现成的，最后发现了这个网站：https://mp.weixin.qq.com/s/oYIrNOy8Co-SbKTPlGOlLw 的博主提供了不少EFI，虽然要5元付费（**我没有打广告，你通过其他路径也可以，我也是说我成功安装的一个过程**），下图是他的一个展示，值得注意一点的是，即使你付费了，他把百度云的密码给你，并且你也找到适合你的EFI下载后还需要他的一个解压密码，整个过程有点烦，不过他也是说的怕乱传播，我感觉有点繁琐就是了。

![](https://pic.superbed.cn/item/5cfbacbf451253d178d9404e.png)

如果你找到了合适的EFI文件后，就需要对U盘中的EFI文件进行替换了。具体替换过程如下：

step1： 打开DiskGenius工具，进入如图目录。

![](https://pic.superbed.cn/item/5cfbacc1451253d178d9409f.png)

step2: 直接把你下载的EFI文件复制粘贴过来，提示替换，选全部替换即可。

额外说明，下一步就要进行实际安装了，这里打开了DiskGenius，所以正好把你当前系统的EFI文件备份一下，避免安装过程中出错可能把本来系统的EFI文件损坏了。（本人有过类似经历，好在后来通过PE修复了），备份方式如下：

![](https://pic.superbed.cn/item/5cfbacc2451253d178d940f5.png)

三星的硬盘是我的主硬盘，同样进入到EFI文件夹下，可以看到我安装了Windows和Ubuntu两个系统，把这个EFI分区复制到某个自己能识别的目录下，以便后期使用。

### 2.5 正式安装

step1: 关机进入BIOS，进行如下设置：

- SATA Ports = AHCI

- IOMMU = Disabled

- APU = Disabled

- HPET = Enabled

- EHCI Hands-off = Enabled

- XHCI Hands-off = Enabled

- Serial Port = Disabled

- Parallel Port = Disabled

step2:  设置U盘为第一启动项

step3: 重启，进入Clover启动，**选择U盘安装MacOS**

step4: 进行安装界面，选择磁盘工具，抹除上述分区，分区文件系统选择**APFS**

spte5:重回安装界面，进行安装，之后会自动关机

step6: 重启从U盘启动，进入到Clover，**这次不选择从U盘安装MacOS**，而是选择从硬盘中启动（反正就是选择除U盘的Apple标志的另外一个Apple标志），关机重启

step7: 还是以U盘启动，进入到Clover，这次依然选择从硬盘启动MacOS，之后应该就是正常设定。

## 3. AMD台式机

都说AMD装机要比Intel难，其实从我个人的安装体验来说，AMD安装简单多了，毕竟有懒人包。
![](https://pic.superbed.cn/item/5cfbacc4451253d178d94145.jpg)


### 3.1 配置说明

- AMD 2600x
- 影驰：NVIDA 1060 
- 东芝家的SSD。本来想用主力的PM981的，但是从远景上得知PM981不能安装

- 主板 X470

### 3.2 安装过程

step1: 下载**聆曦AMD Ryzen黑苹果镜像**，百度就行，有很多。**不建议下载mojave_10.14.3版本的，因为N卡驱动还不支持，最后导致显示效果很垃圾**。

step2: 刻录镜像、划分分区和上述过程一样，但是**不用替换EFI**。

step3: 设置BIOS的一些选项和Intel相同，在设置从U盘启动。

------------------

下面的步骤就和Intel安装稍有不同了。

step4: 进入Clover，**选择从U盘安装**，进入以后，抹除分区，选择文件系统**APFS**，然后进行系统安装，重启。

step5: 重启进入Clover，**选择从U盘安装**，打开终端，输入`lingxi_one`，跑完代码后，重启。

step6: 重启进入Clover，**选择从硬盘启动**，然后就是漫长的安装系统过程，跑完后会自动重启。

step7: 重启进入Clover，**选择从U盘安装**，打开终端，输入`lingxi_two`，跑完代码后，重启。

step8: 重启进入Clover，**选择从硬盘启动**, 然后就能进入系统了。

## 4. 安装之后的事

如果顺利安装好了，也别多高兴。这仅仅是折腾的第一步，黑苹果最难受的就是折腾驱动。这里给出后续需要做的几点工作。

进入系统的第一件事，打开终端，执行以下命令:

```
sudo spctl --master-disable 
```
上述命令是为了打开任意安装来源，不然无法安装一些破解软件。

必备的软件:

1. clover configuration -- 配置驱动，EFI必备

2. KCPMUtilityPro -- 安装驱动必备

3. VoodooHDA.kext -- 万能声卡驱动

4. VoodooPS2Controller.kext -- 键盘和触摸板驱动

5. 无线网卡几乎不能驱动，所以上淘宝花个20来RMB买个无线网卡吧。

6. NVIDA WebDriver -- N卡驱动，记得找对应苹果系统版本的。

   上述所有软件都可以在**“黑苹果社区”**网站找到。

这里给两个视频教程。

视频教程1： AMD安装黑苹果教程
<iframe id="spkj1" src="//player.bilibili.com/player.html?aid=41865008&cid=73505470&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" width=100%> </iframe>

视频教程2: 安装N卡驱动教程，从7分20秒开始看：

<iframe id="spkj2" src="//player.bilibili.com/player.html?aid=35160335&cid=61653453&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" width=100%> </iframe>
!!!
<script type="text/javascript">  
document.getElementById("spkj1").style.height=document.getElementById("spkj1").scrollWidth*0.76+"px";</script>
!!!
!!!
<script type="text/javascript">  
document.getElementById("spkj2").style.height=document.getElementById("spkj2").scrollWidth*0.76+"px";</script>
!!!




常用的软件：

1. CleanMyMacXChinese4.4.0 -- 清理软件
2. Paste -- 剪贴板历史
3. ssr-mac -- 禾斗xue上网


