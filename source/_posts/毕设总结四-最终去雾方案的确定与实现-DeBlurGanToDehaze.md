---
title: 毕设总结四-最终去雾方案的确定与实现-DeBlurGanToDehaze
categories: 深度学习
toc: true
tags: 图像去雾
abbrlink: 6631bc9b
date: 2019-07-19 14:58:59
mathjax: true
---

i> 本科毕设题目为《基于深度学习的图像去雾方法研究与实现》


毕设总结系列：
- [毕设总结零--绪论](https://ravenxrz.github.io/archives/880ba6e.html)
- [毕设总结一--传统去雾理论](https://ravenxrz.github.io/archives/a513887d.html)
- [毕设总结二--论文复现工作1--暗通道去雾(DCP)](https://ravenxrz.github.io/archives/81fcc536.html)
- [毕设总结三--论文复现工作2--MSCNN去雾](https://ravenxrz.github.io/archives/15ae521b.html)
- [毕设总结四--最终去雾方案的确定与实现--DeBlurGanToDehaze](https://ravenxrz.github.io/archives/6631bc9b.html)
- [毕设总结五--杂项](https://ravenxrz.github.io/archives/2caf6b87.html)
<!-- more -->
## 1. 前言

上一篇文章中，我们介绍了MSCNN的去雾方法以及MSCNN的Keras实现。这篇文章中将介绍笔者的毕设中所采用的去雾结构。

## 2.  去雾前夕

### 2.1 端到端的去雾网络

在总结系列的一、二、三中，我们所介绍的理论和去雾方法均是以大气散射模型为基础，以估参作为重点研究过程，包括Aod-Net[1]虽然不是单独估计透射图和大气光，但仍离不开估参。已估参为主的这些去雾方法始终都有一个弊端：**或多或少的都有额外的误差**。

> 能不能建立一种端到端的映射网络呢？给网络输入一张雾图，输出得到一张去雾图。

就像总结零曾有过的一张图那样：

<img src="https://ae01.alicdn.com/kf/HTB10.78c8Kw3KVjSZTEq6AuRpXab.jpg"/>

那什么样的网络可以实现这样的效果呢？

自2014年生成对抗网络（GAN）提出以来，GAN便以一种新式的训练思想逐渐火热。现在已经很多基于GAN的应用，如图像修补，图像风格转换等等。于是自然而然的，笔者采用了GAN来做上图中的去雾网络。

### 2.2 浅谈GAN

去雾GAN的核心思想如下图：

![](https://ae01.alicdn.com/kf/HTB1EboBdf5G3KVjSZPxq6zI3XXaH.jpg)

它有两个重要的子网络，**一个生成网络G，一个判别网络D**。现在假设我们手里有三样东西：

- 设计精良的生成网络G
- 设计精良的判定网络D
- 训练这个大网络所需要的 “ 雾图-无雾图” 多个数据对。

i> 那么这个网络的思想可这样来表达：生成网络G吃一张雾图，吐出一张“假”无雾图，从训练数据集中找到吃掉的雾图的对应“真”无雾图，判别网络D尽自己最大的可能去区分开这“真假”雾图，另一方面，生成网络G尽自己最大的可能去生成接近“真”无雾图的“假”无雾图，用来迷惑判别网络D。两者在这种竞争关系下不断进化，最后达到一种平衡。

关于GAN的文章，推荐阅读：

- [GAN学习指南：从原理入门到制作生成Demo](https://zhuanlan.zhihu.com/p/24767059)
- [简单理解与实验生成对抗网络GAN - 我爱智能 - CSDN博客](https://blog.csdn.net/on2way/article/details/72773771)

- [Keras-GAN](https://github.com/eriklindernoren/Keras-GAN)

- [Image-to-Image Translation with Conditional Adversarial Nets的介绍与应用Demo](https://phillipi.github.io/pix2pix/)

- [Isola et al_2016_Image-to-Image Translation with Conditional Adversarial Networks](Isola et al_2016_Image-to-Image Translation with Conditional Adversarial Networks.pdf)
- [Patch-GAN Image-to-Image Translation with Conditional Adversarial Networks超细致解析：使用条件Gan经行图像的转换](https://www.jianshu.com/p/57ff6f96ce4c)

- [17种GAN变体的Keras实现请收好 | GitHub热门开源代码](https://cloud.tencent.com/developer/article/1067030)

- [GAN with Keras: Application to Image Deblurring – Sicara's blog](https://blog.sicara.com/keras-generative-adversarial-networks-image-deblurring-45e3ab6977b5?gi=f1d5c50e7800)
- [李宏毅GAN视频教学](https://www.bilibili.com/video/av54339012)

## 3. 去雾网络结构

前文我们说了GAN有两个核心的子网络，如何分别设计两个网络的结构呢？**反正我是不会设计**，哈哈。嘛，总之这时候我是求助了师兄，师兄丢给了我一篇DeBlur的论文，然后我做了个Transfering Learning。没想到效果相当好，于是就这样抄过来了。

DeBlur的论文：《DeblurGAN: Blind Motion Deblurring Using Conditional Adversarial Networks》

### 3.1 生成网络结构

![](https://ae01.alicdn.com/kf/HTB1tOehbAxz61VjSZFrq6xeLFXad.jpg)

- 2个编码单元提取特征
- 9个ResBlock作为特征转换器，将它从雾图特征转到无雾特征
- 2个解码单元恢复无雾图

你要问我为什么这样设计？不好意思，我也不知道。*个人觉得深度学习就是一个黑盒子，讲究的是最终效果，往往没有特别好的理论支撑。*

### 3.2 判别网络结构

![](https://ae01.alicdn.com/kf/HTB1E7evcL1G3KVjSZFkq6yK4XXaa.jpg)

判别网络参见《Image-to-Image Translation with Conditional Adversarial Networks》中的PatchGAN。
[关于PatchGAN的理解 - xiaoxifei的专栏 - CSDN博客](https://blog.csdn.net/xiaoxifei/article/details/86506955)

## 4. 损失函数设计

这里损失函数由两项构成：
$$
L = L_{感知损失} +\\lambda L_{对抗损失}
$$
在说感知损失(perception loss)前，先来看看简单的MSE损失，我们曾在MSCNN网络结构中提到过MSE损失函数。他比较的是生成图与标签图之间的像素级别均方差。
$$
L(t_i(x),t_i^{\\delta}(x)) = \\frac{1}{q}\\sum_{i=1}^{q}||t_i(x)-t_i^{\\delta}(x)||^2
$$
那什么感知损失？简单的说就是将生成图和标签图在通过一个网络，得到**两张图的特征图**，在这两张特征图上做MSE。一般来说，通过的这个网路会选择VGG网络。感知损失的计算公式如下：
$$
	L_{感知损失} = \\frac{1}{CHW} \\sum_{j=1}^{3}
	|| \\phi_j(I^{Label}) - \\phi_j(G(I^{haze})) ||^2	
$$

- $\phi$理解为通过网络的等效函数
- CHW分别代表图像通道数，图像高度和宽度

paper中的对抗损失采用的是WGAN-GP中的Loss，但是我通过阅读它的源码发现它只采用了wgan loss。虽然我后期想自己修改为wgan-gp loss，但是无奈，WGAN-GP后面的理论真的好多，加上时间已经不够且当时基于WGAN中的loss效果已经很好了，所以最后未能完成修改。

关于WGAN 和 WAGN-GP推荐阅读：

- [GAN — Wasserstein GAN & WGAN-GP – Jonathan Hui – Medium](https://medium.com/@jonathan_hui/gan-wasserstein-gan-wgan-gp-6a1a2aa1b490)
- [DCGAN、WGAN、WGAN-GP、LSGAN、BEGAN原理总结及对比 - Double_V的博客 - CSDN博客](https://blog.csdn.net/qq_25737169/article/details/78857788)
- [WassersteinGAN-GP](http://www.pianshen.com/article/5135241105/)

## 5. 实现代码参考

### 5.1 构建生成器代码

```python
def generator_model():
    """Build generator architecture."""
    # Current version : ResNet block
    inputs = Input(shape=image_shape)

    x = ReflectionPadding2D((3, 3))(inputs)
    x = Conv2D(filters=ngf, kernel_size=(7, 7), padding='valid')(x)
    x = BatchNormalization()(x)
    x = Activation('relu')(x)

    n_downsampling = 2
    for i in range(n_downsampling):
        mult = 2 ** i
        x = Conv2D(filters=ngf * mult * 2, kernel_size=(3, 3), strides=2, padding='same')(x)
        x = BatchNormalization()(x)
        x = Activation('relu')(x)

    mult = 2 ** n_downsampling
    for i in range(n_blocks_gen):
        x = res_block(x, ngf*mult, use_dropout=True)

    for i in range(n_downsampling):
        mult = 2 ** (n_downsampling - i)
        x = UpSampling2D()(x)
        x = Conv2D(filters=int(ngf * mult / 2), kernel_size=(3, 3), padding='same')(x)
        x = BatchNormalization()(x)
        x = Activation('relu')(x)

    x = ReflectionPadding2D((3, 3))(x)
    x = Conv2D(filters=output_nc, kernel_size=(7, 7), padding='valid')(x)
    x = Activation('tanh')(x)

    outputs = Add()([x, inputs])
    # outputs = Lambda(lambda z: K.clip(z, -1, 1))(x)
    outputs = Lambda(lambda z: z / 2)(outputs)
    model = Model(inputs=inputs, outputs=outputs, name='Generator')
    return model
```

### 5.2 构建判别器

```python
def discriminator_model():
    """Build discriminator architecture."""
    n_layers, use_sigmoid = 3, False
    inputs = Input(shape=input_shape_discriminator)

    x = Conv2D(filters=ndf, kernel_size=(4, 4), strides=2, padding='same')(inputs)
    x = LeakyReLU(0.2)(x)

    nf_mult, nf_mult_prev = 1, 1
    for n in range(n_layers):
        nf_mult_prev, nf_mult = nf_mult, min(2 ** n, 8)
        x = Conv2D(filters=ndf * nf_mult, kernel_size=(4, 4), strides=2, padding='same')(x)
        x = BatchNormalization()(x)
        x = LeakyReLU(0.2)(x)

    nf_mult_prev, nf_mult = nf_mult, min(2 ** n_layers, 8)
    x = Conv2D(filters=ndf * nf_mult, kernel_size=(4, 4), strides=1, padding='same')(x)
    x = BatchNormalization()(x)
    x = LeakyReLU(0.2)(x)

    x = Conv2D(filters=1, kernel_size=(4, 4), strides=1, padding='same')(x)
    if use_sigmoid:
        x = Activation('sigmoid')(x)

    x = Flatten()(x)
    x = Dense(1024, activation='tanh')(x)
    x = Dense(1, activation='sigmoid',name='d-output')(x)

    model = Model(inputs=inputs, outputs=x, name='Discriminator')
    return model
```

### 5.3 构建整个网络结构

```python
def generator_containing_discriminator_multiple_outputs(generator, discriminator):
    inputs = Input(shape=image_shape)
    generated_image = generator(inputs)
    outputs = discriminator(generated_image)
    model = Model(inputs=inputs, outputs=[generated_image, outputs])
    return model
```

### 5.4 训练网络

```python
def train(batch_size, epochs, critic_updates=5):
    """
    训练网络
    :param batch_size:
    :param epochs:
    :param critic_updates: 每个batch_size 中 Discriminator需要训练的次数
    :return:
    """
    # 加载数据
    data_loader = DataLoader(batch_size)

    # 构建网络模型
    g = generator_model()
    # g.summary()
    d = discriminator_model()
    d.summary()
    d_on_g = generator_containing_discriminator_multiple_outputs(g, d)

    # 保存模型结构--用于可视化
    g.save(os.path.join(model_save_dir, "generator.h5"))
    d.save(os.path.join(model_save_dir, "discriminator.h5"))
    d_on_g.save(os.path.join(model_save_dir, "d_on_g.h5"))

    # 编译网络模型
    d_opt = Adam(lr=1E-4, beta_1=0.9, beta_2=0.999, epsilon=1e-08)
    d_on_g_opt = Adam(lr=1E-4, beta_1=0.9, beta_2=0.999, epsilon=1e-08)

    d.trainable = True
    d.compile(optimizer=d_opt, loss=wasserstein_loss)
    d.trainable = False
    loss = [perceptual_loss, wasserstein_loss]
    loss_weights = [100, 1]
    d_on_g.compile(optimizer=d_on_g_opt, loss=loss, loss_weights=loss_weights)
    d.trainable = True

    # 设置discriminator的real目标和fake目标
    output_true_batch, output_false_batch = np.ones((batch_size, 1)), -np.ones((batch_size, 1))
    # tensorboard_callback = TensorBoard(log_dir)

    # TODO: 可以在这里加入恢复权重，接力学习

    # 训练
    start = datetime.datetime.now()
    for epoch in tqdm.tqdm(range(epochs)):
        d_losses = []
        d_on_g_losses = []
        for index in range(data_loader.file_nums // batch_size):
            img_haze_batch, img_clear_batch = next(data_loader.train_generator)
            # 放缩到-1 - 1
            img_haze_batch = img_haze_batch / 127.5 - 1
            img_clear_batch = img_clear_batch / 127.5 - 1

            generated_images = g.predict(x=img_haze_batch, batch_size=batch_size)

            for _ in range(critic_updates):
                d_loss_real = d.train_on_batch(img_clear_batch, output_true_batch)
                d_loss_fake = d.train_on_batch(generated_images, output_false_batch)
                d_loss = 0.5 * np.add(d_loss_fake, d_loss_real)
                d_losses.append(d_loss)

            d.trainable = False

            d_on_g_loss = d_on_g.train_on_batch(img_haze_batch, [img_clear_batch, output_true_batch])
            d_on_g_losses.append(d_on_g_loss)

            d.trainable = True

            # print log
            print('d loss %f d_on_g loss %f' % (d_loss, d_on_g_loss[1] + d_on_g_loss[2]))

            if index % 50 == 0:
                # Test
                img_haze_test, img_clear_test = next(data_loader.test_generator)
                generated_images = g.predict(x=img_haze_test / 127.5 - 1, batch_size=batch_size)
                # 放缩为0-255
                generated_images = (generated_images + 1) * 127.5

                fig, axs = plt.subplots(batch_size, 3)
                for idx in range(batch_size):
                    axs[idx, 0].imshow((img_haze_test[idx].astype('uint8')))
                    axs[idx, 0].axis('off')
                    axs[idx, 0].set_title('haze')

                    axs[idx, 1].imshow((img_clear_test[idx].astype('uint8')))
                    axs[idx, 1].axis('off')
                    axs[idx, 1].set_title('origin')

                    axs[idx, 2].imshow(generated_images[idx].astype('uint8'))
                    axs[idx, 2].axis('off')
                    axs[idx, 2].set_title('dehazed')
                fig.savefig("./dehazed_result/image/dehazed/%d-%d.jpg" % (epoch, index))

        now = datetime.datetime.now()
        print(np.mean(d_losses), np.mean(d_on_g_losses), 'spend time %s' % (now - start))
        # 保存所有权重
        save_all_weights(d, g, epoch, int(np.mean(d_on_g_losses)))
```

和MSCNN一样，代码中还有很多需要注意的地方，可自行参考文末的开源链接。

### 5.5 参数设置

不懂这些参数的，可自行参考代码。

- epochs = 50
- batch size = 2
- critic_updates = 4
- Adam 优化器

在GTX1060 的显卡上跑了8个小时。

## 6. 实现效果展示

![](https://ae01.alicdn.com/kf/HTB1W27JdoGF3KVjSZFvq6z_nXXaX.jpg)

![](https://ae01.alicdn.com/kf/HTB1wuIHdgmH3KVjSZKzq6z2OXXa7.jpg)

![](https://ae01.alicdn.com/kf/HTB1UXoQdlCw3KVjSZR0q6zcUpXaO.jpg)

除此之外，我还做了一个Android App，电脑作为服务器，Android端上传雾图进行去雾，服务处理后回馈，Android显示。视频Demo：

<iframe id="spkj" src="//player.bilibili.com/player.html?aid=55396421&cid=96861817&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" width=100%> </iframe>
!!!
<script type="text/javascript">  
document.getElementById("spkj").style.height=document.getElementById("spkj").scrollWidth*0.76+"px";
</script>
!!!


## 7. 结语

虽然是copy的Deblur结构，但是自己实现出来的时候还是非常高兴的。

所有源代码地址:https://github.com/raven-dehaze-work/DeblurGanToDehaze


