---
title: 毕设总结五-杂项
categories: 深度学习
toc: true
tags: 图像去雾
abbrlink: 2caf6b87
date: 2019-07-19 15:58:58
---

i> 本科毕设题目为《基于深度学习的图像去雾方法研究与实现》

毕设总结系列：
- [毕设总结零--绪论](https://www.ravenxrz.ink/archives/880ba6e.html)
- [毕设总结一--传统去雾理论](https://www.ravenxrz.ink/archives/a513887d.html)
- [毕设总结二--论文复现工作1--暗通道去雾(DCP)](https://www.ravenxrz.ink/archives/81fcc536.html)
- [毕设总结三--论文复现工作2--MSCNN去雾](https://www.ravenxrz.ink/archives/15ae521b.html)
- [毕设总结四--最终去雾方案的确定与实现--DeBlurGanToDehaze](https://www.ravenxrz.ink/archives/6631bc9b.html)
- [毕设总结五--杂项](https://www.ravenxrz.ink/archives/2caf6b87.html)
<!-- more -->
## 1.  前言

上一篇文章中，介绍了笔者在毕设中所采用的去雾方法及其代码实现。这一篇算是总结一些在毕设过程中的一些杂项事物，以及收尾吧。

## 2.  一些杂项

### 2.1 去雾新想法

- CycleGan的思想其实也很有趣，但是我没有去实现了，算是一个思路吧。
- DeBlurGan中可以尝试把ResBlock改为DenseBlock，但是注意DenseBlock很容易显存爆炸，我当初是实现了DenseBlock的，但是无奈显存直接爆炸了。曾经看过Pytorch似乎可以解决DenseBlock显存爆炸的问题，但是keras暂时还没有。显存爆炸的原因我已经忘记了，所以无法贴出相关链接。但是当时是在Github的issue上看到的，算是一条线索吧。

### 2.2 展望与不足

- 虽然没有实际测试服务器的去雾速度，但是个人感觉还是很快的。那么这样其实可以做一个**视频实时去雾**的应用，比如在咱们重庆高速公路上雾天天气还是蛮频繁的，做一个实时去雾处理可以大大降低车祸事故率。**什么？担心上传雾图下载雾图速度慢？**5G都要来了，咱应该不怕这个。更何况也可以把网络迁移到硬件设备上去，参考[TensorFlow Lite](https://www.tensorflow.org/lite)。

  ---

  **-------------下段话已过时。-------------**

  如果有读过我的代码的小伙伴，肯定知道不论是train还是test，我都把整个网络的输入固定了size，这就造成了整个网络只接受固定size的图片，其余size的图片需要做resize处理，resize肯定会失真，这是一个缺点。修复这个缺点的重点有两个：

  1. 把判别网络后两个FC层改为多个卷积层，那就实现了整个网络（包括生成网络和判别网络）为全卷积网络，如此一来可以接收任意size的图片了。
  2. 据个人所知，Keras和TensorFlow这两种框架必须固定输入size，要想实现任意size，可用matlab（matlab是基于操作来实现的，所以肯定可以实现全卷积），其余框架如caffe,pytorch等等我没有学习，无法给出确定答案。

  ---

  **2019/8/27更新**

  感谢评论区**@lihanyu1204**的提醒，理论上keras和tensorflow是可以接收任意size的输入的（前提是你的网络为全卷积）。个人试验后也的确如此，不过建议train时还是要固定size，test可不固定。另外，如果固定了输入size来train，判别网络也就不用将后两个FC层更改为卷积层了。代码实现如下：

  ```python
   if mode == 'train':
          inputs = Input(shape=image_shape)
      elif mode == 'test':
          inputs = Input(shape=(None,None,3))
  ```

  其中`image_shape`是固定的图片大小，在我的代码里为`(256,256,3)`。

  当然了，这只是理论上可以接收任意size，实际情况还要看你的网络结构，由于我的网络设计已经固定，**在不修改网络结构的情况下只能接收图片宽高能整除4的这类图片。**因为我的网络结构最前两层卷积降维，最后两层升维，且单次升降仅让图片size乘2或除2。如果遇到输入不能整除4的情况，会造成输入输出的size不一致。例如输入为450, 经过第一层为225,第二层为113（padding='same'的情况下，向上取整），倒数第二层为226，最后一层输出为452。所以我的网络只能接收能整除4的。

  **如果你要考虑接收任意size，则要考虑好网络结构。**

  ---

  **新问题**

  不过这样又会引发一个新问题，那就是在keras的predict函数中所接受的输入需要具有相同的shape，否则会报错。个人的解决方案是采用循环predict，每次循环仅predict一张图片。不知道keras有不有相应的predict函数能够适应不同的输入大小。

### 2.3   Android app展示去雾？

看过本系列四的同学，应该注意到我在文末贴了一个视频，演示的是Android App上传图片到服务器，服务器处理回传后Android显示的效果。请注意那个时候Android app和服务器并不是在一个局域网内，因为我做这个app的初衷是为了答辩演示，答辩的时候肯定和寝室的台式机不在一个局域网内，于是做了内网穿透。关于这段实现过程，可参见我的另一篇博文

[post cid="194" /]

### 2.4 Latex 写毕设论文？

说到Latex，我真的是又爱又恨。爱的是他的简单美观，恨的是他的复杂难看。看到这里的小伙伴一定觉得我疯了，说了一堆反义词。但是Latex就是这样，**如果有提供好的Latex模板**，那写起来就很舒服，就像一个程序你只用给它一个简单输入，它在后台帮你把所有排版、公式编号、插图等等全部安排好。**如果没有提供好的模板**，那就要自己去实现，我花了一个星期的时间努力搞出本校的模板，但是我放弃了，太多细节的东西没法修复实现（全怪自己是Latex的弱鸡）。后来Latex对我来说的唯一作用就是拿来写公式，再转到word中去。

i> 所以一条建议：先去TexStudio工作室搜搜有不有你本校的模板，没有的话尽量就用word编排，当然如果你对自己的Latex能力有信心，那就做一份吧，还能为你本校的后辈们留点东西，方便后辈们写论文咯。

[post cid="145"/]

### 2.5 其他参考资料

- [自编码器 Autoencoders with Keras](https://ramhiser.com/post/2018-05-14-autoencoders-with-keras/)
- [Kears中如何保存模型？](How to Check-Point Deep Learning Models in Keras)
- [经典CNN之：VGGNet - 机器会学习的博客 - CSDN博客](https://blog.csdn.net/u014281392/article/details/75152809)
- [深度学习---残差resnet网络原理详解 - Dean - CSDN博客](https://blog.csdn.net/qq_38906523/article/details/79769571)
- [大话深度残差网络（DRN）ResNet网络原理 - 雪饼的个人空间 - OSCHINA](https://my.oschina.net/u/876354/blog/1622896)
- [DenseNet算法详解 - AI之路 - CSDN博客](https://blog.csdn.net/u014380165/article/details/75142664/)
- [感知损失(Perceptual Losses)](https://blog.csdn.net/stdcoutzyx/article/details/54025243)
- [转置卷积、去卷积概念梳理](https://buptldy.github.io/2016/10/29/2016-10-29-deconv/)
- [训练loss不下降原因集合 - jacke121的专栏 - CSDN博客](https://blog.csdn.net/jacke121/article/details/79874555)
- [图像质量评价指标之 PSNR 和 SSIM - 知乎](https://zhuanlan.zhihu.com/p/50757421)

## 3. 结语

总算是写到了本系列的最后一篇文章了，这个系列的结束，基本上意味着我在学校的最后工作也做完了，感谢在毕设5个月来一直帮助我的师兄，
接下来就是浪咯，好久没玩游戏了，最近应该也不会更新博文了吧，去放松一把啦。
