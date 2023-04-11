---
title: STL源码阅读（六）： string
categories: stl
date: 2023-04-10 20:11:31
tags:
---

string是c++中标准的字符类（其实字节类更贴切），几乎所有项目都在使用（除非是明令禁止），了解其内部实现，可以帮我们更好的使用它。今天来分析下它。

> 本文分析的是 3.3 版本string，额外介绍了一些新版的实现。

<!--more-->

和 `string`相关的文件包括: `stdL_string_fwd.h`以及 `string` 文件。

`string` 其实是 `basic_string`的别名：

```cpp
typedef basic_string<char>    string; 
```

`basic_string`实际为一个模板类：

```cpp
// ------------------------------------------------------------
// Class basic_string.  

// Class invariants:
// (1) [start, finish) is a valid range.
// (2) Each iterator in [start, finish) points to a valid object
//     of type value_type.
// (3) *finish is a valid object of type value_type; in particular,
//     it is value_type().
// (4) [finish + 1, end_of_storage) is a valid range.
// (5) Each iterator in [finish + 1, end_of_storage) points to 
//     unininitialized memory.

// Note one important consequence: a string of length n must manage
// a block of memory whose size is at least n + 1.  
template <class _CharT, 
          class _Traits = char_traits<_CharT>, 
          class _Alloc = __STL_DEFAULT_ALLOCATOR(_CharT) >
class basic_string;
```

`basic_string`的类图如下：

![image-20230411174547291](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20230411174547291.png)

看起来和`vector`的继承关系基本一样。

## 1. 默认构造函数

`basic_string`默认构造函数：

```cpp
typedef _String_base<_CharT,_Alloc> _Base;


explicit basic_string(const allocator_type& __a = allocator_type())
    : _Base(__a, 8) { _M_terminate_string(); }

// base 的构造函数
_String_base(const allocator_type&, size_t __n)
    : _M_start(0), _M_finish(0), _M_end_of_storage(0)
    { _M_allocate_block(__n); }

// 分配内存
void _M_allocate_block(size_t __n) { 
  if (__n <= max_size()) {
    _M_start  = _M_allocate(__n);
    _M_finish = _M_start;
    _M_end_of_storage = _M_start + __n;
  }
  else
    _M_throw_length_error();
}
_Tp* _M_allocate(size_t __n) { return _Alloc_type::allocate(__n); }


// 构造末尾的\0
void _M_terminate_string() {
  __STL_TRY {
    _M_construct_null(_M_finish);
  }
  __STL_UNWIND(destroy(_M_start, _M_finish));
}

void _M_construct_null(_CharT* __p) {
    construct(__p);
#   ifdef __STL_DEFAULT_CONSTRUCTOR_BUG
    __STL_TRY {
      *__p = (_CharT) 0;
    }
    __STL_UNWIND(destroy(__p));
#   endif
  }

template <class _T1>
inline void construct(_T1* __p) {
  _Construct(__p);
}

// new T1()的方式，构造的默认值是0
template <class _T1>
inline void _Construct(_T1* __p) {
  new ((void*) __p) _T1();
}
```

可以看到，调用 `string`的默认构造函数时，将分配8个`char`(8Bytes), 并将第一个字节构造为 `\0`。此时， `_M_finish`和`_M_start`同时指向`'\0'`位置处。

## 2. += 操作

```cpp
  basic_string& operator+=(const basic_string& __s) { return append(__s); }
  basic_string& operator+=(const _CharT* __s) { return append(__s); }
  basic_string& operator+=(_CharT __c) { push_back(__c); return *this; }
```

先看`push_back` , 下面是老版本实现。

### 1. push_back 

可能存在扩容，如果当前size=0, 则扩容至2， 如果size!=0, 则扩容至2倍+1.

源码:

```c++
  void push_back(_CharT __c) {
    if (_M_finish + 1 == _M_end_of_storage)
      reserve(size() + max(size(), static_cast<size_type>(1)));
    _M_construct_null(_M_finish + 1);
    _Traits::assign(*_M_finish, __c);
    ++_M_finish;
  }

// Change the string's capacity so that it is large enough to hold
//  at least __res_arg elements, plus the terminating null.  Note that,
//  if __res_arg < capacity(), this member function may actually decrease
//  the string's capacity.
template <class _CharT, class _Traits, class _Alloc> 
void basic_string<_CharT,_Traits,_Alloc>::reserve(size_type __res_arg) {
  if (__res_arg > max_size())
    _M_throw_length_error();

  size_type __n = max(__res_arg, size()) + 1;
  pointer __new_start = _M_allocate(__n);
  pointer __new_finish = __new_start;

  __STL_TRY {
    __new_finish = uninitialized_copy(_M_start, _M_finish, __new_start);
    _M_construct_null(__new_finish);
  }
  __STL_UNWIND((destroy(__new_start, __new_finish), 
                _M_deallocate(__new_start, __n)));

  destroy(_M_start, _M_finish + 1);
  _M_deallocate_block();
  _M_start = __new_start;
  _M_finish = __new_finish;
  _M_end_of_storage = __new_start + __n;
}
```

`push_back`流程中：

1. 如果当前有可用空间，则在 `_M_finish+1` 处构造`'\0'`,并在`_M_finish`处添加`__c`
2. 否则，扩容2倍
3. `reserve`和 `vector`的 `reserve` 基本一致，不再分析。

### 2. append

可能存在扩容，2倍扩容或更多（满足要append传入的size)

