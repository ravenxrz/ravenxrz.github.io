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

<!--more-->

## 1. 第一级配置器 `__malloc_alloc_template`

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

## 2. 第二级配置器 `__default_alloc_template`

**总结：** 第二级配置采用 free list 的方式实现，核心目的为减少小内存分配时产生的碎片。其设计图如下：

![stl内存分配器-第二级内存分配器](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgstl%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E5%99%A8-%E7%AC%AC%E4%BA%8C%E7%BA%A7%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E5%99%A8.svg)


共16条list， 每条list负责分配固定大小的内存大小，第0条为8Bytes， 第1条为16B...最后一条为128B。

1. 分配时，若分配大小超过 **128B , 调用第一级配置器**， 否则根据需要分配的大小找到合适的list：
   1. 若该list有足够的内存空间，则直接分配一个内存单元
   2. 若该list无可用内存空间，则从内存池中分配一个chunk（一个chunk为20个该list的内存单元）到该list， 然后再走分配流程。 // 实际上还有内存池内存不足的处理，具体见下文分析
2. 释放时，若原分配大小超过 128B, 则调用第一级配置器释放内存。 否则找到对应的list，将内存归还至该list

### 1. 接口说明

第二级配置器为: `__default_alloc_template` , 以下为其部分定义：

```cpp
template <bool threads, int inst>
class __default_alloc_template {

private:
  // Really we should use static const int x = N
  // instead of enum { x = N }, but few compilers accept the former.
#if ! (defined(__SUNPRO_CC) || defined(__GNUC__))
    enum {_ALIGN = 8};  // 小型区块的上调边界
    enum {_MAX_BYTES = 128}; // 小区区块的上限
    enum {_NFREELISTS = 16}; // _MAX_BYTES/_ALIGN  free-list 的个数
# endif 
  // 将任何小额区块的内存需求量上调至 8 的倍数
  static size_t
  _S_round_up(size_t __bytes) 
    { 
      return (((__bytes) + (size_t) _ALIGN-1) & ~((size_t) _ALIGN - 1));
    }

__PRIVATE:
  // free-list 的节点结构，降低维护链表 list 带来的额外负担
  union _Obj {
        union _Obj* _M_free_list_link;  // 利用联合体特点
        char _M_client_data[1];    /* The client sees this.        */
  };
private:
# if defined(__SUNPRO_CC) || defined(__GNUC__) || defined(__HP_aCC)
    static _Obj* __STL_VOLATILE _S_free_list[]; 
        // Specifying a size results in duplicate def for 4.1
# else
    static _Obj* __STL_VOLATILE _S_free_list[_NFREELISTS];  // 维护 16 个空闲链表(free list)，初始化为0，即每个链表中都没有空闲数据块  
# endif 
  //根据申请数据块大小找到相应空闲链表的下标，n 从 0 起算
  static  size_t _S_freelist_index(size_t __bytes) {
        return (((__bytes) + (size_t)_ALIGN-1)/(size_t)_ALIGN - 1);
  }

  // Returns an object of size __n, and optionally adds to size __n free list.
  static void* _S_refill(size_t __n);
  // Allocates a chunk for nobjs of size size.  nobjs may be reduced
  // if it is inconvenient to allocate the requested number.
  static char* _S_chunk_alloc(size_t __size, int& __nobjs);

  // Chunk allocation state.
  static char* _S_start_free;  // 内存池起始位置。只在 _S_chunk_alloc() 中变化
  static char* _S_end_free;    // 内存池结束位置。只在 _S_chunk_alloc() 中变化
  static size_t _S_heap_size;

# ifdef __STL_THREADS
    static _STL_mutex_lock _S_node_allocator_lock;
# endif

    // It would be nice to use _STL_auto_lock here.  But we
    // don't need the NULL check.  And we do need a test whether
    // threads have actually been started.
    class _Lock;
    friend class _Lock;
    class _Lock {
        public:
            _Lock() { __NODE_ALLOCATOR_LOCK; }
            ~_Lock() { __NODE_ALLOCATOR_UNLOCK; }
    };

public:

  /* __n must be > 0      */
  // 申请大小为n的数据块，返回该数据块的起始地址 
  static void* allocate(size_t __n)
  {
    void* __ret = 0;

    // 如果需求区块大于 128 bytes，就转调用第一级配置
    if (__n > (size_t) _MAX_BYTES) {
      __ret = malloc_alloc::allocate(__n);
    }
    else {
      // 根据申请空间的大小寻找相应的空闲链表（16个空闲链表中的一个）
      _Obj* __STL_VOLATILE* __my_free_list
          = _S_free_list + _S_freelist_index(__n);
      // Acquire the lock here with a constructor call.
      // This ensures that it is released in exit or during stack
      // unwinding.
#     ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance;
#     endif
      _Obj* __RESTRICT __result = *__my_free_list;
      // 空闲链表没有可用数据块，就将区块大小先调整至 8 倍数边界，然后调用 _S_refill() 重新填充
      if (__result == 0)
        __ret = _S_refill(_S_round_up(__n));
      else {
        // 如果空闲链表中有空闲数据块，则取出一个，并把空闲链表的指针指向下一个数据块  
        *__my_free_list = __result -> _M_free_list_link;
        __ret = __result;
      }
    }

    return __ret;
  };

  /* __p may not be 0 */
  // 空间释放函数 deallocate()
  static void deallocate(void* __p, size_t __n)
  {
    if (__n > (size_t) _MAX_BYTES)   
      malloc_alloc::deallocate(__p, __n);   // 大于 128 bytes，就调用第一级配置器的释放
    else {
      _Obj* __STL_VOLATILE*  __my_free_list
          = _S_free_list + _S_freelist_index(__n);   // 否则将空间回收到相应空闲链表（由释放块的大小决定）中  
      _Obj* __q = (_Obj*)__p;

      // acquire lock
#       ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance;
#       endif /* _NOTHREADS */
      __q -> _M_free_list_link = *__my_free_list;   // 调整空闲链表，回收数据块
      *__my_free_list = __q;
      // lock is released here
    }
  }

  static void* reallocate(void* __p, size_t __old_sz, size_t __new_sz);

} ;
```

