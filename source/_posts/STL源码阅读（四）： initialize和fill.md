---
title: STL源码阅读（四）： copy & fill & uninitialized
categories: stl
date: 2023-04-02 15:34:31
tags:
---

这是stl源码阅读系列的第四篇，这一篇来看看stl中的 `copy`, `fill`和`uninitialized`函数族。本篇中，可以看到第二篇曾提到过的 `iterator_category`的作用。

<!--more-->

 ## 1. copy

copy函数有两个: copy, copy_n和 copy_backward, 下面为copy和copy_backward两者的调用图:

![stl-copy-algo](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgstl-copy-algo.svg)



![stl-copyback-algo](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgstl-copyback-algo.svg)

两者调用链几乎一致，用到的trick也是一样的，故本文仅分析`copy`:

```cpp
// 完全泛化版本-copy，调用 __copy_dispatch
template <class _InputIter, class _OutputIter>
inline _OutputIter copy(_InputIter __first, _InputIter __last,
                        _OutputIter __result) {
  __STL_REQUIRES(_InputIter, _InputIterator);
  __STL_REQUIRES(_OutputIter, _OutputIterator);
  typedef typename iterator_traits<_InputIter>::value_type _Tp;
  typedef typename __type_traits<_Tp>::has_trivial_assignment_operator
          _Trivial;
  return __copy_dispatch<_InputIter, _OutputIter, _Trivial>
    ::copy(__first, __last, __result);
}
```

利用triats技术，提取当前模板迭代器的 `value_type` 和 `has_trivial_assignment_operator`， 接着调用 `__copy_dispatch`。`_copy_dispatch`是一个模板仿函数，其定义如下：

> trivial_assignment_operator:
>
> In C++, a trivial assignment operator is an implicitly-defined copy assignment operator for a class or struct that meets certain criteria. A copy assignment operator is a special member function of a class or struct that is invoked when an object is assigned a value of the same type.
>
> For the copy assignment operator to be trivial, the following conditions must be satisfied:
>
> - **The class or struct has no virtual functions or virtual base classes.**
> - **All the non-static data members of the class or struct have trivial copy assignment operators.**
>
> If these conditions are met, the copy assignment operator for the class or struct will be implicitly defined by the compiler as a trivial assignment operator. A trivial assignment operator simply copies the values of all the non-static data members of the object being assigned from the source object.
>
> Trivial assignment operators are useful in C++ because they can be more efficient than user-defined assignment operators, as they can often be optimized by the compiler. In addition, certain types of operations, such as copying objects into arrays or memory-mapped files, require that the object has a trivial assignment operator.

```cpp
// 完全泛化版本 __copy_dispatch
template <class _InputIter, class _OutputIter, class _BoolType>
struct __copy_dispatch {
  static _OutputIter copy(_InputIter __first, _InputIter __last,
                          _OutputIter __result) {
    typedef typename iterator_traits<_InputIter>::iterator_category _Category;
    typedef typename iterator_traits<_InputIter>::difference_type _Distance;
    return __copy(__first, __last, __result, _Category(), (_Distance*) 0);
  }
};

// 偏特化
template <class _Tp>
struct __copy_dispatch<_Tp*, _Tp*, __true_type>
{
  static _Tp* copy(const _Tp* __first, const _Tp* __last, _Tp* __result) {
    return __copy_trivial(__first, __last, __result);
  }
};

// 偏特化
template <class _Tp>
struct __copy_dispatch<const _Tp*, _Tp*, __true_type>
{
  static _Tp* copy(const _Tp* __first, const _Tp* __last, _Tp* __result) {
    return __copy_trivial(__first, __last, __result);
  }
};
```

对于特化版本，如果是指针类型并且其`has_trivial_assignment_operator`为`__true_type`，则调用 `__copy_trivial`:

```cpp
// 使用 memmove
template <class _Tp>
inline _Tp*
__copy_trivial(const _Tp* __first, const _Tp* __last, _Tp* __result) {
  memmove(__result, __first, sizeof(_Tp) * (__last - __first));
  return __result + (__last - __first);
}
```

