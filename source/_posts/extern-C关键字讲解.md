---
title: extern "C"关键字讲解
date: 2021-01-06 09:42:20
categories: 编程语言
tags:
---

偶尔在看别人的代码时，会遇到 `extern “C” `的关键字，一般来说有两种场景：

case 1:

```c
#ifdef __cplusplus
extern "C" {
#endif

// all of your legacy C code here

#ifdef __cplusplus
}
#endif
```

或者说：

```c
extern "C" {
#include "legacy_C_header.h"
}
```

两种写法均有，*个人觉得第一种更好，没有将extern “C”的职责转移到client，是一种更好的设计。*

由于自己写代码时就没用过，所以之前没去理解这种用法的含义，这里查漏补缺，记录一下。

<!--more-->

学过C和C++的同学都知道，C是不支持重载的(overload)，而C++比C多了很多高级语法，重载、类、命名空间、模板等等。以重载为例，C++是如何区分相同名字的函数的呢？C++使用了一种叫 **name mangling**的技术，在编译的过程中，为函数名添加一些额外的字符串。而C编译时就是函数名本身。 我们来验证一下：

编写main.cpp文件：

```c++
void g(){

}

void k(){

}

int main()
{
    return 0;
}
```

执行编译：

```shell
 g++ -c -o maincpp.o main.cpp
```

在编写main.c文件：

```c
void g(){

}

void k(){

}

int main()
{
    return 0;
}
```

执行编译：

```c
gcc -c -o mainc.o main.c
```

有了目标文件 `maincpp.o`和`mainc.o`，下面可以通过读取ELF格式中的符号表区来验证（如果不知道ELF是什么的同学，可自行百度，csapp课程中Linkiing章节也有提到）。

```shell
readelf -s mainc.o
```

![image-20210106100257458](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20210106100257458.png)

```shell
readelf -s maincpp.o
```

![image-20210106100350659](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20210106100350659.png)

可以清楚的看到，g++编译过程中为函数g k添加了些字符串，这些就是用于对函数做唯一性标识的原理。

现在的问题是如果我要在C++中调用C的函数，该如何处理？举个例子：

c.h

```c
//
// Created by raven on 1/6/21.
//

#ifndef TEST_C_H
#define TEST_C_H


int f(void);

#endif //TEST_C_H

```

c.c

```c
//
// Created by raven on 1/6/21.
//
#include "c.h"

int f(void){
    return 1;
}

```

main.cpp

```c++
#include <cassert>
#include "c.h"


int main()
{
    assert(f() == 1);
    return 0;
}
```

如果直接编译链接：

```shell
gcc -c -o c.o c.c
g++ -c -o main.o main.cpp
g++ -o main main.o c.o
```

则一定会报：

```c++
main.o: In function `main':
main.cpp:(.text+0x5): undefined reference to `f()'
collect2: error: ld returned 1 exit status
```

为什么找不到f(), 通过前面的 readelf 过程，你应该知道为什么了，因为g++在链接过程中，寻找的不是 `f()`, 而是 `_Z1fv`

那该怎么解决？这就是 `extern "C"`关键字的作用了，通过添加 `extern "C"`关键字，告诉g++编译器，包含在 `extern “C”`中的函数，不要使用 name mangling 机制，直接寻找函数本体名字即可。

还是上面的例子，我们对 c.h 做点小修改：

```c
//
// Created by raven on 1/6/21.
//

#ifndef TEST_C_H
#define TEST_C_H

#ifdef __cplusplus
extern "C" {
#endif

int f(void);

#ifdef __cplusplus
}
#endif

#endif //TEST_C_H

```

编译链接：

```shell
gcc -c -o c.o c.c
g++ -c -o main.o main.cpp
g++ -o main main.o c.o
```

