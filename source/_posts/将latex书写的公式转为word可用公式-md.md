---
title: 将latex书写的公式转为word可用公式
date: 2019-05-31 13:58:50
categories: 杂文
toc: true
tags: Latex
---

# 将latex书写的公式转为word可用公式

## 1. 前文

word公式编辑一般有3种：

1. MathType 软件，这是最常用的，但是我并不喜欢用鼠标去点上标下标等，而且MathType所插入的公式并不好看。

2. 高版本的Word自带的公式标记，如下图，这个兼容性很好，而且也很漂亮，但书写起来不方便
   ![](https://pic1.superbed.cn/item/5cfbaced451253d178d94696.png)
3. 插图，用图代表公式。

本文的方法就是利用pandoc转为情况2下的公式，和word完美兼容。
另外，如果你要写的公式不多，其实完全没必要使用这种方法，因为反而绕了路。本文的方式只适合公式特别多的情况。
<!-- more -->
## 2. 正文

### 2.1 准备材料

1. [texlive](http://tug.org/texlive/) 安装包 -- 要编写latex总得要有编译环境吧。
2. [texstudi](http://texstudio.sourceforge.net/)o -- 个人认为最好用的latex编辑器。笔者曾尝试过
   1. Atom自搭建
   2. texwork
3. [pandoc](https://www.pandoc.org/installing.html)  -- 将ltex文件转为word文档工具，当然这个工具不止可以转换到word识别文件
4. office 2016及其以上套件（2013版我没测试，2010肯定不行）

**上述材料依次安装**（除了office），否则可能造成命令缺失，无法编译运行。

### 2.2 Demo

在使用之前请把编译器设置为xelatex, 设置方式为:

![](https://pic2.superbed.cn/item/5cfbacef451253d178d946cb.png)

好，现在正式开始:

1. 建立以下两个文件：

![](https://pic.superbed.cn/item/5cfbacf0451253d178d9471d.png)

   第一个文件是latex编译所需要用到的文件。

   第二文件则是待会从tex中得到的文件。

2. 现在开始写点latex代码吧，代码如下：

   ```
   \documentclass{article}
   \usepackage{xeCJK}		% 中文支持
   \usepackage{amsmath}	% 一些高级数学宏包
   
   \begin{document}
   	% 随便写点公式
   	\[
   	a+b = c
   	\]
   	
   	\[
   	\int_{0}^{1} xdx = \frac{1}{2} 
   	\]
   	
   	% 公式中含有中文
   	\[
   	\text{张}a+\text{张}b = \text{张}c
   	\]
   \end{document}
   ```

   使用xelatex编译器编译后：

   可以得到如下:

   ![](https://pic.superbed.cn/item/5cfbacf2451253d178d94788.png)

3. 可以看到对应生成的pdf还是相当漂亮的。现在我们把它转成word可用的docx文档。打开终端:

   ![](https://pic.superbed.cn/item/5cfbacf3451253d178d947c8.png)

   

4. 执行以下命令:

   ```
   pandoc document.tex -o "formula.docx"
   ```

   

5. 现在在打开我们的"formula.docx"文档:

   ![](https://pic1.superbed.cn/item/5cfbacf5451253d178d94802.png)

生成的公式是完全和word兼容的。这个时候你就可以复制粘贴到你的文章中去了。

### 2.3 添加些改造

每次都要敲命令是不是有点难受了？而且敲命令之前还要先关闭"formula.docx"这个文件，不然会提示文件已打开无法转换。 所以我们来搞个批运行代码，linux可以直接使用makefile+make命令，mac和linux一样，采用makefile。我们要实现的功能如下:

1. 能够自动关闭已经打开的"formula.docx"文档
2. 能够自动编译刚新写的tex文档
3. 编译完成后能够自动再打开"formula.docx"文档

在和```document.tex```和```formula.docx```目录下新建一个`run.bat`文件，并用记事本打开，添加以下代码:

```
:: 生成公式脚本
:: 关闭当前已打开的word公式文档
taskkill -FI "WINDOWTITLE eq formula.docx*"	
:: 重新生成文档
pandoc document.tex -o formula.docx
:: 重新打开文档
start /b "C:\Program Files (x86)\Microsoft Office\root\Office16\WINWORD.EXE" formula.docx
```

其中需要大家的修改的地方有一处，在最后一行的

```C:\Program Files (x86)\Microsoft Office\root\Office16\WINWORD.EXE"``` 需要修改为你的word程序路径，这个路径怎么找，推荐使用everything这款软件，如下图:

![](https://pic.superbed.cn/item/5cfbad00451253d178d948f7.png)

这样就可以找到了。

### 2.4 视频Demo

我将上述操作流程（不包括安装）录了个视频发在了B站，有需要的朋友可以看一下。

<iframe id="spkj"  src="//player.bilibili.com/player.html?aid=54188330&cid=94790979&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" width=100% allowfullscreen="true"> </iframe>
!!!
<script type="text/javascript">  
document.getElementById("spkj").style.height=document.getElementById("spkj").scrollWidth*0.76+"px";
</script>
!!!


## 3. latex写公式简单说明

还是看我们在2.2节中贴过的代码:

```C
\documentclass{article}
\usepackage{xeCJK}		% 中文支持
\usepackage{amsmath}	% 一些高级数学宏包

\begin{document}
	% 随便写点公式
	\[
	a+b = c
	\]
	
	\[
	\int_{0}^{1} xdx = \frac{1}{2} 
	\]
	
	% 公式中含有中文
	\[
	\text{张}a+\text{张}b = \text{张}c
	\]
\end{document}
```

我们一行行的分析。

- `\documentclass{article}  ` -- 这是latex必备的一条指令，意思是本篇文章使用article模板，除了article以外，常用的还有report、book、ctexart等。一般来说我们只写公式的话，导入article即可。
- `\usepackage{xeCJK}` -- article模板并不支持中文，\usepackage{xxx}是导入宏包的意思，通过宏包我们可以拓展文章的功能，xeCJK宏包用于支持中文。
- `usepackage{amsmath}` -- amsmath是拓展数学宏包，一些高级的数学环境需要它的支持。
- 再看`\begin{document} -- \end{document}` ， 这就是我们文章的主体了，我们所有需要显示的内容都是在这里书写。

如果你不懂上面说明的意思，只用记住这就是模板，你所需要的就是修改`\begin{document}` 到 `\end{document}`之间的内容。

再来说说latex里面如何书写公式。

latex公式中包括：行间公式和行内公式两种。什么意思呢。看下图就知道了。

![](https://pic.superbed.cn/item/5cfbad02451253d178d94931.png)

书写行内公式的方式有两种:

![](https://pic3.superbed.cn/item/5cfbad03451253d178d94962.png)

如图:

![](https://pic.superbed.cn/item/5cfbad05451253d178d949a0.png)

数学行间公式的方式有多种，这里介绍常用的两种:

![](https://pic.superbed.cn/item/5cfbad06451253d178d949e8.png)

如图:

![](https://pic1.superbed.cn/item/5cfbad07451253d178d94a23.png)

注意一点，在数学模式中是无法写中文的，写了是不会显示出来。如：

![](https://pic.superbed.cn/item/5cfbad09451253d178d94a5d.png)

正确的做法是用\\text{}指令包围中文，如:

![](https://pic.superbed.cn/item/5cfbad0a451253d178d94a93.png)


下面介绍一些常用的数学符号指令:

- 求和 -- \\sum_{}^{}			_ 代表下表^代表上标

- 积分 -- \\int_{}^{}

- 无穷大 -- \\infty

- 小于等于 -- \\le

- 大于等于 -- \\ge

- 多个约束条件并用一个大括号括起来 :

  ```
  $$
  \left\{
               \begin{array}{lr}
               x=\dfrac{3\pi}{2}(1+2t)\cos(\dfrac{3\pi}{2}(1+2t)), &  \\
               y=s, & 0\leq s\leq L,|t|\leq1.\\
               z=\dfrac{3\pi}{2}(1+2t)\sin(\dfrac{3\pi}{2}(1+2t)), &  
               \end{array}
  \right.
  $$
  ```

  效果如图:

  ![](https://pic.superbed.cn/item/5cfbad0c451253d178d94ae5.png)

更多的公式可以从texstudio侧边栏找:

![](https://pic2.superbed.cn/item/5cfbad0d451253d178d94b21.png)


