---
title: TexStudio语法检查插件--Language Tools安装教程
tags: Latex
abbrlink: a05d8f5e
date: 2020-01-08 17:02:38
---

今天给大家介绍如何在TexStudio中如何启用Language Tools插件。

前提,以下所有软件请都升级到最新版本。

- [TexStudio]( http://texstudio.sourceforge.net/ )
- [Language Tools]( https://blog.csdn.net/yinqingwang/article/details/54583541 ),下载后缀为.zip的文件。
- [English dictionary]( https://extensions.libreoffice.org/extensions/english-dictionaries )，后缀为oxt, 此文件下载后放置于你安装texstudio路径下的dictionaries路径下，如我的是`D:\texstudio\dictionaries`。
- [Jdk]( https://www.oracle.com/technetwork/java/javase/downloads/index.html ),jdk安装LTS的版本就好，不一定需要最新。



## 1. Java环境安装

下载好JDK并安装后，需要配置环境变量。右键"此电脑"，然后如图操作：

![](https://pic.superbed.cn/item/5e159d7876085c32896a86de.jpg)

然后，继续新建变量，变量名和变量值如下:

```
CLASSPATH
.;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar;
```



![](https://pic.superbed.cn/item/5e159dab76085c32896a9609.jpg)

接着找到Path变量，双击新加如下内容：

```
%JAVA_HOME%\bin
```

![](https://pic.superbed.cn/item/5e159dee76085c32896a9e8a.jpg)

现在可以打开cmd输入java -version验证是否安装成功，如图:

![](https://pic.superbed.cn/item/5e159e3576085c32896aa560.jpg)

## 2. Language Tools配置

把你下载好的Language Tools（以下简称“LT"）解压到任意路径，打开cmd并进入到该路径，然后执行以下命令:

```
java -jar languagetool.jar
```

![](https://pic.superbed.cn/item/5e159ec176085c32896ac4ba.jpg)

![](https://pic.superbed.cn/item/5e159ee476085c32896ac882.jpg)

如果你的LT界面和我不一样，不用管，我只是换了各主题而已。

点击Text Checking->选项->General,选择运行服务器端口:

![](https://pic.superbed.cn/item/5e159f2e76085c32896ad3f0.jpg)

现在打开浏览器，输入"http://localhost:8081/v2/check?language=en&text=you+is+girl ,得到json说明配置成功。

![](https://pic.superbed.cn/item/5e159f8576085c32896adce0.jpg)



## 3. TexStudio配置

打开TexStudio，选项->语言检查,

然后选择导入路径，选择之前下载的后缀为oxt的文件，如图

![](https://pic.superbed.cn/item/5e15a00376085c32896ae841.jpg)

然后一路确定就行。

之后如图配置：

![1578475684632](C:\Users\a2855\AppData\Roaming\Typora\typora-user-images\1578475684632.png)

- 服务器URL：http://localhost:8081
- Java，输入java就好，网上很多说要输入全路径的，其实没必要。
- LT路径就是你下载的LT jar包路径
-  LT参数保持默认就好

然后重启TexStudio.

进行下图操作检查是否配置成功。

![](https://pic.superbed.cn/item/5e15a0f876085c32896afdd8.jpg)

![](https://pic.superbed.cn/item/5e15a12776085c32896b0338.jpg)

或者看TexStudio右下角有个LT的图标。

![](https://pic.superbed.cn/item/5e15a14776085c32896b06c0.jpg)

## 4. 检查效果

如果成功走到这里那就是配置成功了，现在来检验下效果。

![](https://pic.superbed.cn/item/5e15a1e676085c32896b23e9.jpg)

输入 You is a smart guy. 显然is应该改为are或者were。对于错误的语法，texstudio会自动标出颜色，然后点击在由underline的地方右键就可以修改了。



## 参考

[Installing Language Tool in TexStudio](https://www.bbsmax.com/A/D854kYRQJE/)

