---
title: cmake之PUBLIC|PRIVATE|INTERFACE关键字
date: 2021-03-30 20:22:19
categories: 编程语言
tags: cmake
---

近期重学CMake，为后续的开发做点准备。遇到`PUBLIC|PRIVATE和INTERFACE`三个关键字不是很理解，搜索了一些资料，最后发现这篇文章说得比较浅显易懂。

[CMake: Public VS Private VS Interface](https://leimao.github.io/blog/CMake-Public-Private-Interface/)

本文在他的基础上，加上了一些实际验证的例子。

<!--more-->

使用到这三个关键字的地方基本在：`target_include_directories` 和 `target_link_libraries`两个命令。为了方便理解，我们以一个实际的例子来先说说 `target_include_directories`。

建立文件目录如下：

![image-20210330203443446](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20210330203443446.png)

hello.h 和 hello.cpp 组成 hello lib.

main依赖hello lib。

下面看一下每个文件的定义是如何的：

hello.h:

```c++
#ifndef CMAKETEST_HELLO_H
#define CMAKETEST_HELLO_H

void print_hello();

#endif //CMAKETEST_HELLO_H
```

hello.cpp:

```c++
#include "hello.h"
#include <iostream>

void print_hello()
{
    std::cout << "hello\n";
}

```

main.cpp:

```c++
#include <iostream>
#include "hello.h"

int main()
{
    print_hello();
    return 0;
}
```

ok, 下面重点看看CMakeLists.txt

```cmake
# create a lib target
add_library(hello src/hello.cpp)
# specify header path for hello lib
target_include_directories(hello PUBLIC ${PROJECT_SOURCE_DIR}/include)

# create main driver
add_executable(main src/main.cpp)
# add link deps
target_link_libraries(main hello)
```

很简单，就是定义了hello lib target， 给hello lib指定 include dir, 定义main driver target， 给main driver target 添加链接到hello的依赖。

至于这里：`target_include_directories(hello PUBLIC ${PROJECT_SOURCE_DIR}/include)`为什么使用public。 我马上就会解释。

想一个问题，在build一个target时，target是如何绑定include dir（也就是上面的的`${PROJECT_SOURCE_DIR}/include`)的？。 

这是因为每个target都有两个用来装include dir的变量，分别叫做`INCLUDE_DIRECTORIES`和`INTERFACE_INCLUDE_DIRECTORIES`。 前者用于存放专给自己用的INCLUDE DIR, 后者则用于存放暴露给外部程序的INCLUDE DIR。你可以把他想象成两个放食物的篮子，`INCLUDE_DIRECTORIES`篮子放的东西给自己吃，`INTERFACE_INCLUDE_DIRECTORIES`篮子放的东西给别人吃。

以hello lib和main driver为例， 这里的“自己”说的是hello lib， 这里的“别人”说的是main driver. 更泛化了的说，“自己”指的是要build的库， 而“别人”指的是依赖库的程序，而所谓的食物，指的是库的 include dir。

**`PUBLIC|PRIVATE|INTERFACE`就是在指定将定义的INCLUDE DIR放置到哪个篮子，或者两个都放。**

1. PUBLIC: 将include dir放到两个篮子中，自己用，也给别人用。
2. PRIVATE: 将include dir只放到自己的篮子中。
3. INTERFACE: 将include dir只放到别人的篮子中。

ok，记住这三点。我们再来看`target_include_directories(hello PUBLIC ${PROJECT_SOURCE_DIR}/include)`的public如何解释：将`${PROJECT_SOURCE_DIR}/include`同时放到自己和别人的篮子中。

 `target_link_libraries(main hello)` 命令不仅链接了hello库。 同时还将hello lib的`INTERFACE_INCLUDE_DIRECTORIES`篮子中的东西也装到了自己的篮子（和自己向外暴露的篮子，这一点晚点解释，可忽略）。于是我们可以成功编译并运行。

