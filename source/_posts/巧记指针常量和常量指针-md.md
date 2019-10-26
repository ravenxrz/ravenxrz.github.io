---
title: 巧记指针常量和常量指针
date: 2019-09-17 13:58:51
categories: 编程语言
toc: true
tags:
thumbnail: /thumbnails/flower.jpg
---

本文中，你可以学习到：

- 什么是指针常量和常量指针
- 如何巧记两者的区别

9月，决定以后走c++路线，于是买了c++ primer再看，看到指针常量和常量指针的概念觉得蛮绕的，于是搜了一下巧记两者的方式，特此记录。
<!-- more -->
<!-- more -->
## 0. 问题提出

先提出问题吧，问问自己分得清下列几种情况吗？

```c++
int val = 10;
const int *p1 = &val;
int const *p2 = &val;
int *const p3 = &val;
const int *const p4 = &val;
int const *const p5 = &val;
```

OK，晕了吗？现在让我们来区别它们吧。

## 1. 指针常量和常量指针

### 1.1 指针常量

其实叫指针常量是略有不准确的，准确的说法是“指向常量的指针”。**它不能改变其所指对象的值，但要想存放常量对象的地址，只能使用指向常量的指针。**

eg:

```c++
const int val = 10;
int val2 = 10;
int *p1 = &val;	// error! val是常量，普通指针这样指向是错误的
const int *p2 = &val; //correct
*p2 = 0;	// error! p2 指向的是常量，自然也就无法更改对象的值咯
p2 = &val2;	//correct p2 本身还是可以指向另一个对象
```

### 1.2 常量指针

指针是一种对象，同其它对象一样，允许把指针本身定为常量。指针常量的特点在于，**一旦完成指针的初始化，那它的值（也就是指针存放的地址）将无法再更改，换言之这类指针无法再指向其它对象。**

eg：

```c++
int val = 0;
int val2 = 0;
int *const p1 = &val;
*p1 = 10;	// correct 常量指针是可以通过 * 解引用来赋值的
p1 = &val2; // error! 指针常量完成初始化后就无法再指指向其它对象了
```

### 1.3 两者区别

常量指针和指针常量是互斥的，可以由下表表示

|          | 是否可更改存放的地址值 | 是否可更改指向对象的值 |
| :------: | :--------------------: | :--------------------: |
| 指针常量 |          Yes           |           No           |
| 常量指针 |           No           |          Yes           |

如果一个指针既是指向常量的指针，本身也是个常量指针怎么办？

eg:

```c++
int val = 0;
const int *const p = &val;
```

**那它既不可以更改存放的地址值，也不可以更改指向的对象的值。**

## 2. 如何巧记？

看完两者的介绍，难免容易搞混，const一会儿放这儿，一会放那儿的，于是给一个巧记的方法：

C++ Primer给出了一条法则：

> 当在阅读一个复合型的变量时，可以把一条语句从右往左来阅读来确定变量的类型。

再结合参考[2]，我们就可以把一条语句展开成英文，进而辅助我们判断。举几个例子：

```c++
const int *p1 = &val;
```

我们从右往左展开，

p1 is a pointer  (int type) which point to int const

也就是说p1是一个指针，它指向了一个整数常量。

又如:

```c++
int const *p1 = &val;
```

p1 is a pointer which point to const int

p1是一个指针，指向了一个整数常量，也就是和上面的类型相同，注意到const的位置是可以在int左右交换的。

上面两个例子都是“指针常量”-- 指向常量的指针。

再来，

```c++
int *const p1 = &val;
```

p1 is a const point which point to int.

p1是一个常量指针，指向了整数类型。

最后：

```c++
const int *const p1 = &val;
int const *const p2 = &val;
```

p1 is a const pointer which point to int const

p1 is a const pointer which point to const int

两个const，即指向常量的常量指针！

## 3. Answer

现在可以对文初提出的问题做一个解答了：

```c++
int val = 10;
const int *p1 = &val;	// p1 is a pointer point to int const 指针常量
int const *p2 = &val;	// p2 is a pointer point to const int 指针常量
int *const p3 = &val;	// p3 is a const pointer point to int 常量指针
const int *const p4 = &val;	// double const 
int const *const p5 = &val; // double const
```



## 4. 额外补充

如果前面3节你已经晕了，那最好就别看这一小节了。

通过前面3节的说明，相信你已经比较能分清指针常量和常量指针了。可是这里我再给你一个例子，也许会让你有点晕，不过我会慢慢来解释。

先看例子：

```c++
typedef char *pstring;
const pstring cstr = nullptr;
cosnt char *pstr = nullptr;
```

现在的问题是 cstr 和 pstr是一个类型吗？它们都是指针常量吗？

so，答案是，pstr是指针常量，而cstr是一个常量指针。验证的方法，就是直接写个程序看编译器通得过不咯。如下图：

![](https://ae01.alicdn.com/kf/H2b34a04551584b1ea2862656a42d22deo.jpg)

可以看在语句`*cstr = &val`上报错了，也验证了`cstr`不是一个指针常量。

其实大部分人都会犯一个错误，那就是将typedef所重定义的数据类型在语句中直接展开，在大部分的场景下是行得通的，但是这里不行。要理解为什么，我们首先需要理解一条声明语句是如何组成的。

i> 声明语句 = 修饰符 + 基本数据类型 + 声明符列表

- 修饰符， 对基本数据类型进行修饰，如const
- 基本数据类型， 如int float
- 声明符列表，如 * &

《C++ primer》 中只有 基于数据类型 + 声明符*，我个人觉得把修饰符单独拿出来好理解一点。

举个例子：

```c++
int val = 1;	// int 为基本数据类型 val 为声明符
int *p = &val;  // int 为基本数据类型 *val 为声明符
const int *ptr = &val; // const 为修饰符 int为基本数据类型 *ptr 为声明符
```

好，理解了一条声明语句是如何构成的，就可以分析分析我们的老朋友，指针常量：

```c++
int val = 10;
const char *ptr = &val; // const 为修饰符 char 为基本数据类型 *ptr 为声明符
```

以从右往左的原则，*ptr声明符表明 ptr变量为一个指针，而这个指针与基本类型char 相关，最后基本类型char由修饰符const限定，说明char为一个常量。

那么看看加入typedef后是怎么回事呢？

```c++
int val = 10;
typedef char *pstring;
const pstring ptr = &val; // 得把pstring看为一个整体，那么const 为修饰符, char *为基本数据类型，ptr 为声明符
```

还是从右往左看，ptr代表变量名，这个变量是一个char *的类型，而const则表明它是一个char \*型的常量指针。

i> 不能简单地对typedef进行展开，而应该把重定义的数据类型看成一个整体

## 参考

- [C++ Primer 中文版（第 5 版)](https://book.douban.com/subject/25708312/)
- [指针常量和常量指针的区别和巧记方式](https://blog.csdn.net/youyou519/article/details/82704401)
