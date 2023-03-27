---
title: STL源码阅读（一）： 内存分配器
categories: stl
date: 2023-03-27 20:34:31
tags:
---

接近一年未更新博文，忙着适应工作环境，当然自己也变懒了不少。感觉还是得多抽时间充实自己，本文开始分析stl源码。

本系列分析的stl 版本为 sgi v3.3。

首先分析内存分配器，因为容器需要依赖内存分配器来实现。如最常用容器的定义:

```cpp
// alloc 是 SGI STL 的空间配置器
template <class _Tp, class _Alloc = __STL_DEFAULT_ALLOCATOR(_Tp) >
class vector : protected _Vector_base<_Tp, _Alloc> 
{
  // requirements:
```

这里的 `__STL_DEFAULT_ALLOCATOR` 定义为：

```cpp
# ifndef __STL_DEFAULT_ALLOCATOR
#   ifdef __STL_USE_STD_ALLOCATORS
#     define __STL_DEFAULT_ALLOCATOR(T) allocator< T >
#   else
#     define __STL_DEFAULT_ALLOCATOR(T) alloc
#   endif
# endif
```

 `allocator` 内部实现也为 `alloc`, 所以本文分析的对象为 `alloc`。

关于 `alloc` 的实现，文件目录为 `allocator/stl_alloc.h`

关于`alloc`的定义有两种：(通过宏来开关是哪一种)：

1. 第一级配置器：

```cpp
typedef __malloc_alloc_template<0> malloc_alloc;
typedef malloc_alloc alloc;  // 令 alloc 为第一级配置器
```

此时实际类为 ` __malloc_alloc_template`

2. 第二级配置器:

```cpp
typedef __default_alloc_template<__NODE_ALLOCATOR_THREADS, 0> alloc;  // 令 alloc 为第二级配置器
```

此时实际类为 : `__default_alloc_template`

## 1. 第一级配置器

首先看 `__malloc_alloc_template`

**总结：内部实现为直接调用 `malloc` 和 `free`， 可自定义 `oom_handler`  来应对无法分配内存的情况。**

### 1. 接口说明

该类定义如下：

```c++
template <int __inst>
class __malloc_alloc_template {

private:
  
  // 以下函数将用来处理内存不足的情况
  static void* _S_oom_malloc(size_t);
  static void* _S_oom_realloc(void*, size_t);

#ifndef __STL_STATIC_TEMPLATE_MEMBER_BUG
  static void (* __malloc_alloc_oom_handler)();
#endif

public:

  // 第一级配置器直接调用 malloc()
  static void* allocate(size_t __n)
  {
    void* __result = malloc(__n);
    // 以下无法满足需求时，改用 _S_oom_malloc()
    if (0 == __result) __result = _S_oom_malloc(__n);
    return __result;
  }

  // 第一级配置器直接调用 free()
  static void deallocate(void* __p, size_t /* __n */)
  {
    free(__p);
  }
  
  // 第一级配置器直接调用 realloc()
  static void* reallocate(void* __p, size_t /* old_sz */, size_t __new_sz)
  {
    void* __result = realloc(__p, __new_sz);
    // 以下无法满足需求时，改用 _S_oom_realloc()
    if (0 == __result) __result = _S_oom_realloc(__p, __new_sz);
    return __result;
  }

  // 以下仿真 C++ 的 set_new_handler()，可以通过它指定自己的 out-of-memory handler
  // 为什么不使用 C++ new-handler 机制，因为第一级配置器并没有 ::operator new 来配置内存
  static void (* __set_malloc_handler(void (*__f)()))()
  {
    void (* __old)() = __malloc_alloc_oom_handler;
    __malloc_alloc_oom_handler = __f;
    return(__old);
  }

};
```

1. 该类虽然是一个模板类，但实际上模板参数并未使用（实际上`__inst`恒等于0）
2. `allocate` 和 `reallocate` 分配内存的函数其内部直接调用了 `malloc` 和 `realloc`
3. `deallocate`直接调用 `free`
4. 对应内存不足时的处理，将调用 `_S_oom_malloc`和 `_S_oom_realloc` 来处理

### 2. 内存不足时的处理

现在来看，如果直接调用`malloc`遇到无法分配时，将使用`_S_oom_malloc`函数处理:

```cpp
template <int __inst>
void*
__malloc_alloc_template<__inst>::_S_oom_malloc(size_t __n)
{
    void (* __my_malloc_handler)();
    void* __result;

    // 不断尝试释放、配置
    for (;;) {
        __my_malloc_handler = __malloc_alloc_oom_handler;
        if (0 == __my_malloc_handler) { __THROW_BAD_ALLOC; }
        (*__my_malloc_handler)();  // 调用处理例程，企图释放内存
        __result = malloc(__n);   // 再次尝试配置内存
        if (__result) return(__result);
    }
}
```

该函数实际上是一个死循环，不断调用 `__malloc_alloc_oom_handler`来尝试释放内存，然后再调用 `malloc`分配，直到分配到需要的内存，或`__malloc_alloc_oom_handler`为空。

`__malloc_alloc_oom_handler`是一个成员变量：

```cpp
  static void (* __malloc_alloc_oom_handler)();
  // 初始化为：
  template <int __inst>
  void (* __malloc_alloc_template<__inst>::__malloc_alloc_oom_handler)() = 0;
```

所以默认情况，采用 `_S_oom_malloc` 时，会抛出异常。

设置 `__malloc_alloc_oom_handler` 的接口为：

```cpp
  // 以下仿真 C++ 的 set_new_handler()，可以通过它指定自己的 out-of-memory handler
  // 为什么不使用 C++ new-handler 机制，因为第一级配置器并没有 ::operator new 来配置内存
  static void (* __set_malloc_handler(void (*__f)()))()
  {
    void (* __old)() = __malloc_alloc_oom_handler;
    __malloc_alloc_oom_handler = __f;
    return(__old);
  }
```

## 2. 第二级配置器

TODO: