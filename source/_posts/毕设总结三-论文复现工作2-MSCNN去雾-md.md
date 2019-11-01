---
title: 毕设总结三-论文复现工作2-MSCNN去雾
categories: 深度学习
toc: true
tags: 图像去雾
abbrlink: 15ae521b
date: 2019-07-18 13:58:55
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

上一篇文章中，我们详细讲解分析了传统去雾方法DCP的原理及实现，本篇文章中，我们再来谈谈近年来火热的“深度学习”以及如何应用深度学习进行去雾。

本篇中的所有代码均已开源，地址将放于文末。

## 2. 一点题外话--浅谈深度学习

自2012年ILSVRC竞赛中，基于深度学习的AlexNet已绝对优势获得冠军后，深度学习开始爆热。深度学习几乎成为AI的代名词，现已被运用到如自然语言处理、图像分类、语音识别等等诸多领域。要学习deep learning， 首先要知道什么是深度学习，以及”人工智能“、“机器学习“、”深度学习“的区别。

引用一篇博文中的定义，深度学习可这样定义：

> 1.wiki：深度学习是机器学习的分支，它试图使用包含复杂结构或者由多重非线性变换构成的多个处理层对数据进行高层抽象的算法。
> 2.李彦宏：简单的说，深度学习就是一个函数集，如此而已。
> 3.深度学习将特征提取和分类结合到一个框架中，用数据学习特征，是一种可以自动学习特征的方法。
> 4.深度学习是一种特征学习方法，把原始的数据通过非线性的复杂模型转换为更高层次、更抽象的表达。

下图很直观的展示这三者之间的关系。

