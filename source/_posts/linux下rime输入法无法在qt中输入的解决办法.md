---
title: linux下rime输入法无法在qt中输入的解决办法
abbrlink: 167d6ba3
date: 2020-08-11 19:33:51
categories: 杂文
tags: 输入法
---

## 问题

今天在manjaro下做qt开发时，发现fcitx-rime输入法无法在qtcreator中输入，google了一波，发现应该是差了某个动态库，网上的发行版大多以debian系列为主，apt安装个包就完了，arch系列不是这样解决的。

<!--more-->

## 解决

继续看arch的wiki，搜索fcitx qt等关键词后，发现现在arch都使用的fcitx5了，fcitx5有qt的插件，能够完美解决这些这些问题，具体来说，卸载以前的旧版本fcitx，安装以下包：

![image-20200730113135729](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/5f22402714195aa594ef0f0e.png)

参照arch的wiki，安装这些包后，还需要配置环境变量：

vim ~/.pam_environment 

```shell
INPUT_METHOD  DEFAULT=fcitx5
GTK_IM_MODULE DEFAULT=fcitx5
QT_IM_MODULE  DEFAULT=fcitx5
XMODIFIERS    DEFAULT=@im=fcitx5
```

要开机自启动fcitx，执行：

```shell
cp /usr/share/applications/fcitx5.desktop ~/.config/autostart
```

注销后重新登录，打开fcitx5配置GUI：

![image-20200730113337696](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/5f224cc914195aa594f584e9.png)

rime的配置文件路径也变了，具体路径：

```
~/.local/share/fcitx5/rime
```

最后一个问题，**修改候选框的字体大小**，这个我找了很久：

![image-20200730113538459](https://pic.downk.cc/item/5f22404814195aa594ef2114.png)