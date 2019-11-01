---
title: Win10-TensorBoard使用方式及踩坑日记
categories: 深度学习
toc: true
abbrlink: b6686206
date: 2019-04-23 13:58:21
tags:
---


之前在ubuntu上使用tensorboard还挺顺利的，没想到最近在win10却踩了很久的坑，这里记录一下。

<!-- more -->

我的`python`环境如下:

- python 3.5.6
- tensorflow 1.10.0



这里先说一下如何使用`TensorBoard`。基本上分两步：
- 第一步将你需要保存的计算图输出到日志文件中
-  第二步使用`TensorBoard`生成可视化图形

**Step1：将计算图输出到日志中，这里以两个张量的加法作为例子。**

```
"""
TensorBoard日志输出
"""
import tensorflow as tf

# 定义了一个简单的计算图，实现向量加法的操作
input1 = tf.constant([1.0,2.0,3.0],shape=[1,3],name='input1')
input2 = tf.Variable(tf.random_normal([1,3]),name='input2')
output = tf.add_n([input1,input2],name='add')

# 生成一个写日志的writer。并将当前的TensorFLow计算图写入到日志中。
writer = tf.summary.FileWriter('./log',tf.get_default_graph())
writer.close()
```

**Step2: 使用TensorBoard程序生成可可视化图形**

在`step1`中，我使用的路径是相对路径`./log`，为便于后续说明，先贴上它的绝对路径：`D:\Projects\PythonProjects\TensorFlowLearning\google_tensorflow_practise\chapter9\log`

。之后找到`TensorBoard`程序所在地（可以使用**Everything**软件来搜索），如下图:

![](https://pic.superbed.cn/item/5cfbb5ea451253d178d9d42b.png)

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbb790451253d178d9ec24.png)
然后在这里打开你的打开`power shell`或者`cmd`或者`cmder`窗口（笔者这里使用的是`cmder`，三个都可以）。
`shift` + 鼠标右键可选择`power shell`。

执行`tensorboard.exe --logdir=D:\Projects\PythonProjects\TensorFlowLearning\google_tensorflow_practise\chapter9\log`，将`logdir`后面的路径替换为你的绝对路径即可。

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbb7a0451253d178d9ed2b.png)
**注意上图是失败的**，如果你能看到类似下图，**含有个网址的结果那就是成功的**，之后把地址贴到浏览器就好了，然后就可以溜啦，不要浪费时间在阅读后面的文章了。

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbb7a9451253d178d9edd8.png)
如果你和我一样不幸，出现了类似`Unable to create process using .....`的结果，那么这里给出几种解决方案。

**solution 1: 我的解决方案**

首先确定你用得`python`解释器是多少，如果你用的`pycharm`的话，可如下图来确定：

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbb7a9451253d178d9edd8.png)
可以看到，我的解释器是`3.5`，确定了这个以后就按照它给的路径去文件管理器下找到`python.exe`程序:

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbb7b2451253d178d9ee5b.png)
另外找到之前`tensorboard.exe`程序所在地,但是这次我们关注的不是`tensorboard.exe`，而是`tensorboard-script.py`文件：

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbb7c4451253d178d9ef65.png)
ok，只要确定了这两个，打开`power shell`或者其他命令窗口。按照

` python.exe路径 tensorboard-script.py路径 --logdir=生成的日志目录路径`的结构去执行命令（**将每个程序拖入到命令窗口就好了，不用手敲的**）

现在，你是否能得到下图结果呢？

![在这里插入图片描述](https://pic2.superbed.cn/item/5cfbb7d2451253d178d9f05e.png)
**solution 2: google出来的解决方案**

执行以下命令:

`python.exe -m tensorflow.tensorboard --logdir=日志路径`

注意`python.exe`依然是要对应你的解释器的。

不过笔者用这个命令的时候出现了`No module named tensorflow.tensorboard`的错误。

**solution 3: github 上的解决方案**

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbb7da451253d178d9f0e1.png)
这里贴出原文网址:https://github.com/tensorflow/tensorflow/issues/10017
如果上面2种方式你都没解决这个问题，可以尝试使用这个的方法。

```
backup the tensorboard.exe
open tensorboard.exe with 010 Editor,I use this
search "python.exe"
change all the ascil code between #! and python.exe to 20(whitespace)
change the quotation marks after python.exe to whitespace
In my computer:
before:
#!"c:\program files\python35\python.exe"
after:
#!　　　　　　　　　　　　　　　python.exe
```



如果你能成功的话，把生成的网址贴到浏览器就好啦。可以得到类似下图的结果：

![在这里插入图片描述](https://pic.superbed.cn/item/5cfbb7ef451253d178d9f258.png)


嗯，好了。希望你能够解决你的问题咯。

