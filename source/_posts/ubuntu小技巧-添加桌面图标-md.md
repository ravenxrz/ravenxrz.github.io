---
title: ubuntu小技巧-添加桌面图标
date: 2019-07-30 13:58:19
categories: Linux
toc: true
tags:
---

说来惭愧，用linux还是比较久了，每次想要添加桌面的图标的时候总是要百度一下，所以这里就记录一下，加深印象。

首先要知道的是，如果想要添加自己的桌面图标，我们需要在**/usr/share/applications**目录下创建一个**xxx.desktop**的文件，**xxx**为你应用名。

比如，我要创建一个微信的desktop，那么执行以下命令：
<!-- more -->
```shell
sudo vim /usr/share/applications/wechat.desktop
```

然后，输入以下内容：

```
// 文件头，必须，告诉系统这是一个desktop文件
[Desktop Entry]
// 图标名
Name=wechat
// 应用类型
Type=Application
// 可执行文件路径，需要替换为你的可执行文件路径
Exec=/home/dnt/下载/electronic-wechat-linux-x64/electronic-wechat
// 图标路径，需要替换为你的图标路径
Icon=/home/dnt/图片/wx.png
// 鼠标经过上面时的提示名称
Comment=wechat
// 打开应用程序时，是否需要打开termial，一般为false
Terminal=false
```

这样就可以用super键时，看到该应用出现在列表中了。但有时候我们像更方便一点，**类似于windows下的一个创建快捷方式到桌面，在ubuntu桌面上页创建一个图标，点击它就可以打开应用程序**。于是可以这样做：
将刚才创建的desktop文件复制一份到桌面上：

```shell
cp /usr/share/applications/wechat.desktop ~/Desktop   # 注意不要用sudo命令，否则后期需要更改所属主
```

然后到桌面找到刚才复制的desktop文件，右键->属性->权限，打钩以下选项：

![](https://ae01.alicdn.com/kf/H435e931f3af14463b434c19abb794e66C.jpg)

最后双击图标就可以了。
