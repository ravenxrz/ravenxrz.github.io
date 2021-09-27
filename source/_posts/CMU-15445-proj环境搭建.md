---
title: CMU-15445-proj环境搭建&proj0
date: 2021-09-27 14:50:31
categories: CMU15445
tags:
---

## 0. 前言 & 踩坑记录

大半年没有更新过博文，忙着毕设，春招，实习，秋招一系列事情，一直在鸽。最近算是有点时间来充充电了。学生时代还剩下最后一年，跨专业后CS课还有相当多没补，立个flag，未来一年把cmu15445（数据库）, mit6824（分布式）, cs16c（计组）（cs16c待定，可能会换成18447）课程+lab/proj补完，6.828（操作系统）则看情况了，OS理论知识是有的，希望自己有时间能够实现一个小OS吧。

okk，闲话到此，今天简单聊聊CMU15445+proj环境搭建+proj0.

<!--more-->

CMU15445是我调研后发现的数据库最佳入门课程（其实还有一门叫CS186，不过lab需要用scala做，不想学就放弃了），15445授课教授是Andy，梗王（有兴趣的小伙伴可以去youtube上搜搜他）。所以决定跟了，截止目前，youtube上大体上能看到2018， 2019和2020版本的公开课。经过简单搜索，我发现2019版本看得是最多的，于是我也决定跟这个。额外说一下国内有个组织叫做[simviso](https://www.simtoco.com/)，他们做了这个课的人工翻译，不过是收费的，英文不太好的朋友可以考虑。

跟着课做了lab1（写sql），写了一半的proj1（BufferPool）时，发现了一个问题，写完后没办法测试。这可怎么办呢。。。于是在google了一波，发现2020版本才提供了完整的测试。所以又切到了2020的proj，做了proj0（C++ primer），但是做完后提交又会发现很多问题：

1. proj对C++编码规范要求很严，要使用`make format && make check-lint && make-clang-tidy`做格式化和检查，且所有编译warning全部视为error。

2. 现在是2021年了。。。即使是2020年的proj也有些过期，这个问题是我在clone下最新版本的bustub并做完proj0题交到autograde系统后，系统提示找不到一些函数。我感到特别纳闷，于是去了github上搜索commit历史，最终发现才发现了问题所在：

   <img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20210927150948279.png" alt="image-20210927150948279" style="zoom: 33%;" />

查看这个commit会发现，函数签名已经发生了变化，所以按照最新版做是会出现问题：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20210927151302395.png" alt="image-20210927151302395" style="zoom:50%;" />

3. 提交到gradescore上的的压缩包，应该包含路径为`src/include/primer/xxx.h`，否则会出现问题
4. 提交到gradescore后，不要手贱退出，否则很可能执行失败。

**最后，虽然开了这个坑，但是这里不会公开代码，也请大家做完labs/projs后不要公开。**

## 1. 环境搭建

1. OS选择：Linux或者WSL

> 这里又有一个坑，最初我是在WSL上开做开发的，但可能git clone下来的时候用的windows，最后造成所有拉下的源码全是CRLF的结尾符，而Linux只认LF，所以一直编译没过，如果你也遇到这个问题，推荐使用 [dos2unix](https://linux.die.net/man/1/dos2unix)来转换。

2. 包安装： clang-format, clang-tidy, cmake, g++

> 额外提示，最好自家挂个代理，为什么要挂代理？因为在cmake执行时，cmake会自动录取google-test，这个是在github上的，天朝的环境如此没办法。
>
> 挂了代理后，有两种使用代理的方法：
>
> 1. 配置`~/.gitconfig`, 比如我代理的网络端口为7890. 则修改`~/.gitconfig`:
>
>    ```
>    [http]
>    	proxy= socks5://127.0.0.1:7890
>    [https]
>    	proxy= socks5://127.0.0.1:7890
>    ```
>
> 2. 使用 proxychains, 具体使用方法，可自行百度。

3. clone bustub: `git clone https://github.com/cmu-db/bustub.git`。 checkout到2020版本时的commit：`git checkout 444765a7fcccbee8295fda5313c8e0647245ec86`

4. 进入 bustub目录：

   ```
   $ mkdir build
   $ cd build
   $ cmake ..
   $ make -j 4
   ```

## 2. 任务

proj0的任务非常简单，基本上是在确认你的c++掌握程度，比如指针，构造函数，继承等概念。具体是要求实现一个Matrix类和其相关的一些操作，很简单，这里不赘述。

## 3. 测试

proj0自带了简单测试：

```
$ mkdir build
$ cd build
$ make starter_test
$ ./test/starter_test
```

不过，完整的测试还是需要的gradescore的，可看课程[faq](https://15445.courses.cs.cmu.edu/fall2020/faq.html#q5)中所提到的.