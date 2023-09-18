---
title: STL源码阅读（五）： vector
categories: stl
abbrlink: 63c9d778
date: 2023-04-06 20:11:31
tags:
---

vector可以说用到最多的容器了。今天来分析它。

![image-20230410165416561](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20230410165416561.png)

<!--more-->

## 1. `_Vector_base`

`_Vector_base`是 `vector` 的模板基类，模板接收一个值类型`_Tp`, 和一个分配器类型`_Alloc`， 成员变量有三个 `_M_start`, `_M_finish`, `_M_end_of_storage`。 示意图如下：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgvector%E4%B8%89%E4%B8%AA%E6%88%90%E5%91%98%E5%8F%98%E9%87%8F.svg" alt="vector三个成员变量" style="zoom:150%;" />

- `_M_start` ：当前可插入的第一个位置
- `_M_finish`: 当前可插入的最后一个位置的后一个位置
- `_M_end_of_storage`: 当前申请的内存单元个数

代码如下：
```cpp
// 默认走这里，vector base 构造函数和析构函数
// vector 继承 _Vector_base
template <class _Tp, class _Alloc> 
class _Vector_base {
public:
  typedef _Alloc allocator_type;
  allocator_type get_allocator() const { return allocator_type(); }
  
  // 初始化
  _Vector_base(const _Alloc&)
    : _M_start(0), _M_finish(0), _M_end_of_storage(0) {}
  
  // 初始化，分配空间 
  _Vector_base(size_t __n, const _Alloc&)
    : _M_start(0), _M_finish(0), _M_end_of_storage(0) 
  {
    _M_start = _M_allocate(__n);
    _M_finish = _M_start;
    _M_end_of_storage = _M_start + __n;
  }
  
  // 释放空间
  ~_Vector_base() { _M_deallocate(_M_start, _M_end_of_storage - _M_start); }

protected:
  _Tp* _M_start;  // 表示目前使用空间的头
  _Tp* _M_finish; // 表示目前使用空间的尾
  _Tp* _M_end_of_storage; // 表示目前可用空间的尾

  // simple_alloc 是 SGI STL 的空间配置器
  typedef simple_alloc<_Tp, _Alloc> _M_data_allocator;  // 以元素大小为配置单位
  _Tp* _M_allocate(size_t __n)
    { return _M_data_allocator::allocate(__n); }
  void _M_deallocate(_Tp* __p, size_t __n) 
    { _M_data_allocator::deallocate(__p, __n); }
};

```

可以看到`_Vector_base`无特别的用途，仅是包装了三个重要变量和分配器类型（还记得vector的默认分配类型是什么吗？）

## 2. vector

通常来说，使用vector的接口包括：

1. reserve: 预留capacity
2. push_back: 尾追加
3. pop_back: 尾删除
4. insert: 插入
5. swap: 交换（通常用于快速析构释放内存）
6. Iterator

逐个说下如上接口是如何实现的：

### 1. reserve

`reserve`可能涉及内存分配与拷贝。推荐在构造`vector`后，立即调用`reserve`，而不要再“使用”过vector后，再reserve，除非明确要“扩容”到某个大小。

```c++
 size_type capacity() const  // vector 的容量
    { return size_type(_M_end_of_storage - begin()); }

  // 预留存储空间，若 __n 大于当前的 capacity() ，则分配新存储，否则该方法不做任何事。
 void reserve(size_type __n) {
    if (capacity() < __n) {
      const size_type __old_size = size();
      iterator __tmp = _M_allocate_and_copy(__n, _M_start, _M_finish);
      destroy(_M_start, _M_finish);
      _M_deallocate(_M_start, _M_end_of_storage - _M_start);
      _M_start = __tmp;
      _M_finish = __tmp + __old_size;
      _M_end_of_storage = _M_start + __n;
    }
  }

```

1. 如果 `capacity` 比传入的 `__n` 大，不做任何事情
2. 否则：分配`__n`个元素大小的内存，将原先的 `_M_start` 到 `_M_finish` 之间的内存拷贝到这片内存上，并返回拷贝后的末尾地址返给`__tmp`
3. 析构`_M_start`到`_M_finish`之间的对象
4. 释放`_M_start`到`_M_finish`之间的内存
5. 更新 `_M_start`，`_M_finish`和`_M_end_of_storage`指针

### 2. push_back

push_back可能扩容，扩容的机制是2倍扩容。

```cpp
  // 尾部插入
  void push_back(const _Tp& __x) {
    if (_M_finish != _M_end_of_storage) { // 有备用空间
      construct(_M_finish, __x);    // 全局函数，将 __x 设定到 _M_finish 指针所指的空间上
      ++_M_finish;         // 调整
    }
    else
      _M_insert_aux(end(), __x);  // 无备用空间，从新分配再插入
  }
```

