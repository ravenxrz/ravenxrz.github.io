---
title: gdb调试详解
categories: Linux
tags: gdb
abbrlink: 37784c45
date: 2020-06-21 16:24:32
---

一直用惯了IDE集成的debug工具, 忽略了gdb这个命令行debug工具. 而最近在做csapp的bomblab, 就不得不来学习它了. 所以特此记录.

## 1. 基本使用

考虑以下我们在IDE中要进行debug一般需要哪些功能?

1. 设置断点(包含条件断点)
2. 开启debug
3. step in, step over, continue
4. 观察某些值的变化, 打印数组value, 打印某个地址value
5. 函数调用stack, 切换stack
6. 临时更改某个变量,参数的值

下面讲解如何用gdb实现这些功能。

为了更方便讲解, 这里提前把所有常用命令贴出, 读者可不用一一记住, 在逐渐使用的过程中,自然就能形成记住了.

<!-- more -->

| 命令              | 简写  | 含义                                                         |
| ----------------- | ----- | ------------------------------------------------------------ |
| list              | l     | 列出10行代码                                                 |
| break             | b     | 设置断点                                                     |
| break if          | b if  | 设置条件断点                                                 |
| delete [break id] | d     | 删除断点047(按照break id)删除,没有break id,删除所有段6       |
| disable           |       | 禁用断点                                                     |
| enable            |       | 允许断点                                                     |
| info              | i     | 显示程序状态. info b(列出断点), info regs(列出寄存器)等      |
| run [args]        | r     | 开始运行程序, 可带参数                                       |
| display           | disp  | 跟踪查看那某个变量, 每次停下来都显示其值                     |
| print             | p     | 打印内部变量值                                               |
| watch             |       | 监视变量值新旧的变化                                         |
| step              | s     | 执行下一条语句，如果该语句为函数调用，则进入函数执行第一条语句 |
| next              | n     | 执行下一条语句,如果该语句为函数调用,不会进入函数内部执行(即不会一步步地调试函数内部语句） |
| continue          | c     | 继续程序的运行，直到遇到下一个断点                           |
| finish            |       | 如果进入了某个函数，返回到调用调用它的函数，jump out         |
| set var name = v  |       | 设置变量的值                                                 |
| backtrace         | bt    | 查看函数调用信息（堆栈）                                     |
| start             | st    | 开始执行程序，在main函数中的第一条语句前停下                 |
| frame             | f     | 查看栈帧，比如 frame 1 查看1号栈帧                           |
| up                |       | 查看上一个栈帧                                               |
| down              |       | 查看那下一个栈帧                                             |
| quit              | q     | 离开gdb                                                      |
| edit              |       | 在gdb中进行编辑                                              |
| whatis            |       | 查看变量的类型                                               |
| search            |       | 搜索源文件中的文本                                           |
| file              |       | 装入需要调试的程序                                           |
| kill              | k     | 终止正在调试的程序                                           |
| layout            |       | 改变当前布局(必备命令)                                       |
| examine           | x     | 查看内存空间(必备命令)                                       |
| checkpoint        | ch    | debug快照, 需要反复调试某一段代码时,非常有用                 |
| disassemble       | disas | 反汇编                                                       |
| stepi             | si    | 下一行指令(遇到函数,进入函数)                                |
| nexti             | ni    | 下一行指令                                                   |

这么多命令, 但是不要紧, 看完一个例子, 就掌握其中大半了.

example：

例子来自: https://www.geeksforgeeks.org/gdb-command-in-linux-with-examples/

```c++
// gfg.cpp
#include <iostream> 
#include <stdlib.h> 
#include <string.h> 
using namespace std; 

int findSquare(int a) 
{ 
	return a * a; 
} 

int main(int n, char** args) 
{ 
	for (int i = 1; i < n; i++) 
	{ 
		int a = atoi(args[i]); 
		cout << findSquare(a) << endl; 
	} 
	return 0; 
} 
```

通过g++进行编译, 注意编译参数需要添加"-g":

```
g++ -g -o gfg gfg.cpp
```

执行:

```shell
gdb gfg
```

开启gdb:

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621144320382.png)

