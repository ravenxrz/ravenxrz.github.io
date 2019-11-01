---
title: 从零开始搭建属于自己的博客系列教程四-去掉烦人的index-php后缀与常用插件介绍
categories: 博客搭建
toc: true
abbrlink: e8deae94
date: 2019-05-04 13:59:36
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

## 2. 去掉烦人的index.php后缀

不知道你有没有注意，你的博客url中总是会有index.php后缀，本篇就教你如何去掉它。

进入typecho后台管理，后台设置--永久链接设置--启用。如果提示你无法启动，那么再往下看。

更改nginx配置文件，具体操作如下：

```
vi /usr/local/nginx/conf/nginx.conf
```

找到server字段，然后再其中添加：

```
location / {

    if (!-e $request_filename) {
        rewrite  ^/(.*)$  /index.php/$1  last;
        break;
    }
}
```

![](https://pic.superbed.cn/item/5cfbace9451253d178d94620.png)

即可。
**最后再去永久链接设置里启用就好了。**

## 3. 常用插件介绍

- [CodeHighlighter](https://github.com/Copterfly/CodeHighlighter-for-Typecho): 代码高亮插件：
- [BaiduSlug](https://www.bijiv.com/usr/uploads/2018/04/2781131976.zip): 给你的博客url改成对应标题的英文名，这么说不好理解，直接上图吧
- ![](https://pic2.superbed.cn/item/5cfbaceb451253d178d94655.png)
- [Sticky](https://github.com/hitop/typechoSticky): 文章置顶
- [ShortLinks](https://github.com/benzBrake/ShortLinks)：博文外链转内链，避免权重流失。

下面两个是做SEO的，方便百度收录，我没用，所以自己研究吧。

- BaiduLinkSeo
- BaiduSubmit

## 参考

- [Typecho去掉index.php](https://seonoco.com/typecho-remove-index-php)
- [Typecho自动翻译slug插件](https://www.bijiv.com/t/46)

