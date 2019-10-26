---
title: 旧手机再利用-基于Aria2的下载机
date: 2018-10-12 13:58:52
categories: 杂文
toc: true
tags:
---


## 1.前言

> 最近换手机，因为原来的旧手机（一加三）性能也还算行，所以就百度了一下看能不能把这台手机给用起来，发现搭建个**基于aria的远程下载机**比较有趣,于是乎就有了本篇文章。
<!-- more -->
什么是Aria？

> aria2 is a lightweight multi-protocol & multi-source command-line download utility. It supports [HTTP](https://wiki.archlinux.org/index.php/HTTP)/[HTTPS](https://wiki.archlinux.org/index.php/HTTPS), [FTP](https://wiki.archlinux.org/index.php/FTP), [BitTorrent](https://wiki.archlinux.org/index.php/BitTorrent) and [Metalink](https://en.wikipedia.org/wiki/Metalink). aria2 can be manipulated via built-in [JSON-RPC](https://en.wikipedia.org/wiki/JSON-RPC) and [XML-RPC](https://en.wikipedia.org/wiki/XML-RPC) interfaces. (抄自：https://wiki.archlinux.org/index.php/aria2)

翻译一下就是:

> aria2是一个轻量级的下载工具，支持多协议（HTPP/HTPPS,FTP,BT,MetaLink)，通过JSON-RPC或XML-RPC进行配置。

为什么要使用Aria？

1. aria轻量，可以尝试同时用迅雷和aria下东西，观察内存占用，你会发现迅雷占用明显高于aria
2. aria跨平台，几乎所有平台都可以使用
3. 用aria**实现远程下载**，使用NAS,路由器，或者旧手机都可以完成
4. 破解百度网盘限速

Aria有什么缺点？

1. 原Aria是纯命令行的，导致很多人望而止步，不过现在经过大神的开发，已经有很多优秀的web管理工具。
2. Aria不支持迅雷的专用链接
3. Aria的Bt下载速度很慢

## 2. 准备工作

