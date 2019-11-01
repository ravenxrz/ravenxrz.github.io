---
title: 用Android手机作PC的麦克风和摄像头
categories: Android
toc: true
abbrlink: fc472545
date: 2019-07-07 13:59:03
tags:
---

## 1.麦克风
要使用Android手机作PC的麦克风，需要在PC和手机上都下载[WO Mic](http://www.wirelessorange.com/womic).
- [Android APK](http://www.wirelessorange.com/womic/softwares/app-release.apk)
- [Windows WO Mic client program](http://www.wirelessorange.com/womic/softwares/wo_mic_client_setup.exe)
- [Windows WO Mic virtual device driver](http://www.wirelessorange.com/womic/softwares/wo_mic_driver_signed.exe)
<!-- more -->
### 1.1 选择传输模式
注意必须安装虚拟驱动，才能使用。

使用WO Mic，PC和Android之间通信的方式有一下3种:
- USB模式
- WIFI模式
- Bluetooth模式
看个人方便自行选择。

### 1.2 开启手机服务端
1. 打开Wo Mic App
2. 在Settings中选择所要使用的传输模式。
3. 回到主界面点击start即可。

*注意可能需要给予一些权限*

### 1.3 在PC端连接手机
1. 打开WO Mic Client
2. 选择相应的传输模式，如果选择的是wifi模式，需要填写IP地址
3. 连接即可。

**Linux版本可参考:http://www.wirelessorange.com/womic/wo_mic_linux.html#div_operations**

## 2.摄像头
同样也需要使用一款软件--[DroidCam](http://www.dev47apps.com)
支持Windows和Linux平台。
![](https://ae01.alicdn.com/kf/HTB19AxgXAT2gK0jSZFkq6AIQFXaM.jpg)

### 2.1 安装Android APP
APP地址: https://pan.baidu.com/s/1SmJfQxO0ZVo5yjeB4gL0Xw#list/path=%2F
 密码：7fp3

### 2.2 PC端安装对应Client软件
由于Win只用安装一个软件，比较简单这里就不说具体怎么安装了。现在说说linux如何安装:
首先贴出官网的教程地址:http://www.dev47apps.com/droidcam/linuxx

按顺序执行以下命令:
```
cd /tmp/
sudo apt-get install linux-headers-`uname -r`
tar xjf droidcam-${bits}bit.tar.bz2
cd droidcam-${bits}bit/
sudo ./install
```

### 2.3 测试

在命令行中执行`droidcam`,打开droidcam程序。

为测试是否有效，需要下载一个视频播放软件，这里推荐使用VLC Player，安装命令:
```
sudo apt install vlc
```
打开vlc播放器后，选择Media->Open Capture Device,点击Play即可。