![](https://ae01.alicdn.com/kf/HTB1fYn8dlKw3KVjSZFOq6yrDVXaL.jpg)

再来谈谈近年来深度学习中的主角，卷积神经网络(CNN)。它有什么特点呢？一篇经典的文章里是这样描述它的：

> Convolutional Neural Networks (**ConvNets** or **CNNs**) are a category of [Neural Networks](https://ujjwalkarn.me/2016/08/09/quick-intro-neural-networks/) that have proven very effective in areas such as image recognition and classification. ConvNets have been successful in identifying faces, objects and traffic signs apart from powering vision in robots and self driving cars.

也就是说，它特别**擅长图像识别处理与分类**。

这里推荐几篇文章学习CNN与了解CNN的进化史：

- [An Intuitive Explanation of Convolutional Neural Networks](https://ujjwalkarn.me/2016/08/11/intuitive-explanation-convnets/)
- [CNN可视化](http://scs.ryerson.ca/~aharley/vis/conv/flat.html)
- [CNN概念之上采样，反卷积，Unpooling概念解释 - g11d111的博客 - CSDN博客](https://blog.csdn.net/g11d111/article/details/82350563)
- [CNN视频教程](https://www.bilibili.com/video/av54311437/?p=19)
- [CNN网络架构演进：从LeNet到DenseNet - Madcola - 博客园](https://www.cnblogs.com/skyfsm/p/8451834.html)

----------------



## 3.  MSCNN去雾

上小节算是一点题外话了，但是这篇的主题是基于深度学习的去雾，那么至少要知道什么是深度学习以及深度学习中最常用的CNN吧。

MSCNN,全称：《Single Image Dehazing via Multi-scale Convolutional Neural Networks》，由Ren等人于2016年提出。其思想在于**学习雾图与其对应的透射图之间的映射关系，在基于某种算法估计大气光A，最后应用大气散射模型恢复出雾图。**（不懂大气散射模型去雾的可参考本系列总结一）

个人认为这篇文章有3个核心关注点：

1. 提出了一种多尺度训练网络，这个网络分为了粗颗粒估计和细颗粒估计两个子网络。
2. 采用NYU数据集合成出训练网络所有的雾图-透射图训练对。
3. 利用估计出来的透射图和大气光，利用大气散射模型去雾。

### 3.1  MSCNN中的网络结构

先上一章总图：

![](https://ae01.alicdn.com/kf/HTB10WIwX3FY.1VjSZFqq6ydbXXaP.jpg)

可以看到上半部为Coarse Net，下半部分为其Fine Net。由Coarse Net估计出Coarse Transmission Map，然后将它级联到Fine Net的第一个Upsample层后，在进行精细提取。简单说一下它的参数设置：

每一个长方体代表的是特征图，下面的数字代表的是卷积核的个数，下面的文字代表相应的操作和操作的size。

**训练损失函数上**，MSCNN采用了MSE函数,即：

$$
L(t_i(x),t_i^{\\delta}(x)) = \\frac{1}{q}\\sum_{i=1}^{q}||t_i(x)-t_i^{\\delta}(x)||^2
$$

其中，q代表的是一个batch size中的雾图个数。

**其余参数：**

- 采用sgd优化器，动量参数设置为0.9
- batch size 设置为100
- 输入图进行归一化处理，统一为320*240
- 学习率设置0.01并且每20个epoch衰减0.1
- 总epoch设置为70

### 3.2 训练数据合成

作者采用了NYU数据集来构建训练集。NYU数据集提供了清晰图与对应的深度图。合成公式：
$$
\\left\\{
\\begin{array}{l}
t(x) = e^{-\\beta d(x)}\\\\
I(x) = t(x) J(x) + A(1-t(x))
\\end{array}
\\right.
$$
现在已知d(x)和J(x)，通过随机取$\\beta$和$A$，即可得到$t(x)$ 进而 获取$I(x)$雾图。详细的合成方法可参考:

- [Benchmarking Single Image Dehazing and Beyond](https://www.google.com/url?q=https%3A%2F%2Farxiv.org%2Fpdf%2F1712.04143.pdf&sa=D&sntz=1&usg=AFQjCNHzdt3kMDsvuJ7Ef6R4ev59OFeRYA) 
- [RESIDE](https://sites.google.com/view/reside-dehaze-datasets/reside-v0)

### 3.3 整个去雾流程

![](https://ae01.alicdn.com/kf/HTB1LKf7dgmH3KVjSZKzq6z2OXXas.jpg)

## 4. 搭建MSCNN网络结构参考代码

代码基于Keras结构，不过MSCNN有MATLAB版本和Tensorflow版本，连接都放在了文末。

### 4.1 设置超参数和辅助参数

```python
self.model_dir_path = './model_save'
self.trans_img_dir = './dehazed_result/image/trans'
self.trans_npy_dir = './dehazed_result/npy/trans'
if not os.path.exists(self.model_dir_path):
    os.mkdir(self.model_dir_path)
if not os.path.exists(self.trans_img_dir):
    os.mkdir(self.trans_img_dir)
if not os.path.exists(self.trans_npy_dir):
    os.mkdir(self.trans_npy_dir)
self.coarse_model_name = 'coarse_net.h5'
self.fine_model_name = 'fine_net.h5'

# 设置超参数
self.batch_size = batch_size
self.epochs = epochs

# 输入图片信息
self.img_height = 240
self.img_width = 320
self.channel = 3
```

### 4.2 建立coraseNet

```python
def _build_coarseNet(self, input_img):
    """
    建立coarseNet
    :param input_img: 输入图片的tensor
    :return: coarseNet
    """
    conv1 = Conv2D(5, (11, 11), padding='same', activation='relu', name='coarseNet/conv1')(input_img)
    pool1 = MaxPooling2D((2, 2), name='coarseNet/pool1')(conv1)
    upsample1 = UpSampling2D((2, 2), name='coarseNet/upsample1')(pool1)
    normalize1 = BatchNormalization(axis=3, name='coarseNet/bn1')(upsample1)
    # dropout1 = Dropout(0.5, name='coarseNet/dropout1')(normalize1)

    conv2 = Conv2D(5, (9, 9), padding='same', activation='relu', name='coarseNet/conv2')(normalize1)
    pool2 = MaxPooling2D((2, 2), name='coarseNet/pool2')(conv2)
    upsample2 = UpSampling2D((2, 2), name='coarseNet/upsample2')(pool2)
    normalize2 = BatchNormalization(axis=3, name='coarseNet/bn2')(upsample2)
    # dropout2 = Dropout(0.5, name='coarseNet/dropout2')(normalize2)

    conv3 = Conv2D(10, (7, 7), padding='same', activation='relu', name='coarseNet/conv3')(normalize2)
    pool3 = MaxPooling2D((2, 2), name='coarseNet/pool3')(conv3)
    upsample3 = UpSampling2D((2, 2), name='coarseNet/upsample3')(pool3)
    # dropout3 = Dropout(0.5, name='coarseNet/dropout3')(upsample3)

    linear = LinearCombine(1,name='coarseNet/linear_combine')(upsample3)
    return linear
```

`LinearCombine`为一个自定义层:

```python
class LinearCombine(Layer):
    """
    paper 中的线性结合层
    """
    def __init__(self, output_dim,**kwargs):
        self.output_dim = output_dim
        super(LinearCombine, self).__init__(**kwargs)

    def build(self, input_shape):
        self.kernel = self.add_weight(name='kernel',
                                      trainable=True,
                                      shape=(input_shape[3],),
                                      initializer='uniform')
        self.biases = self.add_weight(name='bias',
                                      trainable=True,
                                      shape=(self.output_dim,),
                                      initializer='normal')
        super(LinearCombine, self).build(input_shape)  # Be sure to call this at the end

    def call(self, inputs, **kwargs):
        out = K.bias_add(
            K.sum(tf.multiply(inputs, self.kernel), axis=3),
            self.biases
        )
        out = K.expand_dims(out,axis=3)
        return K.sigmoid(out)

    def compute_output_shape(self, input_shape):
        return (input_shape[0],input_shape[1],input_shape[2],self.output_dim)
```

### 4.3 建立FineNet

```python
def _build_fineNet(input_img, coarseNet):
    """
    建立fineNet
    :param input_img: 输入图片的tensor
    :param coarseNet: coarseNet的Tensor
    :return: fineNet
    """
    # paper中的fine net 卷积kernel为4. 但经查看作者提供的源代码，第一层设置的6
    conv1 = Conv2D(6, (7, 7), padding='same', activation='relu', name='fineNet/conv1')(input_img)
    pool1 = MaxPooling2D((2, 2), name='fineNet/pool1')(conv1)
    upsample1 = UpSampling2D((2, 2), name='fineNet/upsample1')(pool1)

    # 级联coarseNet
    concat = concatenate([upsample1, coarseNet], axis=3, name='concat')
    normalize1 = BatchNormalization(axis=3, name='fineNet/bn1')(concat)
    # dropout1 = Dropout(0.5, name='fineNet/dropout1')(normalize1)

    conv2 = Conv2D(5, (5, 5), padding='same', activation='relu',name='fineNet/conv2')(normalize1)
    pool2 = MaxPooling2D((2, 2), name='fineNet/pool2')(conv2)
    upsample2 = UpSampling2D((2, 2), name='fineNet/upsample2')(pool2)
    normalize2 = BatchNormalization(axis=3, name='fineNet/bn2')(upsample2)
    # dropout2 = Dropout(0.5, name='fineNet/dropout2')(normalize2)

    conv3 = Conv2D(10, (3, 3), padding='same', activation='relu',name='fineNet/conv3')(normalize2)
    pool3 = MaxPooling2D((2, 2), name='fineNet/pool3')(conv3)
    upsample3 = UpSampling2D((2, 2), name='fineNet/upsample3')(pool3)
    # dropout3 = Dropout(0.5, name='fineNet/dropout3')(upsample3)

    linear = LinearCombine(1, name='fineNet/linear_combine')(upsample3)
    return linear
```

### 4.4  设置训练的参数

```python
# 设置优化器，损失函数等
self.optimizer = SGD(learning_rate,0.9,0.0005)
# self.optimizer = Adam(learning_rate, decay=1e-6)
self.loss = mse

self.coarseModel.compile(optimizer=self.optimizer,
                         loss=self.loss)
self.fineModel.compile(optimizer=self.optimizer,
                       loss=self.loss)

```

### 4.5  全代码

```python
"""
网络模型搭建
"""
import os
from keras import backend as K
import tensorflow as tf
from keras.engine.topology import Layer
from keras.models import Model
from keras.layers import Conv2D, MaxPooling2D, UpSampling2D, Input, BatchNormalization
from keras.layers.merge import concatenate


def build_net_model():
    """
    建立paper中的MSCNN
    :return: corseModel和fineModel的元祖
    """
    img_height = 240
    img_width = 320
    channel = 3

    # 通用输入
    input_img = Input((img_height, img_width, channel))
    coarseNet = _build_coarseNet(input_img)
    fineNet = _build_fineNet(input_img, coarseNet)

    # 建立coarse Model和fine Model
    coarseModel = Model(inputs=input_img, outputs=coarseNet)
    fineModel = Model(inputs=input_img, outputs=fineNet)

    # summary
    coarseModel.summary()
    fineModel.summary()

    return (coarseModel, fineModel)


def _build_coarseNet(input_img):
    """
    建立coarseNet
    :param input_img: 输入图片的tensor
    :return: coarseNet
    """
    conv1 = Conv2D(5, (11, 11), padding='same', activation='relu', name='coarseNet/conv1')(input_img)
    pool1 = MaxPooling2D((2, 2), name='coarseNet/pool1')(conv1)
    upsample1 = UpSampling2D((2, 2), name='coarseNet/upsample1')(pool1)
    normalize1 = BatchNormalization(axis=3, name='coarseNet/bn1')(upsample1)
    # dropout1 = Dropout(0.5, name='coarseNet/dropout1')(normalize1)

    conv2 = Conv2D(5, (9, 9), padding='same', activation='relu', name='coarseNet/conv2')(normalize1)
    pool2 = MaxPooling2D((2, 2), name='coarseNet/pool2')(conv2)
    upsample2 = UpSampling2D((2, 2), name='coarseNet/upsample2')(pool2)
    normalize2 = BatchNormalization(axis=3, name='coarseNet/bn2')(upsample2)
    # dropout2 = Dropout(0.5, name='coarseNet/dropout2')(normalize2)

    conv3 = Conv2D(10, (7, 7), padding='same', activation='relu', name='coarseNet/conv3')(normalize2)
    pool3 = MaxPooling2D((2, 2), name='coarseNet/pool3')(conv3)
    upsample3 = UpSampling2D((2, 2), name='coarseNet/upsample3')(pool3)
    # dropout3 = Dropout(0.5, name='coarseNet/dropout3')(upsample3)

    linear = LinearCombine(1, name='coarseNet/linear_combine')(upsample3)
    return linear


def _build_fineNet(input_img, coarseNet):
    """
    建立fineNet
    :param input_img: 输入图片的tensor
    :param coarseNet: coarseNet的Tensor
    :return: fineNet
    """
    # paper中的fine net 卷积kernel为4. 但经查看作者提供的源代码，第一层设置的6
    conv1 = Conv2D(6, (7, 7), padding='same', activation='relu', name='fineNet/conv1')(input_img)
    pool1 = MaxPooling2D((2, 2), name='fineNet/pool1')(conv1)
    upsample1 = UpSampling2D((2, 2), name='fineNet/upsample1')(pool1)

    # 级联coarseNet
    concat = concatenate([upsample1, coarseNet], axis=3, name='concat')
    normalize1 = BatchNormalization(axis=3, name='fineNet/bn1')(concat)
    # dropout1 = Dropout(0.5, name='fineNet/dropout1')(normalize1)

    conv2 = Conv2D(5, (5, 5), padding='same', activation='relu',name='fineNet/conv2')(normalize1)
    pool2 = MaxPooling2D((2, 2), name='fineNet/pool2')(conv2)
    upsample2 = UpSampling2D((2, 2), name='fineNet/upsample2')(pool2)
    normalize2 = BatchNormalization(axis=3, name='fineNet/bn2')(upsample2)
    # dropout2 = Dropout(0.5, name='fineNet/dropout2')(normalize2)

    conv3 = Conv2D(10, (3, 3), padding='same', activation='relu',name='fineNet/conv3')(normalize2)
    pool3 = MaxPooling2D((2, 2), name='fineNet/pool3')(conv3)
    upsample3 = UpSampling2D((2, 2), name='fineNet/upsample3')(pool3)
    # dropout3 = Dropout(0.5, name='fineNet/dropout3')(upsample3)

    linear = LinearCombine(1, name='fineNet/linear_combine')(upsample3)
    return linear


class LinearCombine(Layer):
    """
    paper 中的线性结合层
    """

    def __init__(self, output_dim, **kwargs):
        self.output_dim = output_dim
        super(LinearCombine, self).__init__(**kwargs)

    def build(self, input_shape):
        self.kernel = self.add_weight(name='kernel',
                                      trainable=True,
                                      shape=(input_shape[3],),
                                      initializer='uniform')
        self.biases = self.add_weight(name='bias',
                                      trainable=True,
                                      shape=(self.output_dim,),
                                      initializer='normal')
        super(LinearCombine, self).build(input_shape)  # Be sure to call this at the end

    def call(self, inputs, **kwargs):
        out = K.bias_add(
            K.sum(tf.multiply(inputs, self.kernel), axis=3),
            self.biases
        )
        out = K.expand_dims(out, axis=3)
        return K.sigmoid(out)

    def compute_output_shape(self, input_shape):
        return (input_shape[0], input_shape[1], input_shape[2], self.output_dim)

```

### 4.6 其他

代码中还有很多需要注意的地方，如接力学习，训练数据对的统一化处理，显存不足时，不应当将所有图片一次性加入到内存中，而应该采用generator的方式来训练，train过程多少epoch或batch需要sample一次等等。这些代码都可以参考文末的总代码，这里不可能一一道尽。

## 参考

1. [Single Image Dehazing via Multi-scale Convolutional Neural Networks](https://link.springer.com/chapter/10.1007/978-3-319-46475-6_10)

所有源代码地址:

1. https://github.com/raven-dehaze-work/MSCNN_Keras
2. https://github.com/raven-dehaze-work/MSCNN_MATLAB
3. https://github.com/dishank-b/MSCNN-Dehazing-Tensorflow


