---
title: 从零开始搭建属于自己的博客系列教程一-博客系统搭建
date: 2019-05-04 13:58:29
categories: 博客搭建
toc: true
tags:
---

[Meting]
[Music server="netease" id="4937357" type="song"/]
[/Meting]

## 1.  写在最前面
<!-- more -->
> 折腾了整整一个五一，总算是完成了自己的博客系统。为了给想要搭建自己的博客，但是又和我一样是技术小白的人铺点路，所以准备开个博客搭建的系列教程，避免后来人像我一样走了一大堆弯路。

先贴上自己的博客网站：https://www.ravenxrz.ink   

系列教程目录：

- [从零开始搭建属于自己的博客系列教程零 -- 绪论](https://www.ravenxrz.ink/archives/ck27a6coa001t08vm7dom4ddk/)
- [从零开始搭建属于自己的博客系列教程一 -- 博客系统搭建](https://www.ravenxrz.ink/archives/ck27a6co2001d08vm37fp2jrz/)
- [从零开始搭建属于自己的博客系列教程二 -- 域名绑定](https://www.ravenxrz.ink/archives/ck27a6co5001i08vmexcp0gm3/)
- [从零开始搭建属于自己的博客系列教程三 -- 主题选择与CommentToMail插件介绍](https://www.ravenxrz.ink/archives/ck27a6cp5003u08vm39sqe1zg/)
- [从零开始搭建属于自己的博客系列教程四 -- 去掉烦人的index.php后缀与常用插件介绍](https://www.ravenxrz.ink/archives/ck27a6co9001r08vm7zd936xx/)
- [从零开始搭建属于自己的博客系列教程五 -- 图床选择与博客迁移](https://www.ravenxrz.ink/archives/ck27a6co7001l08vm695m3jwl/)
- [从零开始搭建属于自己的博客系列教程六 -- 博客系统备份](https://www.ravenxrz.ink/archives/ck27a6co8001o08vme47fc50a/)
- [从零开始搭建属于自己的博客系列教程七 -- 开启全站HTTPS](https://www.ravenxrz.ink/archives/ck27a6co4001g08vm38779qtt/)

## 2.  正式搭建

### 2.1  系统环境准备

- 域名（可选，但是最好买上，可参考本系列教程二购买）
- 国内或国外服务器（国内需要备案，但是备案有备案的好处，后续的图床CDN加速就需要国内备案服务器）
- XShell6

### 2.2  服务器购买

**这里推荐使用vultr，国外服务器，无需备案，vultr按小时计费，且可无限次更换ip。**

step1: 点击[我的vultr推广](https://www.vultr.com/?ref=7744129) （希望小伙伴支持一下啦~），去[vultr官网](https://my.vultr.com)注册一个账号，并充值，支持支付宝、微信，最低充值10美元。

![vultr充值](https://pic.superbed.cn/item/5cfbacd2451253d178d942d3.png)

step2: 购买一个vps，尽量选择离大陆近的，这里以日本为例，按图示配置（如果有更便宜选项，也可以选购）

![](https://pic.superbed.cn/item/5cfbacd3451253d178d9431e.png)

![](https://pic.superbed.cn/item/5cfbacd5451253d178d9435e.png)

step3: 稍等片刻后，即可以在Servers选项栏中看到你购买的vps，购买后还未完，我们需要测试它是否能够连接上，复制你的ip地址（如图）。

![](https://pic.superbed.cn/item/5cfbacd7451253d178d943a5.png)

进入这个端口扫描网站：http://tool.chinaz.com/port/
输入你的ip，端口设置为22，看22端口是否开通。
![](https://pic.superbed.cn/item/5cfbacd9451253d178d943e5.png)

如果未开通，将这个服务器destory掉，然后重新购买，直到22端口开通为止。


### 2.3  XShell连接，开始搭建环境

这里给一个破解的Xshell链接：https://pan.baidu.com/s/1OlBSYFjdk9oDXot_V9VZUg

一路下一步，安装后打开主程序，文件->新建：

![](https://pic.superbed.cn/item/5cfbacda451253d178d94422.png)

![](https://pic3.superbed.cn/item/5cfbacdc451253d178d94462.png)

用户名和密码如下图，记得替换为你的服务器：

![](https://pic.superbed.cn/item/5cfbace2451253d178d944f3.png)

接下来XShell会提示你接受个证书还是啥的，点击接受并保存就是了。然后你就会来到服务器界面。

  ![](https://pic.superbed.cn/item/5cfbace4451253d178d94532.png)

到这里，已经成功连接到你的服务器了。

### 2.4 安装mysql+nginx+php环境

自己分别安装太过麻烦，还好一键傻瓜式安装包- [lnmp](https://lnmp.org/)

到笔者安装的时候是1.5版本，可能你看到这篇文章时已经更新，所以自己去找最新版本吧，当然也可以使用1.5版本。

在xshell中执行以下命令：

```
wget http://soft.vpser.net/lnmp/lnmp1.5.tar.gz
tar xvf lnmp1.5.tar.gz
cd lnmp1.5
./install.sh
```

![](http://upload-images.jianshu.io/upload_images/606686-e97fffd23200f958.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图来源：https://blog.csdn.net/Boxuerixin/article/details/79106587?tdsourcetag=s_pctim_aiomsg

**回车下一步，会提示你输入MySQL的密码，这个自己决定，但是要记住，后续需要使用。**

接下来就是一路回车，然后进入漫长等待，大约需要半小时吧。直到看到下图：

![](https://pic3.superbed.cn/item/5cfbace5451253d178d94567.png)

接着在浏览器中输入, http://yourip （替换yourip为你的ip），可见下图则说明安装成功：

![](http://upload-images.jianshu.io/upload_images/606686-5b44be3c96513f34.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图来源：https://blog.csdn.net/Boxuerixin/article/details/79106587?tdsourcetag=s_pctim_aiomsg

### 2.5  安装Typecho

同样的，笔者安装时`Typecho`版本为`1.1`，可自行到官网:http://typecho.org/，上选择最新版本。

进如`/home/wwwroot`目录 -- `cd /home/wwwroot`

执行以下代码:

```
 wget http://typecho.org/downloads/1.1-17.10.30-release.tar.gz
 tar xvf 1.1-17.10.30-release.tar.gz 
 cd /home/wwwroot
 mv default/ old
 mv build/ default
 chown -R 755 /home/wwwroot
 chown -R www:www /home/wwwroot/
```

### 2.6  创建typecho数据库

执行以下命令：

`mysql -u root -p`

输入密码，接着会进入mysql管理窗口。

执行以下命令：

```
create database typecho;
exit;
```

### 2.7 完成typecho安装

在浏览器中输入,http://yourip（替换为你的ip），会进入typecho的安装引导界面。

以下图片来源：https://ask.qcloudimg.com/draft/1008453/prk237k4v3.png?imageView2/2/w/1620

![img](https://ask.qcloudimg.com/draft/1008453/prk237k4v3.png?imageView2/2/w/1620)

点击下一步，进入:

![](https://upload-images.jianshu.io/upload_images/3778244-94a8170cbabc82d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

```
端口 3306
数据库名 root
密码： 你的密码
其余不用更改
```

然后下一步，得到类似这种界面：

![](https://pic.superbed.cn/item/5cfbace7451253d178d945a5.png)

最后就可以来看你的博客站点啦：

![](https://ask.qcloudimg.com/draft/1008453/ekmy1frrd9.png?imageView2/2/w/1620)

## 3. 可能遇到的问题

Q:Typecho前台链接或者后台登录出现404?

编程enable-php.conf文件：

`vi /usr/local/nginx/conf/enable-php.conf`

然后改为:

```
location ~ [^/]\.php(/|$)
{
        #try_files $uri =404;
        fastcgi_pass  unix:/tmp/php-cgi.sock;
        fastcgi_index index.php;
        include fastcgi.conf;
        include pathinfo.conf;
}
```

如下图

![](https://pic.superbed.cn/item/5cfbace8451253d178d945e9.png)

最后重启nginx服务即可:

```
/etc/init.d/nginx restart
```



## 参考：

- [使用 Vultr 搭建 WordPress 博客教程（超详细）](https://blog.csdn.net/Boxuerixin/article/details/79106587?tdsourcetag=s_pctim_aiomsg)
- [Xshell6 中文不限时版下载(免密匙)（笔记)](https://blog.csdn.net/qq_31362105/article/details/80706750)
- [如何搭建 Typecho 博客](https://cloud.tencent.com/developer/article/1356132)
- [Typecho前台链接或者后台登录出现404的解决方法](https://boke112.com/5112.html)