首先介绍的是break(b) 命令, 这是用来设置断点的命令, 它的使用格式如下:

```shell
b
break [function name]
break [file name]:[line number]
break [line number]
break *[address] # 这个用来调试汇编很有用
break ***any of the above arguments*** if [condition]
b ***any of the above arguments*** 
```

使用:`b main` 对main函数设置断点.

然后执行:`r 1 10 100`命令,把程序跑起来. `1 10 100`是要传入的参数

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621151305820.png)

可以看到,程序停在了Breakpoint 1这里, break point 1中的1是什么? gdb为每个断点设定了一个id. 

怎么查看当前设置了哪些断点?

```shell
info b
```

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621150915280.png)

第一列Num就是break point id.  Enb表示当前断点是enable的, 可以通过 `disable breakpoint id` disable一个断点. What字段表明了当前断点的位置.

ok, 现在我们做到了. 1. 设置断点.2. 查看断点. 3. 其中程序.

接下来我们就一步步的debug吧.

使用 `n`或者 s进行单步调试,(两者的区别在于,step遇到函数会进入函数, next不会). 

值得说明的是, 执行一条命令后, 直接按回车, 会重复执行上一条命令.

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621151435600.png)

现在应该会单步调试了吧.

使用 `bt`,可以查看函数调用堆栈:

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621152801080.png)

使用 `p a`可以打印a变量的值:

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621152828266.png)

p还可以使用格式符, 如 `p /x a`把a以hex格式打印, 对于数组,如 int arr[3]; 可以使用

```shell
p *arr@3
```

打印. @后跟的是数组长度. 



但是到现在, 有一个很严重的问题, 那就是在debug的时候 ,没办法查看源代码。

gdb当然想到了这个问题, 我们可以通过 `l`命令, 展示最近的源代码.

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621151620849.png)

l 命令默认展示10行代码, 可以通过 l [start_line] [end_line] 展示start_line – end_line之间的代码.

可是, 这还是非常难受, 比如我甚至不知道当前执行到哪儿了.

ok, 接下来介绍一个必备的命令: layout

执行 `h layout`可以查看layout的帮助:

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621151826603.png)

我们关注 LAYOUT-NAME即可.

可以看到,LAYOUT-NAME有四个选项:

- layout src. 展示源代码和命令窗口:

  ![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621151917633.png)

  这就搞定了我们在调试代码时,要查看同步查看源代码的需求. 上面展示了我们当前执行到了哪里. B+展示了我们的断点位置.

- layout asm

  反汇编布局, 可以查看对应的反汇编代码.

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621152053830.png)

这个在bomblab中肯定是要用的. 平常基本不适用,毕竟汇编用得确实不多.

- layout split

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621152229097.png)

这个就是同时展示, src和asm. 没什么好说的.

- layout regs

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621152325113.png)

展示寄存器窗口. 这个在bomblab中也是必备的. 可以分析各寄存器当前的值. **值得注意的是,有时候终端会花屏, 这时执行refresh(或Ctrl+L)命令即可**

ok,  上面都是一些基础操作. 下面按照需求,一个个讲解.

### 问题1: 如何设置条件断点?

比如在main函数中, 我们只在 a = 10时,才停下. 则可以通过 `b 16 if a == 10`命令完成.

上面代码的含义时， 在代码16行， 如果a==10，则停下，否则忽略。

### 问题2: 卡在一直长循环, 如何跳出这个循环?

看下面这个代码:

```c
1: for(int i = 0; i< 1000; i++)
2: {
3: 	// do something
4: }
5: int a = 10;
```

假设我们在第一行打了断点 `b 1`, 现在通过 `n或i`进入了for循环, 此时如何快速执行完这个循环呢? 可以在第5行打断点 `b 5`, 然后执行 `c`  continue命令, 就可以快速执行到第5行了.

### 问题3: 如何删除断点或disable断点 

其实前文已经提到了, 每个断点都有一个id, 通过 `info b`查看, 然后执行d breakpointid即可.

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621153643799.png)

### 问题4: 如何快速清除一个函数中的所有断点

使用clear命令, clear FUNCTION_NAME 即可.

### 问题5: 如何保存一个程序的快照

