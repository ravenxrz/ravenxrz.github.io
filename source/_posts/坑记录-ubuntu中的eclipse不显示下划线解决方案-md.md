---
title: 坑记录-ubuntu中的eclipse不显示下划线解决方案
categories: 杂文
toc: true
abbrlink: d15edb4a
date: 2019-07-29 13:58:46
tags:
---

最近在ubuntu上安装了eclipse，却发现写代码时，它不显示下划线。like：

![](https://ae01.alicdn.com/kf/Hf312184e2fca45b7a6bc89beb8678a13T.jpg)

最后发现是因为eclipse对monospace字体的显示支持不太好，更改一个字体就好了，更改方式：
<!-- more -->
![](https://ae01.alicdn.com/kf/H4432039bb95c45a2b31ed9afdeab0967Z.jpg)

个人最后使用的字体为**Ubuntu Mono**。

效果如下：

![](https://ae01.alicdn.com/kf/Hb6171a9ec53e4c95b1129ecb1b0ff330t.jpg)

### 参考

- [如何在Eclipse中再次显示下划线？](http://cn.voidcc.com/question/p-hfunjcya-cw.html)
