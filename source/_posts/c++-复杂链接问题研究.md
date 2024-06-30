---
title: c++复杂链接问题研究
categories: 编程语言
date: 2024-06-30 13:58:06
tags: 链接
---


最近在给公司项目做编译优化，因为编译时间实在是太长了，导致合代码ci效率太低。本文主要是给整改中遇到的一个坑的总结。考虑以下编译case:

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630093909145.png" alt="image-20240630093909145" style="zoom:50%;" />

其中 liba, libb, libc 均可能是static lib , shared lib。不同的排列组合，可以有不同的结果。有些导致无法编译，有些可以编译但是不能运行。

<!--more-->

# TL;DR

- 一个static lib A 去link另一个lib B（不论是static还是shared的）, lib A**不会**将libB的符号重新copy 一份。如果一个binary只link A，binary会找不到B的符号。
- 一个shared lib A 去link另一个shared lib B, lib A**不会**将libB的符号重新copy 一份。**但**如果一个binary只link A，binary也能找到libB的符号，因为loader 通过A的rpath，会搜索到libB的路径， 从而加载libB。（具体见case 6)
- 一个shared lib A去link另一个static lib B, libA**会**将libB的符号重新copy一份。如果一个binary只link A，binary也能找到B的符号。
- 关于三方动态库重新初始化问题：
  - 如果依赖的两个动态库，对同一份符号都有定义，则会重复初始化。 见 case 2. 
  - 如果依赖的两个库，没有对同一份符号都有定义，则不会重复初始化。 见 case 6
- 尽可能先link 动态库再链接静态库。 这样即使如果动态库和静态库持有一些相同的符号也不会有问题，静态库的符号不会被打包到最后的二进制中。

# 前置说明

本文源码采用cmake构建，但是不使用cmake的target依赖传递。

> cmake 依赖传递： libA target_link 了 target B, 当 libC target_link 了 libA 时，会自动把 libB的依赖也带上。

# 基础源码

libc.h 和 libc.cc

```cpp
// .h
#pragma once

int base_add(int a, int b);


// .cc
#include "libc.h"

struct Test {  // 注意这里的全局变量
  Test() {
    static int i = 0;
    ++i;
    std::cout << "init i " << i << std::endl;
  }
}g_t;


int base_add(int a, int b) {
  return a + b;
}

```

liba.h, liba.cc

```cpp
// .h
#pragma once

int add2(int a, int b);

// .cc
#include "liba.h"
#include "libc.h"

int add2(int a, int b) {
  return base_add(a, b);
}
```

libb.h, libb.cc

```cpp
// .h
#pragma once

int add3(int a, int b, int c);


// .cc
#include "libb.h"
#include "libc.h"

int add3(int a, int b, int c) { return base_add(a, base_add(b, c)); }

```

main.cc

```cpp
#include "liba/liba.h"
#include "libb/libb.h"

int main() {
  std::cout << add2(1, 2) << std::endl;
  std::cout << add3(1, 2, 3) << std::endl;
  return 0;
}
```

# case 1 liba, libb是shared lib,  libc 是static lib

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630094456006.png" alt="image-20240630094456006" style="zoom:50%;" />

结果:

## 编译正常

**查看ldd和符号表情况:**

**liba/libb:**

![image-20240630094708105](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630094708105.png)

可以看到 libc 的 base_add 符号被加入到了 libb 中。

==**Shared lib  link static lib, 会把static lib重新打包进 shared_lib**==

**main:**

![image-20240630094810185](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630094810185.png)

## 运行失败

 程序输出

![image-20240630095050069](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630095050069.png)

**libc的全局变量被初始化了两次，且两个动态库的加载的全局变量地址是一个**

使用LD_DEBUG查看加载顺序:

![image-20240630095020453](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630095020453.png)

**初始化两次的原因是分别加载了libb和liba，而liba和libb都有一份libc的符号copy。**

# case 2: liba 是 shared lib，libb libc 是static lib

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630095133343.png" alt="image-20240630095133343" style="zoom:50%;" />

## 编译正常

**ldd和符号表情况**

liba：

![image-20240630095217600](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630095217600.png)

![image-20240630095221960](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630095221960.png)

可以看到包含libc的base_add， 这和case1场景是一样的:

libb:

![image-20240630095300459](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630095300459.png)

libb本身是一个static lib, 现在又link static libc， 看起来并没有重新打包libc。

==**Static lib 又link另一个static lib， 并不会重新打包。**==

main:

![image-20240630095646026](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630095646026.png)

直接link了liba，符合预期。

符号表：

![image-20240630095658943](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630095658943.png)

