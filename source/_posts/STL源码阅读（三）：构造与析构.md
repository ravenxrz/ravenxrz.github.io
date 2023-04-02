---
title: STL源码阅读（三）： 构造与析构
categories: stl
date: 2023-03-29 15:34:31
tags:
---

这是stl源码阅读系列的第三篇，这一篇来看看stl中对象的构造与析构是如何处理的。

<!--more-->

## 1. placement new

通常来说new操作分为两步：

1. 分配内存buffer
2. 在buffer上构造对象（调用构造函数）

其中第一步称为`operator new`， 第二步称为`placement new`。两者的使用例子如下：

```cpp
#include <iostream>

class MyClass {
public:
    MyClass() {
        std::cout << "MyClass constructor called!" << std::endl;
    }

    ~MyClass() {
        std::cout << "MyClass destructor called!" << std::endl;
    }
};

int main() {
    // 使用 new 操作符来分配内存并调用 MyClass 的构造函数
    MyClass *p1 = new MyClass();
    delete p1;

    // 使用 placement new 语法在已分配的内存上构造 MyClass 对象，但不调用构造函数
    void *memory = ::operator new(sizeof(MyClass));  // 分配内存
    MyClass *p2 = new (memory) MyClass();            // 在已分配的内存上构造 MyClass 对象
    p2->~MyClass();                                 // 手动调用析构函数
    ::operator delete(memory);                      // 释放内存
    return 0;
}
```

了解这种语法后，回到主题，系列第一篇已经介绍了stl默认分配器的工作原理，既然已经有了需要的内存，stl是如何在这些内存上进行构造和析构的呢？

## 2. stl_construct.h

关于stl的构造和析构，源码文件在 `allocator/stl_construct.h`中。

### 1. construct

先看 `construct` 接口：

```cpp
// construct() 接受一个指针 __p 和 一个初值 __value。
template <class _T1, class _T2>
inline void construct(_T1* __p, const _T2& __value) {
  _Construct(__p, __value);
}

template <class _T1>
inline void construct(_T1* __p) {
  _Construct(__p);
}
```

两个函数都接受 `__p` 参数，即要在`__p`所指内存上构造对象。第一个`construct`还允许赋一个初值。

> 如果是多参数构造器，怎么实现？是因为老版本stl所采用的c++版本还没有变参模板么？

`_Construct`函数如下:

```cpp
// 将初值 __value 设定到指针所指的空间上。
template <class _T1, class _T2>
inline void _Construct(_T1* __p, const _T2& __value) {
  new ((void*) __p) _T1(__value);   // placement new，调用 _T1::_T1(__value);
}

template <class _T1>
inline void _Construct(_T1* __p) {
  new ((void*) __p) _T1();
}
```

这里就用到了 `placement new`技术，很简单，不再赘述。

### 2. destroy

再来看看destroy:

```cpp
template <class _Tp>
inline void destroy(_Tp* __pointer) {
  _Destroy(__pointer);
}

// 第二个版本，接受 __first 和 __last 两个迭代器，将 [__first, __last)范围内的所有对象析构掉。
template <class _ForwardIterator>
inline void destroy(_ForwardIterator __first, _ForwardIterator __last) {
  _Destroy(__first, __last);
}
```

**第一个版本`_Destroy`:**

```cpp
// 第一个版本，接受一个指针，准备将该指针所指之物析构掉。
template <class _Tp>
inline void _Destroy(_Tp* __pointer) {
  __pointer->~_Tp();
}
```

直接调用了`_Tp`类型的析构函数。

> 如果是builtin的类型，无对应的析构函数，也未发现特例化版本，是否不应该直接调用第一个版本的`destroy`
>
> TODO: 需要在后文分析容器时，查看`destroy`的具体用法

**第二版本 `_Destroy`:**

```cpp
// 调用 __VALUE_TYPE() 获得迭代器所指对象的类别。
template <class _ForwardIterator>
inline void _Destroy(_ForwardIterator __first, _ForwardIterator __last) {
  __destroy(__first, __last, __VALUE_TYPE(__first));
}
```

这里的 `__VALUE_TYPE`宏是关键，看下它的定义:

```cpp
#define __VALUE_TYPE(__i)        __value_type(__i)

// 决定某个迭代器的类型-value_type
template <class _Iter>
inline typename iterator_traits<_Iter>::value_type*
__value_type(const _Iter&)
{
  return static_cast<typename iterator_traits<_Iter>::value_type*>(0);
}
```

这就用到本系列二介绍的技术--traits技术。从结果来看， `__VALUE_TYPE` 返回的是迭代器所指向类型的一个指针。

再看 `__destroy` 是如何实现的：

```cpp
// 利用__type_traits<T>判断该类型的析构函数是否需要做什么。
template <class _ForwardIterator, class _Tp>
inline void 
__destroy(_ForwardIterator __first, _ForwardIterator __last, _Tp*)
{
  typedef typename __type_traits<_Tp>::has_trivial_destructor
          _Trivial_destructor;
  __destroy_aux(__first, __last, _Trivial_destructor());
}
```

这里又用到了内置类型声明技术，实际上是去看 `_Tp`的 `has_trivial_destructor`是什么类型, 看下 `__type_traits`类：

