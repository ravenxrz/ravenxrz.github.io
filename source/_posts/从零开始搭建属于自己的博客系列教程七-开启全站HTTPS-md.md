---
title: 从零开始搭建属于自己的博客系列教程七-开启全站HTTPS
date: 2019-05-04 16:58:30
categories: 博客搭建
toc: true
tags:
---

系列教程目录：

- [从零开始搭建属于自己的博客系列教程零 -- 绪论](https://www.ravenxrz.ink/archives/ck27kp480001u4gvmay908xku/)
- [从零开始搭建属于自己的博客系列教程一 -- 博客系统搭建](https://www.ravenxrz.ink/archives/ck27kp47t001e4gvm3ztsc97x/)
- [从零开始搭建属于自己的博客系列教程二 -- 域名绑定](https://www.ravenxrz.ink/archives/ck27kp47u001h4gvm669q1u7p/)
- [从零开始搭建属于自己的博客系列教程三 -- 主题选择与CommentToMail插件介绍](https://www.ravenxrz.ink/archives/ck27kp48v003w4gvmdcd2d0fm/)
- [从零开始搭建属于自己的博客系列教程四 -- 去掉烦人的index.php后缀与常用插件介绍](https://www.ravenxrz.ink/archives/ck27kp47y001p4gvm3zsucst2/)
- [从零开始搭建属于自己的博客系列教程五 -- 图床选择与博客迁移](https://www.ravenxrz.ink/archives/ck27kp47w001j4gvm1ltvcpbg/)
- [从零开始搭建属于自己的博客系列教程六 -- 博客系统备份](https://www.ravenxrz.ink/archives/ck27kp47z001s4gvmbldsfb6h/)
- [从零开始搭建属于自己的博客系列教程七 -- 开启全站HTTPS](https://www.ravenxrz.ink/archives/ck27kp47x001m4gvmfyii1whr/)
<!-- more -->
## 1. 前言

> 昨天无聊把博客图床换了，今天继续来优化下博客。发现以前我走的url都是http，所以在浏览器上会显示“不安全”的字眼，今天就来把http转为https。

小科普：https是在HTTP的基础上加入了SSL/TLS协议，依靠SSL证书来验证服务器的身份，并为客户端和服务器端之间建立“SSL加密通道”，确保用户数据在传输过程中处于加密状态，同时防止服务器被钓鱼网站假冒。

## 2. 证书申请

对于个人博客来说，不需要太高等级的安全保证，所以只用申请DV域名型https证书即可。可在腾讯云或者阿里云上免费申请，保质期一年，一年后重新申请即可。下面以腾讯云为例讲讲步骤：
打开[腾讯云SLL控制管理](https://console.cloud.tencent.com/ssl)
![](https://pic3.superbed.cn/item/5cfbb5a7451253d178d9cbb7.png)

确认证书类型：

![](https://pic1.superbed.cn/item/5cfbb5a8451253d178d9cbf0.png)

按照提示一步一步填写信息

![](https://pic2.superbed.cn/item/5cfbb5aa451253d178d9cc32.png)

选择手动DNS验证

![](https://pic3.superbed.cn/item/5cfbb5ab451253d178d9cc7c.png)

点击查看证书详情，跳转到证书信息页面

![](https://pic.superbed.cn/item/5cfbb5ad451253d178d9ccbf.png)

然后到设置一个域名解析就好了，解析记录所需要填写的字段就是现在的证书详情上的字段。（因为我已经申请过了，所以现在看不到这些字段，就不好截图了）。

![](https://pic.superbed.cn/item/5cfbb5ae451253d178d9ccec.png)

等待几分钟，等待CA机构扫描成功后，腾讯云会发送一封邮件给你提醒配置成功，这个时候就可以下载证书。

![](https://pic.superbed.cn/item/5cfbb5af451253d178d9cd1e.png)

下载出来会有四个文件夹，我用的是Nginx，所以只用关注Nginx即可。

## 3. 配置Nginx服务器

step1:将解压出来的Nginx文件夹中的两个文件防止到服务器的`/usr/local/nginx/conf` 目录下。

step2:编辑 Nginx 根目录下的 conf/nginx.conf 文件。修改内容如下：

```
server {
     listen 443;
     server_name www.domain.com; #填写绑定证书的域名
     ssl on;
     ssl_certificate 1_www.domain.com_bundle.crt;#证书文件名称
     ssl_certificate_key 2_www.domain.com.key;#私钥文件名称
     ssl_session_timeout 5m;
     ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #请按照这个协议配置
     ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;#请按照这个套件配置
     ssl_prefer_server_ciphers on;
     location / {
         root /var/www/www.domain.com; #网站主页路径。此路径仅供参考，具体请您按照实际目录操作。
         index  index.html index.htm;
     }
 }
```

注意把相关项对应到你的相关项，如域名，证书文件名等。。

step3: 重启 Nginx，现在你应该可以通过https://域名访问你的博客了。

step4: TYPECHO 配置

登录Typecho后台 \-> 设置 \-> 基本设置 \-> 站点地址改成 https://你的域名

编辑Typecho站点根目录下的文件config.inc.php加入下面一行配置，否则网站后台还是会调用HTTP资源。

```
/** 开启HTTPS */
define('__TYPECHO_SECURE__',true);
```

## 4. 重定向Http到Https

仍然是打开刚才的配置文件nginx.conf,填写以下内容：

```
server {
    listen 443;
    server_name www.domain.com; #填写绑定证书的域名
    ssl on;
    root /var/www/www.domain.com; #网站主页路径。此路径仅供参考，具体请您按照实际目录操作。
    index index.html index.htm;   #上面配置的文件夹里面的index.html
    ssl_certificate  1_www.domain.com_bundle.crt; #证书文件名称
    ssl_certificate_key 2_www.domain.com.key; #私钥文件名称
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
       index index.html index.htm;
    }
}
server {
    listen 80;
    server_name www.domain.com; #填写绑定证书的域名
    rewrite ^(.*)$ https://$host$1 permanent; #把http的域名请求转成https
}
```

注意把相关项对应到你的相关项，如域名，证书文件名等。

至此就完成了http转换到https的配置。

## 参考

[博客全站启用HTTPS](https://buxuhunao.com/article/blog-to-https.html#directory00916761880772338811)

[Nginx 服务器证书安装](https://cloud.tencent.com/document/product/400/35244)


  [1]: https://ravenxrz.ink/2019/06/07/start-from-scratch-to-build-your-own-blog-series-of-tutorials-6-open-the-whole-site-https.html

