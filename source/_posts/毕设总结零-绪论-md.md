---
title: 毕设总结零-绪论
date: 2019-07-17 12:03:00
categories: 深度学习
toc: true
tags: 图像去雾
---

i> 本科毕设题目为《基于深度学习的图像去雾方法研究与实现》

毕设总结系列：
- [毕设总结零--绪论](https://www.ravenxrz.ink/archives/ck27kp48k00344gvmcq9b8i5j/)
- [毕设总结一--传统去雾理论](https://www.ravenxrz.ink/archives/ck27kp48e002s4gvm7s5s82xx/)
- [毕设总结二--论文复现工作1--暗通道去雾(DCP)](https://www.ravenxrz.ink/archives/ck27kp48i002z4gvm5r0m07ge/)
- [毕设总结三--论文复现工作2--MSCNN去雾](https://www.ravenxrz.ink/archives/ck27kp48g002u4gvmb1uw8rmj/)
- [毕设总结四--最终去雾方案的确定与实现--DeBlurGanToDehaze](https://www.ravenxrz.ink/archives/ck27kp48m00384gvm2tlq4efw/)
- [毕设总结五--杂项](https://www.ravenxrz.ink/archives/ck27kp48j00314gvm1uobgc59/)
<!-- more -->

## 1. 前言

**2019/6/11**日，毕设答辩结束，意味着我本科四年的学习生涯也画上了句号。从1月初拿到毕设课题到今日已过5月，收获良多，也算是为未来的研究生生涯提前试水吧（虽然研究生方向不做深度学习，不过算是了解了做一个课题需要做的大体流程，以及掌握了一些软件的使用）。

i> 本科毕设题目为《基于深度学习的图像去雾方法研究与实现》

题目简单易懂，也就是两点：

- 要**基于深度学习**
- 目的就是做**图像去雾**。

也就是：

![](https://ae01.alicdn.com/kf/HTB10.78c8Kw3KVjSZTEq6AuRpXab.jpg)

嗯，既然目标明确，那就撸起袖子开始干吧。

## 2. 任务时间线

其实我也记不清所有的时间线了，但是还是知道个大概。

个人基础：在深度学习领域上，个人最开始只懂点Python，所以就从这里出发讲讲学习的过程吧。

### 2.1 深度学习起航-2月

1月份的大部分时间都集中在了**毕业实习**上,直到**1/26日**左右，从师兄那儿借了本《Tensorflow：实战Google深度学习框架》，就是下面这本书：

![](https://ae01.alicdn.com/kf/HTB1IH..c8Kw3KVjSZTEq6AuRpXa7.jpg)

开始搭建深度学习TensorFlow开发环境。这分为了**Win10**上搭建和**Ubuntu**上搭建。

i> 起初我是在Win10上搭建的，但后期怀疑Linux上能跑得更快，故后期又在Ubuntu搭建了深度学习环境。当然后期发现，**Ubuntu上的GPU确实是比Win上高**。搭建环境推荐博文：

[post url="https://www.cnblogs.com/guoyaohua/p/9265268.html" title="Win10 Anaconda下TensorFlow-GPU环境搭建详细教程（包含CUDA+cuDNN安装过程）"/]

[post url="https://blog.csdn.net/weixin_41863685/article/details/80303963" title="Ubuntu18.04深度学习GPU环境配置" /]



之后在开学以前都是在学习《Tensorflow：实战Google深度学习框架》这本书，**懒懒散散地到开学时大约还剩两章没看。**



**这段时间总结：**

√>  1. 学会如何搭建TensorFlow的开发环境。2.学会深度学习中的部分理论，包括梯度下降回馈，卷积神经网络的相关理论，观看李宏毅老师的深度学习视频等等。



推荐几个学习CNN，深度学习的链接：

- [感知机可视化](https://playground.tensorflow.org/)
- [CNN可视化](http://scs.ryerson.ca/~aharley/vis/conv/flat.html)，以MNIST手写体识别为例，可视化整个CNN过程，可用于辅助理解卷积操作、卷积核、池化层等等。
- [An Intuitive Explanation of Convolutional Neural Networks](https://ujjwalkarn.me/2016/08/11/intuitive-explanation-convnets/) **推荐**
- [深度学习理论知识(李宏毅)](https://www.bilibili.com/video/av54344731)，李宏毅老师的视频很多，涉及到的东西很宽，适合想全面入手深度学习的同学。
- [Tensorflow 搭建自己的神经网络 (莫烦 Python 教程)](https://www.bilibili.com/video/av16001891?from=search&seid=14694185683153094899)



### 2.2 聚焦论文-3月

2月底开学了，在师兄的敦促下，发现自己的进度似乎过慢，毕竟这个时候也只是懂点CNN原理，tf的部分操作罢了。毕设的重点**“图像去雾”**完全没接触，于是在一边完成《实战Google深度学习框架》这本书的同时，开始刚近年的论文了。

3月初读完了那本书，大部分代码照着敲了一遍，然后整个月基本上是在读论文中渡过的，其实最开始读论文我是非常拒绝的(去你M的英文)，不过没办法，必须读下去啊。**最开始读取的是一篇英文短期刊，发现里面的网络结构实在看不懂，什么跳跃连接，金字塔结构，书上根本没提到过，去雾的理论在一篇短期刊文章中也没办法读懂。**

读的第一篇文章:

![](https://ae01.alicdn.com/kf/HTB1rJFTdlaE3KVjSZLeq6xsSFXaz.jpg)

后来学聪明了，不从英文期刊中读，**改到去读国内硕士毕业论文**，这样的文章一是中文对我更友好，二是介绍更全面;弊端就是太长，读起来有些累，不过初入门多读点总是好的。

随着文章越读越多，是时候**找个文献管理器**了，曾经做数模时就用过的一个文献管理器再次排上用场--**zotero**。不知道给多少人安利过这款软件了，反正就是特别好用啦，不过是在加上一定的插件和坚果云备份的场景下。如果你有一个**ipad**的话，可以使用**papership**这款软件，通过**坚果云和zotero的文献库同步**，这样ipad上也可以读文献。

我读论文的习惯有两个：1.使用pdf批注工具。2.使用markdown进一步提炼内容，并添加到zotero文献库对象项中。如：

![](https://ae01.alicdn.com/kf/HTB1YrF0dlGE3KVjSZFhq6AkaFXa2.jpg)

纪念一下毕设期间读过的论文，还是蛮有成就感的。

![](https://ae01.alicdn.com/kf/HTB1W4JQdoGF3KVjSZFoq6zmpFXaY.jpg)



文献是理解理论的重要依据，但是文献的弊端就是**“将一个很简单的东西用各种术语复杂化”**，所以更多的理解途径是带着对文献的迷惑去百度去谷歌，在别人的博文上找到答案。随着读的博文增多，如何去管理读过的博文呢？我又用了**两个工具**（其实是三个，其中一个废弃掉了）。

- *印象笔记+印象笔记剪切插件（已废弃）*

- Diigo：Chrome插件，在网页上做笔记（类似于pdf做笔记），并转存到在线库中。这个插件其实是也不是重点，效果可见下图：

  ![](https://ae01.alicdn.com/kf/HTB1_clRdhiH3KVjSZPfq6xBiVXah.jpg)

![](https://ae01.alicdn.com/kf/HTB18PJ.daSs3KVjSZPi763siVXae.png)

第一张图是在一个网站上做了高亮的操作，然后通过点击插件图标选择Library，可进入到曾经有过标注的库。当然这只是其中一个功能，pdf标注，截图等等，这个插件都是可以实现的。

- **OneNote+OneNote剪贴插件**，强烈推荐。我是从印象笔记转到OneNote的，以前一直听有人说OneNote赛高来着，去百度了一波印象笔记和OneNote的区别，感觉也就那这样，但是又一次闲得蛋疼的去安装了OneNote并且系统的学习了一下后，个人觉得印象笔记弱爆了。那么OneNote如何去辅助我们读博文呢。

  ![](https://ae01.alicdn.com/kf/HTB1PgdSdfWG3KVjSZFPq6xaiXXao.jpg)

  ![](https://ae01.alicdn.com/kf/HTB15QJRdf1H3KVjSZFBq6zSMXXa1.jpg)

这里还看不出和印象笔记的区别，但是OneNote有太多好用的功能，最喜欢的是“**停靠窗口“和”链接笔记“**了。本节总结中我会给出OneNote的学习视频。

说了这么多工具的使用，似乎都跑题了。嘛，就算[干货推荐篇](https://www.ravenxrz.ink/2019/05/03/over-the-years-i-have-used-the-best-software-pc-dry-chapter.html)的补充吧。

**这段时间总结：**

√> 1. 完成实战框架这本书。2. 读了很多去雾领域的相关论文。3. 掌握了几个用于辅助学习的软件工具。

推荐工具和链接：

- [Zetero](https://www.zotero.org/),我安装过的插件列表，如何安装使用请自行百度了：

  ![](https://ae01.alicdn.com/kf/HTB13Lh6da5s3KVjSZFNq6AD3FXa6.jpg)

- [十课精通onenote 2016—onenote教程，靠谱学院，星月制作](https://www.bilibili.com/video/av4805650?from=search&seid=10442147840652969199)

- [kopernio--文献寻找插件](https://kopernio.com/)

### 2.3 论文复现-4月

3月读了较多论文，代码实现倒还没有多少，所以这个月主要是做代码实现的工作。再此之前，只学习过TensorFlow，不过感觉tf做起来还是有点繁琐，所以我在寻找有没有什么替代方案，实现起来简单高校的。于是就学习了**Keras**。Keras封装得更为高层，以函数式编程来实现神经网络相当方便。于是从简单的MLP到CNN，MNIST全部熟悉了一遍。然后又学习了**自编码器、（条件）生成对抗网络**的相关理论及Keras实现。至此，Keras基本上是掌握了。

约4月中上旬，准备着手做论文复现工作，按以下条件去选复现的：

- 经典去雾算法，几乎每篇论文都会拿来做对比的，于是选中了DCP。
- 简单易实现且引用率比较高的，于是选中了MSCNN。
- 已有开源的代码，并且是以tf或Keras实现的，于是选中了《Single Image Haze Removal using a Generative Adversarial Network》。

做复现工作首先要有**训练数据集**（当然DCP不是深度学习方法，不需要训练集，不过也需要测试集呀），所以做了以下工作：

1. 通过NYU数据集合成雾图。
2. 直接引用了RESIDE数据集。

数据集是第一步，有了数据集就准备复现了。通过参考博文、开源代码算是做完了这几篇复现工作吧。不过MSCNN的Keras版本我没至今没有训练出来，对比过作者提供的Matlab版本，发现除输入图片size不一样，其余参数全部一样，训练集都是一样的（但是我归一化resize处理）。

x> 说一下为什么输入size不一致，因为MSCNN是全卷积网络，按道理是可以支持任何size的，Matlab基于操作来实现，所以可以支持任何size，但是keras不一样，它是基于层来实现的，必须要指定输入size，所以我才将训练集进行了归一化resize处理。

这个月还做了一个工作就是着手写毕设论文，因为师兄觉得我太慢了，于是就写了论文的前两章。

4月玩得还是蛮浪的，工作量不算大。

所有复现论文的代码都已经开源，地址将贴在本文文末。

收获如下：

√>  1.学习了Keras，更容易实现网络。2.学习合成雾图原理，并作了自己的实现。下载网上提供的RESIZE的雾图数据集。3. 复现了DCP,MSCNN和GAN去雾。4. 完成毕设前两章。

推荐链接：

1. [Netron-Kears网络结构可视化工具](https://github.com/lutzroeder/netron),这个工具还是很有用的，经常用来看网络结构是否出错，复现时看是否和别人的网络结构一致。效果如下图：

   ![](https://ae01.alicdn.com/kf/HTB1uBdYdf1H3KVjSZFHq6zKppXaG.jpg)

### 2.4 实现毕设中网络结构-5月

虽说完成了几个去雾网络结构，但是到底该如何去做自己的网络结构来去雾却是一点头绪都没有。因为要我去修改网络结构，我可以说是没有任何理论支撑，给我一张网络图去实现，那没问题。所以这个时候就求助师兄了，后来师兄给了一篇**DeBlurGan**的paper，这篇文章算是我的救星，虽然不是Dehaze的论文，但是把它迁移过来试试看。花了差不多大半天的时间，完成了迁移，没想到train起来效果相当好，跑完一个epoch就可以得到清晰的图像。最后跑完50个epoch大约花了8个小时。

迁移的代码也已开源，将在文末贴出开源地址。同时在之后会给出一篇文章专门阐述这个DeBlurGan，以及实现过程。

后期就是做测试了，包括复现的算法的主观客观测试，指标统计等。

完成了去雾实验，5月的后半段就全是在写论文了，约在5/26日，完成了所有论文，修订，查重等。

### 2.5 终章-答辩-6月

完成了论文后，其实我已经飘了，玩了大概一个星期，端午的前一天突然被通知要预答辩，可是我连ppt都没做啊。于是又跑回学校去做ppt，准备材料。预答辩来得及，要修改的东西也蛮多的，所以飘的后果就是熬夜加班改东西。

最终，6月11日，完成了答辩。

6月17日，拿到了结果，only B+。 其实觉得自己应该还是能得A的，虽然5个月来中间也蛮浪的，但是也还算努力吧。嘛，反正过了就行。重要的是收获了很多嘛。

## 3. 结尾

从1月到6月，5月时间，体验了一把“课题”的感受，为未来的日子打打基础，收获了很多。也要感谢师兄5个月来的各种帮助。

最后，毕业快乐。

所有去雾代码的开源地址：https://github.com/raven-dehaze-work

![](https://ae01.alicdn.com/kf/HTB1Uth2doCF3KVjSZJnq6znHFXag.jpg)