注意这里和 case 1的区别，case1中main是看不到 base_add 符号的。

> 个人觉得是这里由于libb缺 base_add 符号定义，作为static lib , 这个缺失进一步传递到了main中。



## 运行成功

程序输出

![image-20240630095734668](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630095734668.png)

这一次初始化来自于liba的加载：

![image-20240630095746169](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630095746169.png)



### 让main直接link libc

进一步让main直接link static libc：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630100002075.png" alt="image-20240630100002075" style="zoom:50%;" />

结果和case2一样, main依然不包含对libc的符号定义。

![image-20240630100013364](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630100013364.png)

查看 link 命令:

```shell
/usr/bin/c++  -g   CMakeFiles/main.dir/main.cc.o  -o main -Wl,-rpath,/data00/home/zhangxingrui/Projects/tmp/liba/build ../liba/build/libliba.so ../libb/build/liblibb.a ../libc/build/liblibc.a 
```

实际上命令link了liblibc.a。 个人猜想是 libliba.so 中已经有了 base_add 这些符号定义，所以跳过了 liblibc.a的link。

**交换 link 顺序**

```shell
add_executable(main main.cc)
target_link_libraries(main PUBLIC
/data00/home/zhangxingrui/Projects/tmp/libb/build/liblibb.a
/data00/home/zhangxingrui/Projects/tmp/libc/build/liblibc.a
/data00/home/zhangxingrui/Projects/tmp/liba/build/libliba.so
)

// 实际命令
/usr/bin/c++  -g   CMakeFiles/main.dir/main.cc.o  -o main -Wl,-rpath,/data00/home/zhangxingrui/Projects/tmp/liba/build ../libb/build/liblibb.a ../libc/build/liblibc.a ../liba/build/libliba.so 
```

*注意，不能交换 libb 和 libc的链接顺序*。

此时main中就包含了 base_add 等符号定义：

![image-20240630100205837](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630100205837.png)

但是这种case不能运行， 因为全局变量会被初始化两次。

# case 3: liba libb libc 全是static lib

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630100313035.png" alt="image-20240630100313035" style="zoom:50%;" />

## 无法编译

![image-20240630100338955](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630100338955.png)

查看编译命令:

```
/usr/bin/c++  -g   CMakeFiles/main.dir/main.cc.o  -o main ../liba/build/libliba.a ../libb/build/liblibb.a 
```

结合 case 2说的，static lib 在link static lib时，不会重新打包，liba 和 libb 不包含libc的符号定义，而此时没有采用add_subdirectory的方式，即没有利用cmake的依赖传递特性，所以生成的link命令没有包含 libc.a。 也就无法编译。

> 辅助验证 liba 中是否有libc的符号定义：
>
> ![image-20240630100406405](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630100406405.png)
>
> 结果： liba不包含libc的符号定义

可以在cmake中添加libc的link:

```
add_executable(main main.cc)
target_link_libraries(main PUBLIC
/data00/home/zhangxingrui/Projects/tmp/liba/build/libliba.a
/data00/home/zhangxingrui/Projects/tmp/libb/build/liblibb.a
/data00/home/zhangxingrui/Projects/tmp/libc/build/liblibc.a
)

-- 生成的link command:
/usr/bin/c++  -g   CMakeFiles/main.dir/main.cc.o  -o main ../liba/build/libliba.a ../libb/build/liblibb.a ../libc/build/liblibc.a 
```

此时编译正常。



# case 4:  liba 是static lib, libb 和libc是shared lib

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630100504116.png" alt="image-20240630100504116" style="zoom:50%;" />

## 无法编译

编译无法通过。

![image-20240630100742131](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630100742131.png)

先看 Liba:

![image-20240630100751282](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630100751282.png)

==**static lib link一个shared lib, 也不会重新打包。** **所以，static lib不论link何种lib，都不会重新打包。**==

再看 libb:

![image-20240630100807988](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630100807988.png)

==**shared lib link一个shared lib 不会重新打包。** **Shared lib只有在link static lib时，会将static lib的符号重新copy一份到shared lib中。**==

要解决编译问题也很简单，把libc给link进去就行。（或者用cmake target来引入依赖传递）。

```cmake
add_executable(main main.cc)
target_link_libraries(main PUBLIC
/data00/home/zhangxingrui/Projects/tmp/liba/build/libliba.a
/data00/home/zhangxingrui/Projects/tmp/libb/build/liblibb.so
/data00/home/zhangxingrui/Projects/tmp/libc/build/liblibc.so
)
```

![image-20240630100832080](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630100832080.png)

