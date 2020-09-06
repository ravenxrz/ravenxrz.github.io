---
title: 好玩的数学网站-Desmos
categories: 数学建模
toc: true
abbrlink: 27d14722
date: 2019-01-25 13:58:47
tags:
	- Desmos
---


## 1. 前言

笔者是个比较喜欢尝试新东西的人，最近找到一个好玩又好用的数学绘图网站。可用于数值计算，函数绘图（支持动态函数），几何绘图等等。在数学建模，论文函数绘制，动态展示函数变化等等方面可能会用到。所以，就来介绍一下这个网站。
<!-- more -->
## 2. 使用介绍

首先当然是贴出它的官网了：https://www.desmos.com/

![官网](https://pic3.superbed.cn/item/5cfbad1a451253d178d94d0d.jpg)


点击**Start Graphing**就可以开始了，*下面几个蓝色按钮是它的几个子功能，不用太关注*。

### 2.1 主界面功能介绍

![主界面](https://pic3.superbed.cn/item/5cfbad1e451253d178d94d57.jpg)


主要关注左边的空白区域，因为后续的操作几乎都是围绕着这片区域来完成。除此之外，可通过**分享按钮来下载你绘制好的图像**，**问号按钮有相关的入门教程**（视频教程在youtube上，所以可能需要到科学上网）。

![侧边栏](https://pic.superbed.cn/item/5cfbad24451253d178d94dd0.jpg)


点击侧边栏，可以看到该网站给我们很多模板，我们可以随便点击一个来看看效果，这里我打开了一个四阶多项式拟合：

![拟合](https://pic.superbed.cn/item/5cfbad26451253d178d94e10.jpg)

左边是整个绘制的核心。

首先是输入了散点的`x，y`坐标，然后第二栏中输入要拟合的表达式，在`desmos`中，**~**代表回归拟合的意思，拟合的参数有`a,b,c,d,f`。输入完成后，它会**自动拟合并绘制图图像，然后将各参数显示出来。**

### 2.2 基本功能入门

#### 2.2.1 基本函数绘制与计算

新建一个绘图板。在第一栏中输入$f(x) = sin(x)$。可以得到下图:

![sin函数](https://pic2.superbed.cn/item/5cfbad28451253d178d94e4a.jpg)

如何获取函数在某点的值呢？有三种方式:

- 鼠标点击图形获取，这个最简单，但是也最不精确。

- 通过子栏输入获取：

![子栏获取](https://pic.superbed.cn/item/5cfbad2b451253d178d94e8a.jpg)

- 转换为表格获取，点击坐标面板的设置按钮，点击**convert to table**。得到下图:

![转换表格获取](https://pic3.superbed.cn/item/5cfbad2c451253d178d94ec5.jpg)


  可在x轴中的任意位置插入`x`的值，它会自动给出结果。如我要知道$x=pi，f(x)$等于多少。

![pi值对应f(x)](https://pic.superbed.cn/item/5cfbad2d451253d178d94f03.jpg)

另外，也可以增加列（增加其他函数），得到相同x值下的不同函数值:

![增加列](https://pic3.superbed.cn/item/5cfbad2f451253d178d94f3b.jpg)


在得到不同的函数值的同时，也得到了增加的函数的散点图。如何修改绘图样式呢？长按$cos(x)$左边绿色的图标，得到下图：

![颜色更改](https://pic.superbed.cn/item/5cfbad32451253d178d94f7e.jpg)


这里可以修改样式和颜色，具体的效果可以自己的尝试，我这里就把线也加上吧。

![更改样式](https://pic.superbed.cn/item/5cfbad34451253d178d94fb9.jpg)

**额外补充一点，该网站支持的函数集合可通过下图查看**：

![支持函数](https://pic1.superbed.cn/item/5cfbad37451253d178d94ffe.jpg)



#### 2.2.2 文件夹

当要绘制多个图形时，我们需要一种方式来管理多个图。好在`desmos`提供了文件夹管理方式，我们可以通过文件夹来一键隐藏/显示多个函数。使用方式如下:

1. 点击左上角的+号，新建一个文件夹
2. 书写任意表达式，通过拖拽的方式把书写的表达式放进文件夹中。
3. 点击文件夹坐标的图标就可以实现多个图形的同时隐藏和显示了

#### 2.2.3 列表

在`desmos`中，可定义列表来表达多个图形，定义方式如下:

`a = [1,2,3]`

也可以通过步长来定义

`a = [1...10]` 默认步长1

或者`a=[0,5...100]`推导出步长为5

然后可以像下图这样使用：

![列表使用1](https://pic1.superbed.cn/item/5cfbad3a451253d178d95058.jpg)

或者也可以同时画多个点：

![列表使用2](https://pic.superbed.cn/item/5cfbad3d451253d178d95096.jpg)



#### 2.2.4 滑动变量（slider)

在`desmos`中可定义滑动变量来绘制动态图像。

定义一个滑动变量很简单，只要记住一个规则：不要使用常用数学符号,`x,y,t,r`等。然后再书写表达式desmos会自动提示是否生成滑动变量。如：

![slider变量](https://pic.superbed.cn/item/5cfbad43451253d178d95110.jpg)

通过滑动滑杆或者点击播放，右侧的绘图区会绘制出动态图。（提示：点击播放后右侧有调速滑杆）。

下面展示一个高级点的：

![动态切线](https://raw.githubusercontent.com/ravenxrz/BlogPic/master/img/007LxXtnly1g2ofycr6udg311s0jjk3z.gif)

是不是感觉很好玩勒。

#### 2.2.5 参数方程与极坐标

`desmos`也是支持参数方程，例如我们利用参数方程绘制一个圆形:

![绘制圆](https://pic.superbed.cn/item/5cfbad45451253d178d95147.jpg)


再看看极坐标

在`desmos`中，极坐标以符号r和$\theta$表示，画个四叶草？？？

![四叶草](https://pic1.superbed.cn/item/5cfbad4b451253d178d951a6.jpg)

相当简单是吧。

#### 2.2.6 不等式

讲解不等式之前，先说说如何在`desmos`中限制定义域或者值域。通过在大括号{}之间添加限制条件即可。如下:

![范围限制](https://pic.superbed.cn/item/5cfbad4d451253d178d951ef.jpg)

OK,知道了这一点，咱们再来看看使用不等式的使用方式。假设我要绘制一个这样的图形:

$$
\\left\\{
\\begin{array}{c}
x \\lt 2	\\\\
0 \\lt y \\lt x -1
\\end{array}
\\right.
$$

那么这样写就画出来啦。

![不等式绘制](https://pic.superbed.cn/item/5cfbad51451253d178d9523c.jpg)

(ps:高中的时候有一类求解极大极小值就是需要画图来做吧)

#### 2.2.7 积分

`desmos`支持积分计算，也支持变上（下）限函数。通过输入`int`自动添加积分符号，如：

![积分计算](https://pic.superbed.cn/item/5cfbad58451253d178d952cf.jpg)

## 3. 结语

嗯，`desmos`的基础入门就这样了吧，还有很多高级用法可以阅读它提供的模板和文档，当然也可观看`desmos`的官方教程。笔者觉得蛮好玩的就学习了一下，希望能够对你有所帮助。