因为指针类型代表内存空间连续，`has_trivial_assignment_operator`代表可直接复制。

> 思考：为什么采用`memmove`而不是`memcpy`?
>
> 因为`__result`可能和 `__first, __last`区间重合，`memmove`可以处理重合的case。
>
> 参考：[memmove 和 memcpy的区别以及处理内存重叠问题 - Li_Ning - 博客园 (cnblogs.com)](https://www.cnblogs.com/li-ning/p/9489998.html)

再看泛化版本的`__copy_dispatch`, 其提取`iterator_category`和`_Distance`后，调用了`__copy`:

```cpp
template <class _InputIter, class _OutputIter, class _Distance>
inline _OutputIter __copy(_InputIter __first, _InputIter __last,
                          _OutputIter __result,
                          input_iterator_tag, _Distance*)
{
  for ( ; __first != __last; ++__result, ++__first)
    *__result = *__first;
  return __result;
}

// 1. 此版本避免了 __first != __last 的判断
// 2. 编译器可能会进行循环展开
template <class _RandomAccessIter, class _OutputIter, class _Distance>
inline _OutputIter
__copy(_RandomAccessIter __first, _RandomAccessIter __last,
       _OutputIter __result, random_access_iterator_tag, _Distance*)
{
  for (_Distance __n = __last - __first; __n > 0; --__n) {
    *__result = *__first;
    ++__first;
    ++__result;
  }
  return __result;
}
```

**stl copy针对不同的迭代器category有不同的实现**：

- `_InputIter`, 仅读迭代器，采用顺序拷贝
- `_RandomAccessIter`, 随机迭代器，也是顺序拷贝，和`_InputIter`不同，避免了`__first != __last`, 一定程度上提升了拷贝效率。

## 2. uninitialized_copy

uninitialized_copy算法接受三个输入迭代器和一个输出迭代器。前两个输入迭代器定义了要复制的对象范围，第三个输入迭代器指定了未初始化的内存的位置，复制后的对象将在此处构建。输出迭代器用于在复制操作完成后返回目标范围的迭代器。

```cpp
// 内存的配置与对象的构造行为分离开来。
template <class _InputIter, class _ForwardIter>
inline _ForwardIter
  uninitialized_copy(_InputIter __first, _InputIter __last,
                     _ForwardIter __result)
{
  return __uninitialized_copy(__first, __last, __result,
                              __VALUE_TYPE(__result));
}

template <class _InputIter, class _ForwardIter, class _Tp>
inline _ForwardIter
__uninitialized_copy(_InputIter __first, _InputIter __last,
                     _ForwardIter __result, _Tp*)
{
  typedef typename __type_traits<_Tp>::is_POD_type _Is_POD;
  return __uninitialized_copy_aux(__first, __last, __result, _Is_POD());
}
```

> _Is_POD:
>
> In C++, a POD type (Plain Old Data type) is a class or struct that meets certain requirements that allow it to be used interchangeably with primitive data types.
>
> A type is considered to be a POD type if it satisfies the following conditions:
>
> - It is an aggregate type (a class or struct with no user-defined constructors, no virtual functions, and no private or protected non-static data members).
> - It has no user-defined copy assignment operator, move assignment operator, copy constructor, or destructor.
> - All its non-static data members are themselves POD types or arrays of POD types.
>
> POD types are useful in C++ because they can be more efficiently manipulated and passed around than other types. For example, they can be easily serialized and deserialized, and can be used as the underlying type for memory-mapped files.

如果 `_Is_POD()` 为 `__true_type`:

```cpp
template <class _InputIter, class _ForwardIter>
inline _ForwardIter 
__uninitialized_copy_aux(_InputIter __first, _InputIter __last,
                         _ForwardIter __result,
                         __true_type)
{
  return copy(__first, __last, __result);
}
```

直接调用前文已经分析过的`copy`算法（内部可能直接memmove，或者循环赋值。

如果 `_Is_POD()` 为 `__false_type`:

```cpp
template <class _InputIter, class _ForwardIter>
_ForwardIter 
__uninitialized_copy_aux(_InputIter __first, _InputIter __last,
                         _ForwardIter __result,
                         __false_type)
{
  _ForwardIter __cur = __result;
  __STL_TRY {
    for ( ; __first != __last; ++__first, ++__cur)
      _Construct(&*__cur, *__first);
    return __cur;
  }
  __STL_UNWIND(_Destroy(__result, __cur));
}
```

1. 调用`_Construct`(placement new)
2. 如果未全部成功（抛出异常），则调用`_Destroy`对已经构造部分执行析构。



## 3. fill

fill族函数有: `fill`和`fill_n`， 代码非常简单，如下：

```cpp
// fill and fill_n

// 将 [first, last) 内的所有元素改填新值
template <class _ForwardIter, class _Tp>
void fill(_ForwardIter __first, _ForwardIter __last, const _Tp& __value) {
  __STL_REQUIRES(_ForwardIter, _Mutable_ForwardIterator);
  for ( ; __first != __last; ++__first)
    *__first = __value;
}

// 将 [first, last) 内的前 n 个元素改填新值
template <class _OutputIter, class _Size, class _Tp>
_OutputIter fill_n(_OutputIter __first, _Size __n, const _Tp& __value) {
  __STL_REQUIRES(_OutputIter, _OutputIterator);
  for ( ; __n > 0; --__n, ++__first)
    *__first = __value;
  return __first;
}

// Specialization: for one-byte types we can use memset.
// 特化，使用 memset
inline void fill(unsigned char* __first, unsigned char* __last,
                 const unsigned char& __c) {
  unsigned char __tmp = __c;
  memset(__first, __tmp, __last - __first);
}

inline void fill(signed char* __first, signed char* __last,
                 const signed char& __c) {
  signed char __tmp = __c;
  memset(__first, static_cast<unsigned char>(__tmp), __last - __first);
}

inline void fill(char* __first, char* __last, const char& __c) {
  char __tmp = __c;
  memset(__first, static_cast<unsigned char>(__tmp), __last - __first);
}
```

通常利用了模板特化：

1. 泛化版：循环赋值
2. char类指针，直接采用`memset`

## 4. uninitialized_fill

`uninitialized_fill`是C++标准库中的一个算法函数，它可以将指定范围内的元素全部初始化为给定的值，而不进行赋值操作。

与`fill`算法类似，`uninitialized_fill`也是将指定范围内的元素设置为指定的值，但是它不会调用元素的构造函数或拷贝构造函数，因此它比`fill`更快。`uninitialized_fill`通常用于在未初始化的内存区域中创建对象。它将使用placement new运算符构造对象，这意味着它将调用元素的构造函数。

```cpp
template <class _ForwardIter, class _Tp, class _Tp1>
inline void __uninitialized_fill(_ForwardIter __first, 
                                 _ForwardIter __last, const _Tp& __x, _Tp1*)
{
  typedef typename __type_traits<_Tp1>::is_POD_type _Is_POD;
  __uninitialized_fill_aux(__first, __last, __x, _Is_POD());
                   
}

template <class _ForwardIter, class _Tp>
inline void uninitialized_fill(_ForwardIter __first,
                               _ForwardIter __last, 
                               const _Tp& __x)
{
  __uninitialized_fill(__first, __last, __x, __VALUE_TYPE(__first));
}

template <class _ForwardIter, class _Tp>
inline void
__uninitialized_fill_aux(_ForwardIter __first, _ForwardIter __last, 
                         const _Tp& __x, __true_type)
{
  fill(__first, __last, __x);
}

template <class _ForwardIter, class _Tp>
void
__uninitialized_fill_aux(_ForwardIter __first, _ForwardIter __last, 
                         const _Tp& __x, __false_type)
{
  _ForwardIter __cur = __first;
  __STL_TRY {
    for ( ; __cur != __last; ++__cur)
      _Construct(&*__cur, __x);
  }
  __STL_UNWIND(_Destroy(__first, __cur));
}
```

同`uninitialized_copy`调用原理一致，这里不再赘述。

## 3. 总结

本篇很简单，算是对前几篇的一个补充，为后文分析容器时打下基础。
