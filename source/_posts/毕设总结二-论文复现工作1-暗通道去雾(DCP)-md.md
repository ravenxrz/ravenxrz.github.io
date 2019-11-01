---
title: 毕设总结二-论文复现工作1-暗通道去雾(DCP)
categories: 深度学习
toc: true
tags: 图像去雾
abbrlink: 81fcc536
date: 2019-07-18 12:58:56
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

上一篇文章中，咱们介绍了传统的去雾理论，在传统去雾方法中，最为出名的当属暗通道去雾了。这篇文章将介绍如何实现DCP。

参考文章：[《Single Image Haze Removal Using Dark Channel Prior》一文中图像去雾算法的原理、实现、效果（速度可实时）](https://www.cnblogs.com/Imageshop/p/3281703.html)

本篇中的所有代码均采用Matlab实现，所有代码均已开源，地址将放于文末。

## 2. DCP

### 2.1 暗通道先验

要实现DCP去雾，首先应该知道什么是暗通道先验。

i> 在绝大多数非天空的局部区域里，某一些像素总会有至少一个颜色通道具有很低的值。换言之，该区域光强度的最小值是个很小的数。

我们给暗通道一个数学定义，对于任意的输入图像J，其暗通道可以用下式表达：
$$
J^{dark}(x) = \\\\min \\\\limits_{y\\\\in \\Omega(x)} (\\min \\limits_{c\\in \\{r,g,b\\}}J^c(y))		\\quad \\quad (1)
$$
其中：

- $J^{c}$表示彩图的每个通道图，$\\Omega(x)$表示以像素x为中心的一个窗口。

如何用代码实现对一个原图求其暗通道图呢？可参考如下代码：

```matlab
% 定义去雾参数
kenlRatio = .01;              % 最小化滤波时用的窗口占整幅图片的比例
img = haze_img;             % 原图

% 获取图像宽高
sz=size(img);
w=sz(2);
h=sz(1);

% 获取暗通道
% 暗通道第一步：获取最小化RGB分量得到的通道图 
dc = zeros(h,w);
for y=1:h
    for x=1:w
        dc(y,x) = min(img(y,x,:));
    end
end

% 暗通道第二步：最小化窗口滤波
% 定义滤波窗口
krnlsz = floor(max([3, w*kenlRatio, h*kenlRatio]));
dc2 = minfilt2(dc, [krnlsz,krnlsz]);
dc2(h,w)=0;     % 滤波后，最后一个图像单位没有了，这里手动补齐 
```

i> **暗通道先验的理论指出：**
$$
J^{dark} -> 0 \\quad \\quad (2)
$$
 实际生活中造成暗原色中低通道值主要有三个因素：

1. 汽车、建筑物和城市中玻璃窗户的阴影，或者是树叶、树与岩石等自然景观的投影；
2. 色彩鲜艳的物体或表面，在RGB的三个通道中有些通道的值很低（比如绿色的草地／树／植物，红色或黄色的花朵／叶子，或者蓝色的水面）；
3. 颜色较暗的物体或者表面，例如灰暗色的树干和石头。总之，自然景物中到处都是阴影或者彩色，这些景物的图像的暗原色总是很灰暗的。

###     2.2 基于暗通道的去雾推理

在系列二的文章中，我们曾提到过大气散射模型的简化公式:
$$
I(x) = J(x)t(x) + A(1-t(x))		\\quad\\quad (3)
$$
其中，

- $x $表示图片中的索引位置
- $I$ 表示最终的雾图
- $J$表示无雾图
- $t$表示透射图(这个翻译并不好，英文中为transmission map),每一个$t(x)$表示的是透射率，即光束经过各种散射杂质所未被散射掉的比率。所有$t(x)$组成了$t$。
- $A$全局大气光

两边同除A，可得：
$$
\\frac{I^c(x)}{A^c} = t(x) \\frac{J^c(x)}{A^c} + 1-t(x) \\quad \\quad(4)
$$
​    如上所述，上标C表示R/G/B三个通道的意思。    首先假设在每一个窗口内透射率t(x)为常数，定义他为$\\hat{t}$，并且A值已经给定，然后对式（7）两边求两次最小值运算，得到下式：
$$
\\min\\limits_{y\\in\\Omega(x)}(\\min\\limits_c\\frac{I^c(x)}{A^c}) = \\hat{t(x) }\\min\\limits_{y\\in\\Omega(x)}\\frac{J^c(x)}{A^c} + 1-\\hat{t(x)} \\quad \\quad(5)
$$
​    上式中，J是待求的无雾的图像，根据前述的暗原色先验理论有：
$$
J^{dark}(x) = \\min\\limits_{y\\in\\Omega(x)}(\\min\\limits_cJ^c(y)) = 0 \\quad \\quad(6)
$$
因此，可推导出:
$$
\\min\\limits_{y\\in\\Omega(x)}(\\min\\limits_c\\frac{J^c(y)}{A^c}) = 0 \\quad \\quad(7)
$$


将公式(7)带回公式(5)中，可得:
$$
\\hat{t(x)} = 1- \\min\\limits_{y\\in\\Omega(x)}(\\min\\limits_c\\frac{I^c(y)}{A^c}) \\quad \\quad(8)
$$
i> 上面就是暗通道去雾的理论了，它的目标就是用于估计投射图t。

说了这么多理论，咱们来看看暗通道的实际效果吧。

**对于无雾图**，那么它应该是符合暗通道先验的，也就是说对它求暗通道图的话，暗通道图应该整体偏黑：

<img src="https://ae01.alicdn.com/kf/HTB1UoaFdkWE3KVjSZSy760ocXXay.png" width="300"/><img src="https://ae01.alicdn.com/kf/HTB1DoyFdkWE3KVjSZSy760ocXXap.png" width="300"/>

<img src="https://ae01.alicdn.com/kf/HTB15e5CdfWG3KVjSZFgq6zTspXaY.jpg" width="300"/><img src="https://ae01.alicdn.com/kf/HTB1K8iFdlGE3KVjSZFh763kaFXaz.png" width="300"/>


**对于带雾图，**它是不应该符合暗通道先验的，也就是说对它求暗通道图的话，暗通道图不会整体偏黑：

<img src="https://ae01.alicdn.com/kf/HTB1x3WGdlaE3KVjSZLe760sSFXah.png" width="300"/><img src="https://ae01.alicdn.com/kf/HTB1HUeCdfWG3KVjSZPc762kbXXa8.png" width="300"/>

<img src="https://ae01.alicdn.com/kf/HTB1K1qFdouF3KVjSZK9762VtXXa1.png" width="300"/><img src="https://ae01.alicdn.com/kf/HTB1vLTQavBj_uVjSZFp7630SXXaq.png" width="300"/>


假设全局大气光A已知（实际上我在系列二中也给出了A的估计算法），利用暗通道先验我们也可以求得透射图t，那么就可以利用系列二中提到的去雾公式进行去雾了:
$$
J(x) = \\frac{I(x)-A}{\\max (t(x),t_0)} + A		\\quad\\quad (9)
$$
A的估计算法：

1. 从暗通道图中按照亮度的大小取前0.1%的像素。
2. 在这些位置中，在原始有雾图像I中寻找对应的具有最高亮度的点的值，作为A值。

### 2.3 暗通道去雾效果展示及优化

来看看运用这个模型的去雾效果：

<img src="https://ae01.alicdn.com/kf/HTB1dPODdoCF3KVjSZJnq6znHFXat.jpg" width="300"/><img src="https://ae01.alicdn.com/kf/HTB1Y_iMdlCw3KVjSZFu763AOpXaG.png" width="300"/>

<img src="https://ae01.alicdn.com/kf/HTB1K1qFdouF3KVjSZK9762VtXXa1.png" width="300"/><img src="https://ae01.alicdn.com/kf/HTB19c6RavBj_uVjSZFp7630SXXaX.png" width="300"/>


可以看到，其实去雾效果还不错，但是意到第一幅图的原图两个字的周围明显有一块不协调的地方，而第二图顶部水平方向似乎有一块没有进行去雾处理，这些都是由于我们的透射率图过于粗糙了。

后来提出了**导向滤波**的方法，可以获得更好的透射图。

未使用导向滤波与使用导向滤波的去雾结果及透射图对比图：

<img src="https://ae01.alicdn.com/kf/HTB19c6RavBj_uVjSZFp7630SXXaX.png" width="300"/><img src="https://ae01.alicdn.com/kf/HTB1e3DRavBj_uVjSZFp7630SXXaV.png" width="300"/>

<img src="https://ae01.alicdn.com/kf/HTB1pa5EdoGF3KVjSZFo762mpFXad.png" width="300"/><img src="https://ae01.alicdn.com/kf/HTB1RtKHdkWE3KVjSZSy760ocXXah.png" width="300"/>



加入导向滤波后的去雾效果：

<img src="https://ae01.alicdn.com/kf/HTB1hLuNda1s3KVjSZFA760_ZXXaG.png" width="300"/><img src="https://ae01.alicdn.com/kf/HTB1l9GDdhiH3KVjSZPf760BiVXa6.png" width="300"/>



<img src="https://ae01.alicdn.com/kf/HTB1BtSDdf1H3KVjSZFH762KppXaa.png" width="300"/><img src="https://ae01.alicdn.com/kf/HTB13k9rb7xz61VjSZFr760eLFXaP.png" width="300"/>


## 3. 完整代码及分析

完整代码：

```matlab
function y = dehaze(haze_img)
% 去雾函数: 根据输入雾图矩阵进行去雾，返回去雾后的图片矩阵
%   haze_img: 雾图矩阵
%   y:         去雾后的矩阵

% 定义去雾参数
mint = 0.05;     % 限定最小透射率
maxA = 240;     % 限定A的最大值
kenlRatio = .01;                % 最小化滤波时用的窗口占整幅图片的比例
img = haze_img;             % 原图

% 获取图像宽高
sz=size(img);
w=sz(2);
h=sz(1);

% 获取暗通道
% 暗通道第一步：获取最小化RGB分量得到的通道图 
dc = zeros(h,w);
for y=1:h
    for x=1:w
        dc(y,x) = min(img(y,x,:));
    end
end

% 暗通道第二步：最小化窗口滤波
% 定义滤波窗口
krnlsz = floor(max([3, w*kenlRatio, h*kenlRatio]));
dc2 = minfilt2(dc, [krnlsz,krnlsz]);
dc2(h,w)=0;     % 滤波后，最后一个图像单位没有了，这里手动补齐 

% 估计大气光
A = min(maxA,get_airlight(haze_img,dc2));
% 得到透射图
t = 1 - dc2./A;
t_d=double(t);
img_d = double(img);

% 加入导向滤波，得到更精细的透射图
r = krnlsz*4;
eps = 10^-6;
filtered = guidedfilter(double(rgb2gray(img))/255, t_d, r, eps);
t_d = filtered;

% 根据新的透射图，得到去雾图
t_d(t_d < mint) = mint;
J(:,:,1) = (img_d(:,:,1) - (1-t_d)*A)./t_d;
J(:,:,2) = (img_d(:,:,2) - (1-t_d)*A)./t_d;
J(:,:,3) = (img_d(:,:,3) - (1-t_d)*A)./t_d;

y = uint8(J);

% 显示对比图
subplot(1,2,1);
imshow(haze_img);
subplot(1,2,2)
imshow(y);

end

function A = get_airlight(hazeimg,darkchannel)
% 根据雾图及对应暗通道图获取大气光
% hazeimg 雾图
% darkchannel hazeimg的暗通道图
ratio = 0.01;       % 取暗通道大小的前0.1% 
% 求解前0.1%的暗通道索引位置
sorted_dc = sort(darkchannel(:),'descend');
[h,w] = size(sorted_dc);
tmp_idx = round(h*w*ratio);
tmp_idx_value = sorted_dc(tmp_idx);
idx = darkchannel >= tmp_idx_value;
% 寻找这些位置上最亮的点最为A
A = double(max(hazeimg(idx)));
end
```

其实代码中写的很清楚了，但是为方便理解，在简单说下流程吧。

第一步，求解雾图的暗通道图。

第二步，根据前文提到的算法求解大气光。

第三步，应用公式（8）求解透射图t。

第四步，导向滤波，得到更精细的透射图。

第五步，应用公式(9)去雾。

## 参考

1. [《Single Image Haze Removal Using Dark Channel Prior》一文中图像去雾算法的原理、实现、效果（速度可实时）](https://www.cnblogs.com/Imageshop/p/3281703.html)
2. [guided Filter--引导滤波算法原理及实现](https://blog.csdn.net/piaoxuezhong/article/details/78372787)

所有源代码地址：https://github.com/raven-dehaze-work/DCP-Dehaze