如下图，clion并没有报错。

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20210330205355736.png" alt="image-20210330205355736" style="zoom:50%;" />

现在将`target_include_directories(hello PUBLIC ${PROJECT_SOURCE_DIR}/include)`的PUBLIC，改为PRIVATE.

```cmake
target_include_directories(hello PRIVATE ${PROJECT_SOURCE_DIR}/include)
```

也就相当于 hello lib中给别人吃的篮子中没有放任何东西。所以main driver找不到hello.h。也就会报错了。

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20210330205450620.png" alt="image-20210330205450620" style="zoom:50%;" />

同理如果改为`target_include_directories(hello INTERFACE ${PROJECT_SOURCE_DIR}/include)`

main.cpp是不会报错，但是hello.cpp会报错。因为自己吃的篮子中没有东西，只有给别人的篮子中才有东西。

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20210330205716314.png" alt="image-20210330205716314" style="zoom:50%;" />

> 另外，细心的读者可能会发现，在CMakeLists.txt的最后一句`target_link_libraries(main hello)`没有指定关键字，这里补充一下`target_link_libraries`默认为``PUBLIC`, 为了避免和`target_include_directories`混淆，所以没写。

理解了 `target_include_directories`中的 `PUBLIC，PRIVATE, INTERFACE`， 再来理解 `target_link_libraries`就很简单了。只用记住有两个篮子，一个装给自己用，一个装给别人用即可。

依然以一个例子来说明，目录结果如下:

![image-20210330210016843](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20210330210016843.png)

各文件内容如下：

```c++
// hello.h
#ifndef CTEST_HELLO_H
#define CTEST_HELLO_H

void print_hello();

#endif //CTEST_HELLO_H

// hello.cpp
#include "hello.h"
#include <iostream>


void print_hello()
{
    std::cout << "hello\n";
}

// world.h
#ifndef CTEST_WORLD_H
#define CTEST_WORLD_H

void print_hello_and_world();

#endif //CTEST_WORLD_H

// world.cpp
#include "world.h"
#include "hello.h"
#include <iostream>


void print_hello_and_world()
{
    print_hello();
    std::cout << "world\n";
}

// main.cpp
#include "hello.h"

int main()
{
    print_hello();
    return 0;
}

```

CMakeLists.txt:

```cmake
# create hello lib
add_library(hello src/hello.cpp)
target_include_directories(hello PUBLIC ${PROJECT_SOURCE_DIR}/include/hello)

# create world lib
add_library(world src/world.cpp)
target_include_directories(world PUBLIC ${PROJECT_SOURCE_DIR}/include/world)
target_link_libraries(world PUBLIC hello)

# create main executable file
add_executable(main src/main.cpp)
target_link_libraries(main PUBLIC world)
```

注意到所有地方都是用的``PUBLIC`，现在解释一下这几个target的篮子存放状况。

hello:

自己和暴露给别人的篮子都存放了:`${PROJECT_SOURCE_DIR}/include/hello`

world:

自己和暴露给别人的篮子都存放了: `${PROJECT_SOURCE_DIR}/include/world`以及来自hello的`${PROJECT_SOURCE_DIR}/include/hello`。 因为world依赖了hello，而hello采用public暴露include dir. 所有world dir中的两个篮子都有 来自hello的include dir.

main:

和world一样。

现在去执行：

```shell
# in build dir
cmake ..
make
```

是能成功执行的。

如果将

```cmake
target_link_libraries(world PUBLIC hello)
# 改为
target_link_libraries(world PRIVATE hello)
```

则：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20210330210755386.png" alt="image-20210330210755386" style="zoom:50%;" />

原因在于，main依赖于world，而world private依赖于hello，也就意味着，world的篮子变为了:

自己有：`${PROJECT_SOURCE_DIR}/include/world`以及来自hello的`${PROJECT_SOURCE_DIR}/include/hello`， 但是暴露给别人的只有`${PROJECT_SOURCE_DIR}/include/world`。

所以main找不到`hello.h`。