代码较多，一步步看：

首先定义了几个枚举值，用于指定free list的结构：

```cpp
    enum {_ALIGN = 8};  // 小型区块的上调边界
    enum {_MAX_BYTES = 128}; // 小区区块的上限
    enum {_NFREELISTS = 16}; // _MAX_BYTES/_ALIGN  free-list 的个数
```

free list定义如下：

```cpp
static _Obj* __STL_VOLATILE _S_free_list[_NFREELISTS];  // 维护 16 个空闲链表(free list)，初始化为0，即每个链表中都没有空闲数据块 
```

初始化为全零，即所有list都无可分配内存单元：

```cpp
template <bool __threads, int __inst>
typename __default_alloc_template<__threads, __inst>::_Obj* __STL_VOLATILE
__default_alloc_template<__threads, __inst> ::_S_free_list[__default_alloc_template<__threads, __inst>::_NFREELISTS] = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, };
```

既然是list， 学过数据结构的同学应该都知道，list node通常定义为:

```cpp
struct Node {
	T value;
	Node *next_node;
};
```

可以看到为了存储一个有效单元(value), 链表的每个节点都需要额外存储的next_node指针，回顾下前文所说，第二级配置器是为小内存分配设计的， 在64位程序上指针的大小就有8B，如果采用上述node定义必将浪费大量内存。来看下第二级配置器是如何做的：

```cpp
  // free-list 的节点结构，降低维护链表 list 带来的额外负担
  union _Obj {
        union _Obj* _M_free_list_link;  // 利用联合体特点
        char _M_client_data[1];    /* The client sees this.        */
  };
```

> 个人认为 _M_client_data 其实也没必要。因为`_M_free_list_link`既代表`next_node`,也代表下一个可用内存单元

**第二级配置器采用了常见的union trick 来避免多余开销。**

ok，现在介绍几个辅助函数：

由于第二级配置器只分配8的整数倍内存，所以如果client传入了一个非8整数倍请求size，首先要找到该值向上取8整数倍的值，该逻辑由 `_S_round_up`实现：