有时候我们在debug时, 在到达某个debug点之前, 要做很多重复的工作, 这时,我们可以在这个点上生成一个快照, 这次debug失败后, 下次直接从这个快照中继续运行. 

此时就可以用checkpoint来做.

比如上文的程序, 我可以当 a = 10时,生成一个快照, 然后下次直接从a=10启动程序.

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621154337275.png)

执行 c, run完当前进程. 会看到context自动切换到了下一个进程.

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621154515768.png)

或者手动执行 `restart checkpointid`, 手动切换.

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621154556161.png)

### 问题6: 监听某个变量, 变量发生变化时, 自动打印该变量

使用watch 命令.

比如监听i变量,只要i发生了变化, 就自动打印它.

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621154853484.png)

### 问题7: 每次停顿, 都要打印一些想要监听的变量

使用display命令.

display [var] 可以在每次程序debug中停顿时,打印你想知道的变量值.

如,我要监听 i可以:

```
display i
```

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621155215978.png)

可以看到, 每次停下, i的值都打印了出来.

### 问题8: 如何切换stack frame

有时候, 我们进入到某个函数后, 想要重新查看另一个stack frame的局部变量.比如:

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621155435491.png)

当前在findSqure stack frame中, 想要切换到main frame中去.

可以通过 `frame frameid`切换, 这里是`frame 1`切换.

如何函数调用层次过深, 可以使用frame命令,如果只是想查看两个较为临近的frame, 使用 `up num或down num`命令更合适.up代表向上走多少个frame, down则是向下.

### 问题9:更换执行程序

想要在gdb中直接加载另一个程序, 使用`file [file_path]` 命令即可.

### 问题10:打印某个内存区域中的值

这个问题在c语言中相当场景, 比如要打印数组的value, 打印某个特定内存位置的值. 都可以使用.

使用 `x`命令解决这个问题, x命令的格式如下:

```shell
x /[num][format][width] address
```

- address没什么可说的.就是你要查看的内存开始地址.
- num: 打印多少个单元
- format: 以什么格式打印, 通过有 十六进制(x), 十进制(d), 八进制(o), 字符(c). 具体可通过h x查看
- width: 一个单元的宽度, 常见单位为 byte 8bit(b), half word 16bit(h), word 32bit (w), gaint 64bit(g). 同样,可通过h x查看.

下面就用一些例子来说明吧.

```c
char buf[10] = "hello";
```

现在,要以 字符形式打印buf. 应该怎么写命令?

```shell
x /10cb buf
```

- 10: 代表10个单元
- c: 代表以字符形式打印
- b: 一个单元1个字节,(从语言中的char的长度为1)

又如:

```c
int arr[] = {1,2,3,4};
```

```shell
x /4dw arr
```

- 4: 4个单元
- d: 十进制打印
- w: 一个单元32bit

ok, 到这里基本的调试操作应该都满足了, 如果遇到什么不知道的,直接百度或者查看help吧。

## 2. 反汇编

gdb也是支持反汇编的, 这也是bomblab必备的能力.

同样以下面代码为例:

```c++
#include <iostream> 
#include <stdlib.h> 
#include <string.h> 
using namespace std; 

int findSquare(int a) 
{ 
	return a * a; 
} 

int main(int n, char** args) 
{ 
	for (int i = 1; i < n; i++) 
	{ 
		int a = atoi(args[i]); 
		cout << findSquare(a) << endl; 
	} 
	return 0; 
} 
```

演示如何反汇编.

启动gdb,  

```
b main
r 1
```

通过 disassemble命令进行反汇编.

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621161232831.png)

如果指向反汇编时,添加源代码和行号, 执行

```
disassemble /s
```

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621161438046.png)

上面的命令用来临时看看汇编还可以, 但是要跟踪还是得使用layout命令.

```
layout asm
```

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621161534374.png)

### 问题1: 为某个特定的指令地址加断点

```shell
b *address
b *(function_name + offset)
```

如:

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200621161656224.png)

ok, gdb的简单使用就到这里了.

**还有个打断点的方式是, 代码走到了指令的位置, 直接输入b, 就在当前位置打了断点.**

### 问题2： 导出汇编代码