1. 一台root的Android机
2. Aria服务：Aria2cdroid或者Aria2Android（前者似乎在酷安市场已下架，但是在码云上有该软件的源码，有一定的Android编程知识的同学可以clone下来编译一下）
3. Aria控制器：不推荐使用酷安市场上的Aria2远程下载，该软件有个无限弹出连接主页面的bug。反之使用web控制端是个相当不错的选择。这里推荐使用：[Ariang](http://euch.gotoip1.com/aria-ng/#!/status)

4. 有了上述的几个东西已经可以实现在局域网中下载东西了，但是要做到外网远程控制我们的下载鸡必要做内网穿透才行，这里我用的是[Sunny-Ngork](https://www.ngrok.cc)。
5. NeoTerm终端
6. BusyBox

**以上大多数软件均可在酷安市场找到。**

## 3.开始搭建

### 3.1 root你的手机

这个我就不细说了，手机解锁->刷入twrp recovery-> 刷入root包。

### 3.2 开启你的Aria后台服务

![Screenshot_20181101-142810.png](https://pic3.superbed.cn/item/5cfbb583451253d178d9c758.png)


打开Aria2（如果你下载的是Aria2Android，就打开Aria2Android）

![Screenshot_20181101-142828.png](https://pic.superbed.cn/item/5cfbb58b451253d178d9c7e5.png)


点击右上角的的三角形，开启服务。

![Screenshot_20181101-142834.png](https://pic2.superbed.cn/item/5cfbb58c451253d178d9c827.png)


记住下面的东西：
第一行的ip:192.168.43.122
第二行的Secret：123456（如果你没改的就是这个，另外Aria2Android的同学也注意一下是多少）


### 3.3 打开Web控制
打开网址：[Ariang](http://euch.gotoip1.com/aria-ng/#!/status())

![web1.png](https://pic.superbed.cn/item/5cfbb58e451253d178d9c85f.png)

![web2.png](https://pic1.superbed.cn/item/5cfbb58f451253d178d9c893.png)

![web3.png](https://pic.superbed.cn/item/5cfbb590451253d178d9c8c4.png)



到这一步局域网中的下载机已经搭建好了，可以来试试效果。

![web4.png](https://pic1.superbed.cn/item/5cfbb592451253d178d9c901.png)


先不用在意我这里的网速。。。（如果你搭建成功，并且下载的是http的文件速度基本上是满速的）

### 3.4 内网穿透

到上一步为止都还是非常容易的，但是实用性还不算高，毕竟在局域网直接用控制端的主机来下载不就好了吗，或者每次出门前就把要下载的东西设置好。不过要是能够远程控制不是更美好吗？

#### 3.4.1 配置环境

打开**busybox**，给root权限，**等加载进度条跑完**，然后选择install就行了。

#### 3.4.2 Sunny-Ngork配置

打开[Sunny-Ngork](https://www.ngrok.cc)官网，注册一个账号进入到后台：

![web5.png](https://pic.superbed.cn/item/5cfbb594451253d178d9c93b.png)

![web6.png](https://pic.superbed.cn/item/5cfbb595451253d178d9c973.png)

![web7.png](https://pic.superbed.cn/item/5cfbb596451253d178d9c9b6.png)

![web8.png](https://pic.superbed.cn/item/5cfbb598451253d178d9c9ee.png)


**将下载下来的linux arm文件放到手机里，并记住放置的目录,并解压，会得到一个名字为sunny脚本**

#### 3.4.3 运行外网穿透程序

打开**NeoTerm**终端，获取超级权限:`su`，会提示给权限。

cd到上一步中你解压的脚本目录下:`cd xxxx`

把sunny文件移动到/system层级下（不移动到这个层级下无法更改该文件的权限）:`mv sunny /system`

cd到system目录:`cd /system`

更改sunny文件权限：`chmod 755 sunny`


运行:`sunny clientid 隧道id`

如果配置成功应该由如下界面:

![Screenshot_20181101-150632.png](https://pic.superbed.cn/item/5cfbb599451253d178d9ca24.png)


#### 3.4.4 更新远端控制程序

![web9.png](https://pic.superbed.cn/item/5cfbb59b451253d178d9ca5b.png)


端口一定要填80端口!

端口一定要填80端口!

端口一定要填80端口!

嗯，到这里我们就完成了远程端搭建，快去试试吧。

## 4.额外的问题

1. **怎么用aria怎么下载百度网盘里的资源？**

   可以用chrome的插件BaiduExporter，但是chrome商店已经下架了，好在该插件在github上有，这里贴出地址：https://github.com/acgotaku/BaiduExporter

![web10.png](https://pic.superbed.cn/item/5cfbb59c451253d178d9ca97.png)


把这个下载下来，然后打开chrome的扩展程序，拖进去。**嗯，完美的没有安装成功**，chrome限制了没有在商店上线的插件。

正确的做法是，把crx后缀该为rar，然后解压，最后在chrome的扩展程序中**加载已解压程序**，就ok了。

使用方法，找到资源，点击导出下载，推送到aria（可能需要设置一下，和ariang面板差不多，就不说了）

![web11.png](https://pic.superbed.cn/item/5cfbb59e451253d178d9cadb.png)


2. **为什么我的aria下载bt，磁力没速度，但是迅雷却跑得很快？**

根据我百度的资料来看（**不一定准确**），迅雷之所以跑得快，是因为迅雷流氓，在后台上传你下载的文件信息。我们都知道bt下载需要去别人手中取文件块，所以人越多下载得越快，迅雷这样做就导致很容易发现其他下载者，而aria就不行了。至于为什么aria速度完全没有的情况，是因为没有配置bt-tracker：把下面这些复制到你的bt-tracker中（在ariang面板中的bt设置中有这个选项）。

```
udp://tracker.coppersurfer.tk:6969/announce

udp://tracker.opentrackr.org:1337/announce

udp://9.rarbg.to:2710/announce

udp://tracker.internetwarriors.net:1337/announce
	
udp://exodus.desync.com:6969/announce

udp://tracker.vanitycore.co:6969/announce

udp://public.popcorn-tracker.org:6969/announce

udp://explodie.org:6969/announce

udp://tracker1.itzmx.com:8080/announce

udp://tracker.torrent.eu.org:451/announce

udp://tracker.tiny-vps.com:6969/announce

udp://tracker.port443.xyz:6969/announce

udp://thetracker.org:80/announce

udp://open.stealth.si:80/announce

udp://open.demonii.si:1337/announce

udp://ipv4.tracker.harry.lu:80/announce

udp://denis.stalker.upeer.me:6969/announce

udp://bt.xxx-tracker.com:2710/announce

udp://tracker.cypherpunks.ru:6969/announce

udp://retracker.lanta-net.ru:2710/announce

```

**不过，tracker更新的很快，当你看到这篇文章的时候，应该已经过时了**，所以这里贴出一个trakcer更新的仓库，需要的时候自己去看看吧:https://github.com/ngosang/trackerslist。

每次更改设置后，**记住重启aria服务**。

好了，愉快的去享受远程下载吧。hiahia。