```cpp
 // 将任何小额区块的内存需求量上调至 8 的倍数
  static size_t
  _S_round_up(size_t __bytes) 
    { 
      return (((__bytes) + (size_t) _ALIGN-1) & ~((size_t) _ALIGN - 1));
    }

```

同理，给定一个请求size，要寻找恰当的list（比如传入的size=8， 则在 list#0中分配，传入size=16, 则找到lsit#1)来分配内存，过逻辑如下:

```cpp
  //根据申请数据块大小找到相应空闲链表的下标，n 从 0 起算
  static  size_t _S_freelist_index(size_t __bytes) {
        return (((__bytes) + (size_t)_ALIGN-1)/(size_t)_ALIGN - 1);
  }
```

### 2. 分配流程

下面就来看下分配的流程，流程图：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgstl_alloc%E6%B5%81%E7%A8%8B.svg" alt="stl_alloc流程" style="zoom:125%;" />

代码如下：

```cpp
  /* __n must be > 0      */
  // 申请大小为n的数据块，返回该数据块的起始地址 
  static void* allocate(size_t __n)
  {
    void* __ret = 0;

    // 如果需求区块大于 128 bytes，就转调用第一级配置
    if (__n > (size_t) _MAX_BYTES) {
      __ret = malloc_alloc::allocate(__n);
    }
    else {
      // 根据申请空间的大小寻找相应的空闲链表（16个空闲链表中的一个）
      _Obj* __STL_VOLATILE* __my_free_list
          = _S_free_list + _S_freelist_index(__n);
      // Acquire the lock here with a constructor call.
      // This ensures that it is released in exit or during stack
      // unwinding.
#     ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance;
#     endif
      _Obj* __RESTRICT __result = *__my_free_list;
      // 空闲链表没有可用数据块，就将区块大小先调整至 8 倍数边界，然后调用 _S_refill() 重新填充
      if (__result == 0)
        __ret = _S_refill(_S_round_up(__n));
      else {
        // 如果空闲链表中有空闲数据块，则取出一个，并把空闲链表的指针指向下一个数据块  
        *__my_free_list = __result -> _M_free_list_link;
        __ret = __result;
      }
    }

    return __ret;
  };
```

流程比较简单：

1. 如果请求size超过128B，则调用第一级配置器分配内存
2. 否则，找到恰当的list， 如果该list还有可用内存单元，则将该内存单元分配给client，同时从list中移除。如果该list没有可用内存单元，则调用 `S_refill(_S_round_up(__n))` 分配。

`S_refill`代码：

流程图如下：

![stl_chunk_alloc流程](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgstl_chunk_alloc%E6%B5%81%E7%A8%8B.svg)

```cpp
/* Returns an object of size __n, and optionally adds to size __n free list.*/
/* We assume that __n is properly aligned.                                */
/* We hold the allocation lock.                                         */
template <bool __threads, int __inst>
void*
__default_alloc_template<__threads, __inst>::_S_refill(size_t __n)
{
    int __nobjs = 20;
    // 调用 _S_chunk_alloc()，缺省取 20 个区块作为 free list 的新节点
    char* __chunk = _S_chunk_alloc(__n, __nobjs);
    _Obj* __STL_VOLATILE* __my_free_list;
    _Obj* __result;
    _Obj* __current_obj;
    _Obj* __next_obj;
    int __i;

    // 如果只获得一个数据块，那么这个数据块就直接分给调用者，空闲链表中不会增加新节点
    if (1 == __nobjs) return(__chunk);
    __my_free_list = _S_free_list + _S_freelist_index(__n);  // 否则根据申请数据块的大小找到相应空闲链表  

    /* Build free list in chunk */
      __result = (_Obj*)__chunk;
      *__my_free_list = __next_obj = (_Obj*)(__chunk + __n);  // 第0个数据块给调用者，地址访问即chunk~chunk + n - 1  
      for (__i = 1; ; __i++) {
        __current_obj = __next_obj;
        __next_obj = (_Obj*)((char*)__next_obj + __n);
        if (__nobjs - 1 == __i) {
            __current_obj -> _M_free_list_link = 0;
            break;
        } else {
            __current_obj -> _M_free_list_link = __next_obj;
        }
      }
    return(__result);
}
```