> 额外说明，在case 6中会说明， 如果main link libX，libX link了 libY,  main也是可以编译通过并运行的，因为loader会通过libX的rpath搜索到libY，从而加载libX。这里无法编译通过的原因，个人猜想是，liba是静态库，它在编译期间，无论如何也在找不到libc的定义，所以编译不过。

# case 5:  liba和libb 是static lib, libc是shared lib

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630101029003.png" alt="image-20240630101029003" style="zoom: 50%;" />

## 无法编译

![image-20240630100935678](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630100935678.png)

liba 和 libb都是static的，在link libc的时候，不会重新打包。所以liba和libb都没有libc的符号定义。而main也没有link libc，所以缺了libc的符号。

修复的方式同case4 ， 直接给main link libc.so 即可



# case 6 : liba, libb, libc 全是shared lib

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630101048278.png" alt="image-20240630101048278" style="zoom: 50%;" />

## 编译正常

ldd和符号表：

liba:

![image-20240630101109043](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630101109043.png)

main:

![image-20240630101119191](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630101119191.png)

main "link" 了 libc

>  **ldd显示的不是direct dependencies, ldd还会显示非direct dependencies(**因为它会用特殊模式来运行一把程序，从而找到非direct依赖）。直接依赖可以用readelf -d显示:
>
> ![image-20240630101225810](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630101225810.png)
>
> 参考：https://ioflood.com/blog/ldd-linux-command/#Digging_Deeper_Alternative_Approaches

结合前面说的，不应该编译成功才对，因为 liba 和 libb 都没有重新打包。liba 和 libb没有libc的符号定义。实际上liba和libb也确实没包含。而是**loader** 通过 liba 中的rpath（动态库搜索路径）自己找到了 libc的路径，所以上面的ldd看到了libc。 使用 LD_DEBUG=libs来验证:

![image-20240630101206925](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630101206925.png)

继续看libc的符号：

![image-20240630101245122](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630101245122.png)

**liblibc.so的base_add的符号定义没有加入liblibb.so中， 未重新打包。**

main:

![image-20240630101307402](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630101307402.png)

main没有base_add 符号的定义

## 运行正常

![image-20240630101332954](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630101332954.png)

**libc中的g_t被init一次**

LD_DEBUG 查看 动态库加载流程：

![image-20240630101353580](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630101353580.png)

可以看到，load libc后，全局变量被初始化。

# case 7: shared liba link shared libc, static libb link static libc

这也是公司的case。

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630101505183.png" alt="image-20240630101505183" style="zoom:50%;" />

## 无法编译

结果：

无法编译，因为liba libb 依赖的libc没有传递给main。

## 修复

这里的修复方式有多种：

方案一： 破除 libb-static 的依赖

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630103812330.png" alt="image-20240630103812330" style="zoom:50%;" />

方案二： 破除 libb-static 的依赖， 将 libc-static link到liba

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630103849945.png" alt="image-20240630103849945" style="zoom:50%;" />

> 相比方案一，main会少一个libc-shared的依赖，对部署来说，少一个依赖更好，线上产物更倾向static 的产物，一个性能更好，一个是不容易出现人工误替换了依赖lib。 

方案三： 破除 liba， libb对libc的依赖， 让libc直接link到main中

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630104128510.png" alt="image-20240630104128510" style="zoom:50%;" />

 **推荐的做法是大家依赖的公共库都用动态库link（即方案一），这样不会出现某个符号被定义了两次而出错。**

> 实际上真实场景，上述方案没有一个好实行（注意是可行但是不好实行），因为 liba  和 libb 作为第三方库是其他团队开发，很难直接让他们破除整个依赖。 

# case 8: shared liba link shared libc, shared libb link static libc

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630105023346.png" alt="image-20240630105023346" style="zoom:50%;" />

 根据之前说的 libb 要打包 libc， liba要把libc传递给 “main” （实际上是传递给loader）。

结果：

## 编译正常

可以直接编译.

## 运行失败

libc依然会加载两次：

![image-20240630105136924](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630105136924.png)

这里正式的改法交给读者自己思考了。介绍一种暴力改法。

## 修复

用patchelf强制改liba的elf，让它不依赖libc:
![image-20240630105243771](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630105243771.png)

这样就可以运行了：

![image-20240630105305401](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240630105305401.png)

# 三方库又依赖了不同版本的三方库？

这个玩意在我看来是解不了的。 也许这也是c++没有好的包管理器的原因之一，另一个则是ABI的问题。

## 总结

本文通过一个例子，详细介绍了不同case的复制link问题与解法。具体规则参考TL;DR一节。