```cpp
template <class _Tp>
struct __type_traits { 
   typedef __true_type     this_dummy_member_must_be_first;
   typedef __false_type    has_trivial_default_constructor;
   typedef __false_type    has_trivial_copy_constructor;
   typedef __false_type    has_trivial_assignment_operator;
   typedef __false_type    has_trivial_destructor;
   typedef __false_type    is_POD_type;
};

// class 没有任何成员，不会带来额外负担，却又能够标示真假。
struct __true_type {
};

struct __false_type {
};

```

可以看到默认 `__type_traits`模板中的 `has_trivial_destructor`为 `__false_type`。 `__true_type`和`__false_type`为两个空成员的结构体类型，用于表示`_Tp`是否是`has_trivial_destructor`, 如果是 `__true_type`则不用调用`_Tp`对象的析构函数（比如内置类型），否则则调用析构函数。

让我们返回到`__destroy`函数的`__destroy_aux`:

```cpp
// 若是 __false_type，这才以循环的方式遍历整个范围，并在循环中每经历一个对象就调用第一个版本的 destory()。
// 这是 non-trivial destructor
template <class _ForwardIterator>
void
__destroy_aux(_ForwardIterator __first, _ForwardIterator __last, __false_type)
{
  for ( ; __first != __last; ++__first)
    destroy(&*__first);
}

// 若是 __true_type，什么都不做；这是 trivial destructor。
template <class _ForwardIterator> 
inline void __destroy_aux(_ForwardIterator, _ForwardIterator, __true_type) {}
```

可以看到，实现符合上文所说，`__false_type`循环调用迭代器所指对象的析构函数（前文说了destroy内部是调析构）。而`__true_type`是空实现。

默认的 `__type_traits` has_trivial_destructor为`__false_type`，哪些情况为 `__true_type`?

```cpp
#ifndef __STL_NO_BOOL

__STL_TEMPLATE_NULL struct __type_traits<bool> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

#endif /* __STL_NO_BOOL */

__STL_TEMPLATE_NULL struct __type_traits<char> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<signed char> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<unsigned char> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

#ifdef __STL_HAS_WCHAR_T

__STL_TEMPLATE_NULL struct __type_traits<wchar_t> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

#endif /* __STL_HAS_WCHAR_T */

__STL_TEMPLATE_NULL struct __type_traits<short> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<unsigned short> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<int> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<unsigned int> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<long> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<unsigned long> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

#ifdef __STL_LONG_LONG

__STL_TEMPLATE_NULL struct __type_traits<long long> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<unsigned long long> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

#endif /* __STL_LONG_LONG */

__STL_TEMPLATE_NULL struct __type_traits<float> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<double> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

__STL_TEMPLATE_NULL struct __type_traits<long double> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

#ifdef __STL_CLASS_PARTIAL_SPECIALIZATION

template <class _Tp>
struct __type_traits<_Tp*> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};

#else /* __STL_CLASS_PARTIAL_SPECIALIZATION */


#endif /* __STL_CLASS_PARTIAL_SPECIALIZATION */
```

**即所有的built-in类型和指针类型均不为 `__true_type`, 符合常识，因为这些类型没有析构函数。**

**其他特化版本_Destroy**

```cpp
// 第二版本，destroy 泛型特化， 实际上不做如下的特例化版本也是
// 可以正常处理的，但是会带来额外开销
inline void _Destroy(char*, char*) {}
inline void _Destroy(int*, int*) {}
inline void _Destroy(long*, long*) {}
inline void _Destroy(float*, float*) {}
inline void _Destroy(double*, double*) {}
#ifdef __STL_HAS_WCHAR_T
inline void _Destroy(wchar_t*, wchar_t*) {}
#endif /* __STL_HAS_WCHAR_T */

```

这些常见内置类型做了特化处理，我并未找到合适的理由，为什么要写这些特例，因为即使不写特例化版本也可以跑过，问了下`chatgpt`，回答是因为考虑了额外开销。所以暂时也认可这个理由。

## 3. 总结

本篇从`placement new`出发，引出了stl的构造函数如何实现，之后详细介绍了 `destroy` 函数族的实现，其中第二个版本的`destroy`最为复杂，利用了前文提到的`traits`技术。

## 补充 - new操作符

除了 placement new 之外，C++ 中还有多种 new 的方式，其中比较常用的包括：

1. 普通的 new，用于在堆上分配内存，并调用构造函数初始化对象。
2. new[]，用于在堆上分配一段连续的内存，并调用构造函数初始化对象数组。
3. operator new，用于在堆上分配内存，但不调用构造函数。
4. operator new[]，用于在堆上分配一段连续的内存，但不调用构造函数。
5. nothrow new，用于在堆上分配内存，如果分配失败则返回空指针而不是抛出异常。
6. Placement new，用于在已有的内存上调用构造函数来初始化对象。

需要注意的是，对于 new 和 new[] 分配的内存，需要使用对应的 delete 和 delete[] 释放内存。而对于 placement new，由于没有进行内存分配，因此不需要使用 delete 或 delete[] 释放内存，而是需要显式调用析构函数来销毁对象。
