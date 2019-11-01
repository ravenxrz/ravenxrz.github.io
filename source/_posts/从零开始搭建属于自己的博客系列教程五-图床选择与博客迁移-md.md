---
title: 从零开始搭建属于自己的博客系列教程五-图床选择与博客迁移
categories: 博客搭建
toc: true
abbrlink: 3cc5aa32
date: 2019-05-04 14:52:33
tags:
---


## 1.  写在最前面

> 折腾了整整一个五一，总算是完成了自己的博客系统。为了给想要搭建自己的博客，但是又和我一样是技术小白的人铺点路，所以准备开个博客搭建的系列教程，避免后来人像我一样走了一大堆弯路。

先贴上自己的博客网站：https://www.ravenxrz.ink   
<!-- more -->
系列教程目录：

- [从零开始搭建属于自己的博客系列教程零 -- 绪论](https://www.ravenxrz.ink/archives/ck27kp480001u4gvmay908xku/)
- [从零开始搭建属于自己的博客系列教程一 -- 博客系统搭建](https://www.ravenxrz.ink/archives/ck27kp47t001e4gvm3ztsc97x/)
- [从零开始搭建属于自己的博客系列教程二 -- 域名绑定](https://www.ravenxrz.ink/archives/ck27kp47u001h4gvm669q1u7p/)
- [从零开始搭建属于自己的博客系列教程三 -- 主题选择与CommentToMail插件介绍](https://www.ravenxrz.ink/archives/ck27kp48v003w4gvmdcd2d0fm/)
- [从零开始搭建属于自己的博客系列教程四 -- 去掉烦人的index.php后缀与常用插件介绍](https://www.ravenxrz.ink/archives/ck27kp47y001p4gvm3zsucst2/)
- [从零开始搭建属于自己的博客系列教程五 -- 图床选择与博客迁移](https://www.ravenxrz.ink/archives/ck27kp47w001j4gvm1ltvcpbg/)
- [从零开始搭建属于自己的博客系列教程六 -- 博客系统备份](https://www.ravenxrz.ink/archives/ck27kp47z001s4gvmbldsfb6h/)
- [从零开始搭建属于自己的博客系列教程七 -- 开启全站HTTPS](https://www.ravenxrz.ink/archives/ck27kp47x001m4gvmfyii1whr/)

## 2. 图床选择

写博客避免不了上传图片，以前在csdn和简书上写文章，图片是自动保存在他们的服务器的，现在你要自己搭建博客了，那么图床就是个问题了。从百度来看，常用图床有：

- 七牛图床:速度快，稳定，但有空间、流量限制。如果你使用的国内备案了的服务器，那么推荐你使用这个，因为我是使用的国外服务器，所以虽然很想用，但是它的CDN加速，必须要要国内的，so..我就放弃了。
- 微博图床: 无限流量、无限空间、稳定自带CDN加速。我本来就准备选择这个了，并且同上床了好多张图片，**然而上个月末(2019/04末)，微博已关闭这个服务**，具体来说是防止盗链（因为微博并没有这个功能，只是广大好友另辟蹊径使用了所谓的“微博图床”）,所以没办法，无法使用。
- Github图床：免费、无限流量、有限空间，速度慢但稳定。**最终我选择的是Github图床**。

## 3 图床上传工具

强烈推荐[PicGo](https://github.com/Molunerfinn/PicGo),三平台支持下载，支持常用主流图床。这里以github图床为例，进行讲解：

### 3.1 创建github博客图片仓库

![](https://pic.superbed.cn/item/5cfbb5ba451253d178d9ce40.png)

如图配置

![](https://pic.superbed.cn/item/5cfbb5bc451253d178d9ce7a.png)

这里我的仓库名为`BlogPic`。

### 3.2 创建github token

进入Settings->Developer settings:

![](https://pic.superbed.cn/item/5cfbb5bd451253d178d9cebc.png)

会让你验证密码，输入即可。

![](https://pic.superbed.cn/item/5cfbb5bf451253d178d9cf05.png)

token描述随意，**记得勾选下面repo选项**，然后确定生成。

最后会有类似下图的token：

![](https://pic.superbed.cn/item/5cfbb5c0451253d178d9cf60.png)

上图来源：https://blog.csdn.net/yefcion/article/details/88412025

**复制下来，注意这个链接仅会显示一次。**

### 3.3 安装PicGo并进行配置

安装PicGo就不细说了，上github下载安装包，下一步到底就ok。

现在来说说如何配置Github图床：

![](https://pic2.superbed.cn/item/5cfbb5c1451253d178d9cf95.png)

最后的自定义域名`https://raw.githubusercontent.com/[仓库名]/master`。如我的域名为:`https://raw.githubusercontent.com/ravenxrz/BlogPic/master`

然后就可以欢快的上传图片了。有三种方式上传

- 快捷键上传
- gui界面选择文件（支持多文件）上传
- 支持剪贴板上传（点击gui或者直接用快捷键上传），这个功能真心赞。

## 4.  博客迁移

为了防止出现类似微博的防盗链情况，建议把所有曾经的博文图片全部重新上传。但是一张张的重新上传实在太麻烦，好在picgo有插件，支持转移。

插件官方：https://github.com/PicGo/picgo-plugin-pic-migrater

虽然有GUI版本的插件，但是我用过一个晚上，第二天准备再用时就报错了，所以这里推荐使用命令行版本的插件。当然了picgo也得重新安装CIL版本的。

报错信息如下：

![](https://pic2.superbed.cn/item/5cfbb5c3451253d178d9cfc3.png)

### 4.1 重新安装picgo CIL版本

先安装[Node.js](https://nodejs.org/en/)，然后打开命令行，执行

```
npm install picgo -g
```

接着执行

```
picgo install pic-migrater
picgo set plugin pic-migrater
```

会出现一个配置界面：参考https://github.com/PicGo/picgo-plugin-pic-migrater/blob/master/README_CN.md

然后还需要配置config.json文件，文件在`C:\Users\用户名\.picgo目录下`，设定图床。这里给个样例：

```
{
  "picBed": {
    "current": "github",
    "proxy": "",
    "github": {
      "repo": "ravenxrz/BlogPic",		# 你的仓库
      "token": "xxx",					# token
      "path": "img/",					# 保存路径
      "customUrl": "https://raw.githubusercontent.com/ravenxrz/BlogPic/master",	# url
      "branch": "master"				#分支
    }
  },
  "picgoPlugins": {
    "picgo-plugin-pic-migrater": true
  },
  "picgo-plugin-pic-migrater": {
    "newFilePrefix": "_new",
    "include": "csdnimg",
    "exclude": ""
  }
}
```

安装完成后，即可进行迁移，例如有一个md文档叫做test.md

```
Examples:
  # migrate file or files
  $ picgo migrate ./test.md 
```

如果想批量迁移（不建议使用，测试发现会有会出现文章混乱的情况），建立一个文件夹入test，把md文件丢进去，然后执行：

```
  # migrate markdown files in folder
  $ picgo migrate ./test/
```

然后就ok了。

### 4.2 额外的一些事情

1. 提醒，迁移简书的图片时，需要将链接的后半段给去掉，如一个链接

(https://upload-images.jianshu.io/upload_images/3213538-73ba9a133ff83204.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么我们需要将从?起到末尾全部去掉。如何批量去掉呢。可以使用notepad++或者sublime这种支持正则表达式的编辑器，然后ctrl+H。

查找: \\?.*\\)

替换为:\\)

2. 同理，csdn，也可以用正则表达式，替换方式相同

## 参考

- [PicGo + GitHub 搭建个人图床工具](https://blog.csdn.net/yefcion/article/details/88412025)