如果有备用空间，则直接在`_M_finish`上构造对象，并`++_M_finish`

否则，调用`_M_insert_aux`:
```cpp
// push_back 调用, insert(position, x)调用
template <class _Tp, class _Alloc>
void 
vector<_Tp, _Alloc>::_M_insert_aux(iterator __position, const _Tp& __x)
{
  if (_M_finish != _M_end_of_storage) {
    construct(_M_finish, *(_M_finish - 1));
    ++_M_finish;
    _Tp __x_copy = __x;
    copy_backward(__position, _M_finish - 2, _M_finish - 1);
    *__position = __x_copy;
  }
  else {// 没有备用空间
    const size_type __old_size = size();
    const size_type __len = __old_size != 0 ? 2 * __old_size : 1;
    iterator __new_start = _M_allocate(__len);
    iterator __new_finish = __new_start;
    __STL_TRY {
      __new_finish = uninitialized_copy(_M_start, __position, __new_start);
      construct(__new_finish, __x);
      ++__new_finish;
      __new_finish = uninitialized_copy(__position, _M_finish, __new_finish);
    }
    __STL_UNWIND((destroy(__new_start,__new_finish), 
                  _M_deallocate(__new_start,__len)));
    destroy(begin(), end());
    _M_deallocate(_M_start, _M_end_of_storage - _M_start);
    _M_start = __new_start;
    _M_finish = __new_finish;
    _M_end_of_storage = __new_start + __len;
  }
}

```

看`else`分支即可：

1. 如果当前size为0， 则分配1个对象内存大小，否则扩容至两倍大小。（也就是说vector的内存分配是 1, 2, 4, 8... 的策略）
2. 先将 `_M_start` 到 `__position`位置处的所有内存拷贝至 `__new_finish`处
3. 在 `__new_finish` 处构造 `__x`，并`++_new_finish`
4. 将`__position`到`_M_finish`处的所有内存，拷贝至`__new_finish`
5. 最后析构并释放原来的老内存
6. 更新三个重要的成员变量

### 3. pop_back

```cpp
  // 将尾端元素删掉，并调整大小
  void pop_back() {
    --_M_finish;
    destroy(_M_finish);
  }
```

析构但并不释放内存。

> 思考： 看样子老版本的vector有个缺点：如果在过去的整个时间段内，只有非常短的时间里将vector插得很长，但只要vector不完全析构，这片很长的内存得不到释放。 老版本只能通过 `swap`的机制来完全释放内存，不像新版本可以使用`shrink_to_fit`可以灵活释放多余内存

### 4. insert

看过 `push_back`后， insert很简单：

```cpp
  // 在 __position 前插入 __x 
  iterator insert(iterator __position, const _Tp& __x) {
    size_type __n = __position - begin();
    if (_M_finish != _M_end_of_storage && __position == end()) {
      construct(_M_finish, __x);
      ++_M_finish;
    }
    else
      _M_insert_aux(__position, __x);
    return begin() + __n;
  }
```

若插入点为`end()`， 则直接插入。 否则走 `_M_insert_aux`流程。

### 5. swap

```cpp
  // 交换内容
  void swap(vector<_Tp, _Alloc>& __x) {
    __STD::swap(_M_start, __x._M_start);
    __STD::swap(_M_finish, __x._M_finish);
    __STD::swap(_M_end_of_storage, __x._M_end_of_storage);
  }

```

swap非常简单，和`x`对象交换内部三个重要的成员变量，此外，注意交换的对象的模板要完全一致，包括分配器的类型。这个操作的cost非常低，一般用于释放原内存，以及多并发的时候，将全局的某个带锁的vector的内存吐到局部不带锁的vector， 减少锁竞争。

如：

```c++
std::vector<int> vec1;
// push_back 100000000000 次
{
	std::vector<int> vec2;
	vec1.swap(vec2);
}
// vec1原内内存被释放
```

### 6. iterator相关

vector的iterator实际上就是value指针：

```cpp
  typedef _Tp value_type;
  typedef value_type* iterator; // vector 的迭代器是普通指针
```

所以:

```cpp
  iterator begin() { return _M_start; }   // 返回指向容器第一个元素的迭代器 
  const_iterator begin() const { return _M_start; }
  iterator end() { return _M_finish; }    // 返回指向容器尾端的迭代器 
  const_iterator end() const { return _M_finish; }
```

如上iterator函数族直接返回的是内存的成员变量。

## 3. 总结

vector容器相对简单，注意点主要是 **容量不足时的扩容策略（2倍扩容）**，另外，**vector也不是线程安全的**，使用时需要拷贝数据竞争问题。