> 参数 `__n` 代表一个内存单元的大小

1. 调用 ` _S_chunk_alloc(__n, __nobjs)` 尝试分配20个内存单元，`__nobjs`以引用传入，因为`_S_chunk_alloc`不一定能分配到20个单元，`__nobjs`代表最终分配了多少个内存单元
2. 如果最终分配的内存单元只有1个，直接返回给client，毕竟没必要再插入到list中，然后又从list中移除。
3. 找到恰当list，将`__nobjs`个内存单元中的第一个返回给client(` __result = (_Obj*)__chunk`), 剩余的 `__next_obj - 1`个单元放置于 list 中，形成链表。

最后来看下最重要的`_S_chunk_alloc`：

```cpp
template <bool __threads, int __inst>
char*
__default_alloc_template<__threads, __inst>::_S_chunk_alloc(size_t __size, 
                                                            int& __nobjs)
{
    char* __result;
    size_t __total_bytes = __size * __nobjs;  // 需要申请空间的大小 
    size_t __bytes_left = _S_end_free - _S_start_free;  // 计算内存池剩余空间

    if (__bytes_left >= __total_bytes) {  // 内存池剩余空间完全满足申请
        __result = _S_start_free;
        _S_start_free += __total_bytes;
        return(__result);
    } else if (__bytes_left >= __size) {  // 内存池剩余空间不能满足申请，提供一个以上的区块
        __nobjs = (int)(__bytes_left/__size);
        __total_bytes = __size * __nobjs;
        __result = _S_start_free;
        _S_start_free += __total_bytes;
        return(__result);
    } else {                             // 内存池剩余空间连一个区块的大小都无法提供                      
        size_t __bytes_to_get = 
	  2 * __total_bytes + _S_round_up(_S_heap_size >> 4);
        // Try to make use of the left-over piece.
	// 内存池的剩余空间分给合适的空闲链表
        if (__bytes_left > 0) {
            _Obj* __STL_VOLATILE* __my_free_list =
                        _S_free_list + _S_freelist_index(__bytes_left);

            ((_Obj*)_S_start_free) -> _M_free_list_link = *__my_free_list;
            *__my_free_list = (_Obj*)_S_start_free;
        }
        _S_start_free = (char*)malloc(__bytes_to_get);  // 配置 heap 空间，用来补充内存池
        if (0 == _S_start_free) {  // heap 空间不足，malloc() 失败
            size_t __i;
            _Obj* __STL_VOLATILE* __my_free_list;
	    _Obj* __p;
            // Try to make do with what we have.  That can't
            // hurt.  We do not try smaller requests, since that tends
            // to result in disaster on multi-process machines.
            for (__i = __size;
                 __i <= (size_t) _MAX_BYTES;
                 __i += (size_t) _ALIGN) {
                __my_free_list = _S_free_list + _S_freelist_index(__i);
                __p = *__my_free_list;
                if (0 != __p) {
                    *__my_free_list = __p -> _M_free_list_link;
                    _S_start_free = (char*)__p;
                    _S_end_free = _S_start_free + __i;
                    return(_S_chunk_alloc(__size, __nobjs));
                    // Any leftover piece will eventually make it to the
                    // right free list.
                }
            }
	    _S_end_free = 0;	// In case of exception.
            _S_start_free = (char*)malloc_alloc::allocate(__bytes_to_get);  // 调用第一级配置器
            // This should either throw an
            // exception or remedy the situation.  Thus we assume it
            // succeeded.
        }
        _S_heap_size += __bytes_to_get;
        _S_end_free = _S_start_free + __bytes_to_get;
        return(_S_chunk_alloc(__size, __nobjs));  // 递归调用自己
    }
}
```

这里用到了两个指针： `_S_start_free`, `_S_end_free`, 两个指针指向了现在内存池（所谓内存池，不过是由malloc分配的一大片chunk罢了）的可用空间起点和终点。代码逻辑如下：