```cpp
 basic_string& append(const basic_string& __s) 
    { return append(__s.begin(), __s.end()); }


template <class _Tp, class _Traits, class _Alloc> 
basic_string<_Tp, _Traits, _Alloc>& 
basic_string<_Tp, _Traits, _Alloc>::append(const _Tp* __first,
                                           const _Tp* __last)
{
  if (__first != __last) {
    const size_type __old_size = size();
    ptrdiff_t __n = __last - __first;
    if (__n > max_size() || __old_size > max_size() - __n)
      _M_throw_length_error();
    if (__old_size + __n > capacity()) {
      const size_type __len = __old_size + max(__old_size, (size_t) __n) + 1;
      pointer __new_start = _M_allocate(__len);
      pointer __new_finish = __new_start;
      __STL_TRY {
        __new_finish = uninitialized_copy(_M_start, _M_finish, __new_start);
        __new_finish = uninitialized_copy(__first, __last, __new_finish);
        _M_construct_null(__new_finish);
      }
      __STL_UNWIND((destroy(__new_start,__new_finish),
                    _M_deallocate(__new_start,__len)));
      destroy(_M_start, _M_finish + 1);
      _M_deallocate_block();
      _M_start = __new_start;
      _M_finish = __new_finish;
      _M_end_of_storage = __new_start + __len; 
    }
    else {
      const _Tp* __f1 = __first;
      ++__f1;
      uninitialized_copy(__f1, __last, _M_finish + 1);
      __STL_TRY {
        _M_construct_null(_M_finish + __n);
      }
      __STL_UNWIND(destroy(_M_finish + 1, _M_finish + __n));
      _Traits::assign(*_M_finish, *__first);
      _M_finish += __n;
    }
  }
  return *this;  
}
```

- 如果当前 `capacity` 足够容纳传入, 则将传入的范围拷贝至 `_M_finish` 处。
- 否则： 扩容（2倍扩容或满足传入的`__n`大小)，然后将老字符串拷贝至新分配地址，同时将传入的要 `append` 的字符创拷贝至默末尾。

### 3. cow机制

**上面的实现是非常老的版本了，实际上现代std::string 已经进化了两个版本，一是COW机制，另一个是SSO 短字符串优化**。

![07001538_63b8492ac460e30177](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img07001538_63b8492ac460e30177.webp)

cow机制是对`std::string `增加了 `ref_count`原子变量，若字符串变量可以共享存储，只有在有“修改”时才做实际拷贝。 **但实际上cow是有一些缺点的：**

1. 可能造成野指针：

```cpp
#include <string>
int main()
{
//    MyString str1("hello world\n");
//    auto str2( str1);
    std::string str1("hello world\n");
    //p指向str1的数据区
    const char * p = str1.data();
    {
        //str2和str1共享数据
        auto str2( str1);
        //访问导致str1复制数据，此时p和str2直线同一块区域
        str1[0];
        //str2离开作用域，调用析构函数，此时str2的RefCount为0，因此str2指向的内存被释放
    }
    //此时p为野指针，这是严重的bug
    printf(p);
    return 0;
}
```

**这是因为 [] 操作符会造成拷贝**，从而造成失效。

2. 对多线程不够友好。

```
std::string a = "hello";
std::string b = a;


// thread 1
a.pop_back();

// thread 2
b.push_back("x");
```

比如如上例子，cow对于如上情况，必须考虑一些同步方式，否则会有并发问题。 而引入额外的同步会带来性能衰减。

### 4. SSO 短字符串优化

sso短字符串优化考虑到大部分字符串是比较短的，那么可以在栈上预分配一些内存，短字符串字符在栈上存储，达到一定阈值后，才在堆上分配。

> 这种思想在c++11后挺多的，比如 std::function。

一种实现方式是：

```cpp
class sso_string // __gun_ext::__sso_string
{
  char* start;
  size_t size;
  static const int kLocalSize = 15;
  union
  {
    char buffer[kLocalSize + 1];
    size_t capacity;
  } data;
}
```

**如果字符长度小于15**， 则直接存在栈中，否则存在分配的heap上。 这里又用到了 union trick, 为什么可以使用union？

因为如果字符长度小于15， 直接可以在 buffer 上使用，否则buffer将不再使用， capacity 存放的是 heap 的长度。 `start` 指向当前可用内存片，`size`为实际所用大小。

**短字符串:**

![07001539_63b8492b06d1e81831](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img07001539_63b8492b06d1e81831.webp)

**长字符串**

![07001539_63b8492b30f0e64872](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img07001539_63b8492b30f0e64872.webp)

扩容和前面的扩容方式一致，2倍扩容。



![07001539_63b8492b5001f59037](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img07001539_63b8492b5001f59037.webp)

## 3. 总结

老版本`std::string`和 `vector`存储方式很相似，只不过默认多存一个`\0`, 在功能上多了很多function。新版包括 `cow` （实际上也被遗弃）和 `sso_string`,  这种小空间占用先放在 stack， 大空间占用再放在 `heap` 的思想在c++标准库中被反复用到，是个可以借鉴的idea.

## 4. 参考

1. [浅谈 C++ 字符串：std::string 与它的替身们_qq6380898cf2b73的技术博客_51CTO博客](https://blog.51cto.com/u_15891800/5995662)
2. [深入剖析 linux GCC 4.4 的 STL string - 360 核心安全技术博客](https://blogs.360.net/post/linux-gcc-stl-string-in-depth.html)