好吧，这个我并不知道如何用gdb实现， 改用 objdump -d 命令即可实现。

## 3. 输出log

如何你想要导出gdb的输出， 那么可以采用log功能。

1. set logging [on/off] 设置log开关， 默认导出的log文件名为 gdb.txt
2. set logging file file_name. 改变log文件名
3. set logging overwrite [on/off] 默认gdb采用append方式，使用overwrite可以每次覆盖写。
4. show logging 显示当前logging设置

## 4. 多线程调试

多线程的调试一直都是难点，好在gdb对多线程的调试也有良好的支持。核心命令为 `thread`

给个最简单的测试程序：

```c++
/**
 * @file gdb 多线程调试
 */
#include <pthread.h>
#include <unistd.h>

long long a = 0;
long long b = 0;

void* func1(void *arg)
{
	while (1)
	{
		a++;
		sleep(1);
	}
}

void* func2(void *arg)
{
	while (1)
	{
		b++;
		sleep(1);
	}
}

int main()
{
	pthread_t t1, t2;
	pthread_create(&t1, NULL, func1, NULL);
	pthread_create(&t2, NULL, func2, NULL);
	pthread_join(t1, NULL);
	pthread_join(t2, NULL);
	return 0;
}
```

现在编译后并采用gdb加载后。可以使用以下一些命令：

采用 `info thread` 查看当前有哪些线程：

![](https://pic.imgdb.cn/item/6120fa244907e2d39c160aae.png)



可以看到现在启动了3个线程，线程的id分别1 2 3.  另外注意有个叫 LWP的东西。 LWP全称为 Light weight process ， 也就是线程的别称。 Frame下面描述的时对应线程目前所处的函数栈。线程id前的\*代表的时当前正在debug的线程。 **注意当一个线程停止下来的时候，所有线程也会停止。不过当我们采用诸如 `next, finish, continue`指令调试一个线程时，其余线程也会同时运行。**

采用 `thread [id]` 切换线程。 如 `thread 2`，将会切换到线程2. 

通过`thread name xxx`为当前线程设置名字。 

通过 `thread find xxx` 找到某个线程，支持正则表达式。

通过 `thread apply [ID][all]  COMMAND` 来为某个或者所有线程，执行某个命令。如，针对上述程序，可以采用 `thread apply all bt`查看所有线程的调用栈。

![](https://pic.imgdb.cn/item/6120fc004907e2d39c19dcf7.jpg)

或者针对某个线程watch. `thread apply 2 watch a > 10` 

或者break: `break xxx thread 2` 来设置断点。

最后， 有时候想要停止其他线程，只对当前线程进行调试， 可以采用 `set shcduler-locking on` 更多的参数，可以看下面的解释。

> ```
> set scheduler-locking mode
> ```
>
> Set the scheduler locking mode. If it is `off`, then there is no locking and any thread may run at any time. If `on`, then only the current thread may run when the inferior is resumed. The `step` mode optimizes for single-stepping. It stops other threads from "seizing the prompt" by preempting the current thread while you are stepping. Other threads will only rarely (or never) get a chance to run when you step. They are more likely to run when you `next' over a function call, and they are completely free to run when you use commands like `continue', `until', or `finish'. However, unless another thread hits a breakpoint during its timeslice, they will never steal the GDB prompt away from the thread that you are debugging.
>
> ```
> show scheduler-locking
> ```
>
> Display the current scheduler locking mode.

## 5. 额外推荐

cgdb: gdb的包装， 默认打开了源代码试图，而且采用了vim模式查看源代码，熟悉vim和gdb的可以试试。

gdbgui: 这个还不错, 采用browser进行调试,比只使用gdb还是好多了.

## 6. 参考

- [gdb command in Linux with examples - GeeksforGeeks](https://www.geeksforgeeks.org/gdb-command-in-linux-with-examples/)
- [gdb中x的用法_lxy的专栏-CSDN博客_gdb x](https://blog.csdn.net/baidu_24256693/article/details/47298513)
- [GDB常用命令与技巧（超好用的图形化gdbgui）_Likes的博客-CSDN博客_gdbgui](https://blog.csdn.net/songchuwang1868/article/details/86132281)---