1. 如果内存池剩余空间能够满足20个内存单元（`__total_bytes`)大小，直接返回，更新`_S_start_free`
2. 否则，查看内存池剩余空间是否至少满足一个内存单元，如果满足，则尽可能将所有剩余的空间都分配出去，`__nobjs` 更新为最终分配了多少个内存单元。
3. 否则，如果内存池剩余空间连一个内存单元都无法满足:
   1. 将内存池剩余的空间交给合适的list使用，相当于清空了内存池。 （注意由于我们分配的粒度和释放的粒度都是8的倍数，所以剩余的空间肯定也是8的倍数，也就是说一定能找到恰当的list来插入）。
   2. 更新要补充内存池的内存大小为： `size_t __bytes_to_get = 2 * __total_bytes + _S_round_up(_S_heap_size >> 4);`,也就是说申请40个单元还要多一点。
   3. 调用`malloc`分配内存，如果分配成功，则内存池补充成功，递归调用`_S_chunk_alloc`完成分配；否则，从本list的右侧list开始，逐个搜索能够分配内存的list，如果找到，则内存池补充成功，递归调用`_S_chunk_alloc`完成分配。 
   4. 如果以上方式均未成功，即malloc失败，从兄弟list中分配也失败，改用 `malloc_alloc` 分配，前文我们说过`malloc_alloc`底层也是`malloc`，实际上这里应该是利用了 `malloc_alloc`的 `oom_handler`处理。默认情况下，将抛出异常。

至此分析完分配流程。

### 3. 释放流程

代码如下:

```cpp
  /* __p may not be 0 */
  // 空间释放函数 deallocate()
  static void deallocate(void* __p, size_t __n)
  {
    if (__n > (size_t) _MAX_BYTES)   
      malloc_alloc::deallocate(__p, __n);   // 大于 128 bytes，就调用第一级配置器的释放
    else {
      _Obj* __STL_VOLATILE*  __my_free_list
          = _S_free_list + _S_freelist_index(__n);   // 否则将空间回收到相应空闲链表（由释放块的大小决定）中  
      _Obj* __q = (_Obj*)__p;

      // acquire lock
#       ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance;
#       endif /* _NOTHREADS */
      __q -> _M_free_list_link = *__my_free_list;   // 调整空闲链表，回收数据块
      *__my_free_list = __q;
      // lock is released here
    }
  }
```

1. 如果释放的大小超过128B，则调用第一级配置器释放
2. 否则，会会受到恰当的list中

> 思考：看起来第二级配置器对于小于128B的分配与释放，并不会把这些内存归还给OS，长次以往，挂链将很长，占用的空间也较高。感觉是否应该设置个单链空间大小阈值？超过该阈值就可以考虑归还给OS了。



## 3. simple_alloc

除了如上配置器外，stl还封装了一个 `simple_alloc` 类：

```cpp
// 单纯地转调用，调用传递给配置器(第一级或第二级)；多一层包装，使 _Alloc 具备标准接口
template<class _Tp, class _Alloc>
class simple_alloc {

public:
    // 配置 n 个元素
    static _Tp* allocate(size_t __n)
      { return 0 == __n ? 0 : (_Tp*) _Alloc::allocate(__n * sizeof (_Tp)); }
    static _Tp* allocate(void)
      { return (_Tp*) _Alloc::allocate(sizeof (_Tp)); }
    static void deallocate(_Tp* __p, size_t __n)
      { if (0 != __n) _Alloc::deallocate(__p, __n * sizeof (_Tp)); }
    static void deallocate(_Tp* __p)
      { _Alloc::deallocate(__p, sizeof (_Tp)); }
};
```

实际上，`simple_alloc`只是做了非常浅的一层包装，使得 `_Alloc`具有标准分配接口，什么是标准分配接口？个人理解是，前面提到的第一级别和第二级别分配器，每次分配都需要精确计算到分配多少个字节，但实际上容器中使用时，分配总是以“对象”为粒度的，即每次分配的大小都是“对象”大小的倍数。`simple_alloc`对外暴露的接口即为 **要分配多少个对象**，而不是要分配多少个字节。



