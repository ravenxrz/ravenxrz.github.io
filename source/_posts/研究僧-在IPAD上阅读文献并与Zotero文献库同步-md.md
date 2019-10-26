---
title: 研究僧-在IPAD上阅读文献并与Zotero文献库同步
date: 2019-07-09 13:59:07
categories: 杂文
toc: true
tags: 
---

## 1. 前言

>  下半年就要正式进入研究僧生活了，研究僧注定要阅读一大堆论文嘛，所以这篇文章就算是提前为阅读文献做准备咯。
>
>  相信常年累月阅读文献的你一定会有一个自己的文献库，并用某款文献管理软件维护着。我来说说个人的一点小经历吧，文献管理软件我用过两款，一个是EndNote一个是**zotero**，经过综合对比，最终我选择了**zotero**。

<!-- more -->

>
>  然而最近面临一个问题，我手上有一个ipad，自从咬咬牙入了apple pencil后，发现拿ipad来读pdf真的是超棒，批注、记笔记都很nice，所以就萌生了拿ipad来读文献的想法。我想实现的功能很简单:
>
>  - **将PC端的zotero文献库同步到IPAD上来，PC批注某篇paper后，IPAD上可以得到批注后的结果，反之亦然。**
>
>  经过一波百度、google后，我发现大多方法都是让在ipad上安装papership这款软件，我立马就试了试，简单说下效果：
>
>  **优点：**能够完美实现文献库的文件同步，包括目录树，一个item下有多个附件均可同步下来。
>
>  缺点：如果想要在ipad上批注并同步回PC端，需要**付费68元**。另外。我也简单尝试了下它的批注功能，发现并不完美。
>
>  zotero文献库图与Papgeship 同步后的效果如图：
>
>  ![](https://ae01.alicdn.com/kf/HTB1ZV6dXAL0gK0jSZFtq6xQCXXap.jpg)
>
>  ![](https://ae01.alicdn.com/kf/HTB13sLbXAP2gK0jSZPx761cQpXa2.png)
>
>  之后继续搜索和尝试，发现另一种比较完美的解决方案。于是，就有了今天这篇分享。

最终实现的视频Demo：
<iframe id="spkj" src="//player.bilibili.com/player.html?aid=58535591&cid=102086302&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" width=100%> </iframe>
!!!
<script type="text/javascript">  
document.getElementById("spkj").style.height=document.getElementById("spkj").scrollWidth*0.76+"px";
</script>
!!!



## 2. 材料准备

需要用到的软件与插件较多，清单如下：

- [zotero PC端](https://www.zotero.org/)
- zotero Chrome插件（和本文同步无关，可以不安装，但是你都安装zotero软件了，不安装这个插件那简直是浪费）
- ZotFile 插件 for zotero（ZotFile插件安装说明见下文）
  **链接：https://pan.baidu.com/s/1aKMs56HpMjRC_IbgGCWRGg  提取码：tzo5 。请勿安装官方版本，因为官网目前在PC到IPAD的同步上还有点问题，这个是我自己对官网稍作修改的版本，不放心安装的可自行查看源码:https://github.com/ravenxrz/zotfile**

- [坚果云 PC端](https://www.jianguoyun.com)
- [坚果云 网页端（不需要安装，但是之后配置同步环境需要使用到）](https://www.jianguoyun.com)
- Pdf Expert 6 for Ipad，付费软件，原价68，可上淘宝买，个人觉得IPAD上最好用的PDF软件了

## 3. 同步原理说明

这里贴一张图说一下本方案的同步原理

![](https://ae01.alicdn.com/kf/HTB1n5zaXCf2gK0jSZFPq6xsopXaS.jpg)

1. 从zotero文献库中选取出需要同步到IPAD的文件，若要全部同步就全选即可。将这些文件放到一个指定的目录下；
2. 使用坚果云设置该目录为“同步目录”，这样每当该目录发生更改，坚果云会自动将更改上传至坚果云盘中；
3. 打开PDF EXERT，自动从坚果云盘中拉取最新目录数据；
4. PDF EXPERT同步完成后，即可使用IPAD批注；
5. IPAD批注完成后，PDF EXPERT将会自动上传文件至云盘；
6. PC端的zotero将PDF EXPERT上传的文件拉取至PC本地设置的同步目录；
7. **采用ZotFile插件将同步目录中的文件再还原至文献库的对应位置。**

上述流程看着略显复杂，但是当环境配置好后，整个流程就会非常简单了。需要人工操作的也就3步：

1. 选择需要同步到IPAD的文献库，用ZotFile自动推送到同步目录
2. IPAD批注
3. 用ZotFile将IPAD批注后的文件拉回Zotero文献库

## 4. 开始配置

### 4.1 ZotFile安装

zotero，坚果云，pdf expert这几个软件安装都是一键式的，没什么可说的，简单说下ZotFile如何安装，从官网下载好ZotFile：

![](https://ae01.alicdn.com/kf/HTB1cuLdXAL0gK0jSZFtq6xQCXXap.jpg)

打开zotero，打开工具->插件,得到如下界面：

![](https://ae01.alicdn.com/kf/HTB1lu_dXAH0gK0jSZFNq6xMqXXa4.jpg)

将ZotFile的文件拖入到这个界面即可，然后重启Zotero软件，就完成了安装。

### 4.2 配置ZotFile

首先需要在PC本地建立一个用于同步的文件夹，如图：

![](https://ae01.alicdn.com/kf/HTB1QATbXCf2gK0jSZFPq6xsopXaF.jpg)

接着打开zotero，工具->ZotFile Preferences->Tablet Setting,然后如图设置:

![](https://ae01.alicdn.com/kf/HTB1oAPcXEY1gK0jSZFCq6AwqXXaR.jpg)

**选择自己的同步文件夹。**

### 4.3 设置坚果云同步

注册坚果云账号，并登陆PC软件，然后来到你刚才设置的同步文件夹下，鼠标右键->右键->同步文件夹，之后每当这个文件夹发生变化，坚果云就会同步它。

### 4.4 设置坚果云WebDAV服务

打开坚果云网页端，登陆到后台，

![](https://ae01.alicdn.com/kf/HTB10BPXXqL7gK0jSZFBq6xZZpXa6.jpg)

点击添加应用后，输入pdfexpert，点完成即可。

### 4.5 设置PDF EXPERT 同步服务

打开IPAD上的PDF EXPERT，添加账号后，设置如下：

![](https://ae01.alicdn.com/kf/HTB14RbdXxD1gK0jSZFK763JrVXab.png)

![](https://ae01.alicdn.com/kf/HTB1lZPeXrY1gK0jSZTE760DQVXaL.png)

最后需要打开右上角的同步。

## 5. 开始使用吧

### 5.1  选择需要同步到IPAD的文献库，用ZotFile自动推送到同步目录

![](https://ae01.alicdn.com/kf/HTB1YBzaXpT7gK0jSZFpq6yTkpXad.jpg)

![](https://ae01.alicdn.com/kf/HTB1PN2fXBr0gK0jSZFnq6zRRXXaE.jpg)

### 5.2 打开PDF EXPERT来同步文件

![](https://ae01.alicdn.com/kf/HTB1PKTeXET1gK0jSZFrq6ANCXXaT.jpg)

可以看到它已经被同步下来，可以打开它随便批注点什么。

修改前：

![](https://ae01.alicdn.com/kf/HTB1FyrfXAH0gK0jSZPiq6yvapXak.jpg)

修改后：

![](https://ae01.alicdn.com/kf/HTB1hcjeXuP2gK0jSZFoq6yuIVXaa.jpg)

### 5.3 用ZotFile拉回文献库

![](https://ae01.alicdn.com/kf/HTB1bNYbXpP7gK0jSZFjq6A5aXXa4.jpg)

这里还可以点击Get From Tablet完成拉回，两者的区别在于，Update FileModication Time不在同步文件夹中删除该条目，之后仍可以在IPAD和PC之间同步，点击Get From Tablet在拉回到文献库的同时，删除该条目，之后无法在IPAD和PC之间同步。

最后来看看PC端同步后的效果：

![](https://ae01.alicdn.com/kf/HTB1EqvcXq67gK0jSZFHq6y9jVXaL.jpg)



至于PC端修改后再同步到IPAD操作过程亦然，修改文件后，在选择Send To Tablet（或选择对应类别目录，如我这里就是Tablet同步测试）即可。

## 6. 结语

算是找到个比较完美的方案吧，唯一的不足就是同步过于频繁的话，坚果云每个月的流量容易不足，所以需要控制一下更新的频率。

最后，希望未来生活顺利吧。
