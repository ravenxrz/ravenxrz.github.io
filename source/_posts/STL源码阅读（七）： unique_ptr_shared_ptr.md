---
title: STL源码阅读（七）： unique_ptr & shared_ptr
categories: stl
abbrlink: ee137bcd
date: 2023-04-15 20:11:31
tags:
---

本文首先分析 unique_ptr 的实现原理，介绍为什么其默认size仅为一个 raw pointer size； 接着分析了 shared_ptr 的实现原理，从两种构造方式入手，分析两种构造方式所带来的不同内存布局，最终通过分析`shared_ptr`的拷贝源码，简述了`shared_ptr`的线程安全性。

<!--more-->

## unique_ptr

`unique_ptr ` 是 c++ 11引入的智能之一，其特点是：

1. 对管理对象具有独占权，在`unique_ptr`析构时自动释放对象
2. 不可复制，但可移动
3. 支持自定义deleter
4. 零开销，`unique_ptr`和`raw pointer`一样大，不引入额外开销。

`unique_ptr` 定义如下：

```cpp
  template <typename _Tp, typename _Dp = default_delete<_Tp> >
    class unique_ptr 
```

`_Tp`为要管理的对象指针对象类型，`_Dp`为删除器类型，其默认值为 `default_delete`:

```cpp
  /// Primary template, default_delete.
  template<typename _Tp>
    struct default_delete
    {
      constexpr default_delete() noexcept = default;

      template<typename _Up, typename = typename
	       enable_if<is_convertible<_Up*, _Tp*>::value>::type>
        default_delete(const default_delete<_Up>&) noexcept { }

      void
      operator()(_Tp* __ptr) const
      {
	static_assert(sizeof(_Tp)>0,
		      "can't delete pointer to incomplete type");
	delete __ptr;
      }
    };
```

一个针对数组类型的特例化版本为:

```cpp
  template<typename _Tp, typename _Dp>
    class unique_ptr<_Tp[], _Dp>

    template<typename _Tp>
struct default_delete<_Tp[]>
{
private:
  template<typename _Up>
using __remove_cv = typename remove_cv<_Up>::type;

  // Like is_base_of<_Tp, _Up> but false if unqualified types are the same
  template<typename _Up>
using __is_derived_Tp
= __and_< is_base_of<_Tp, _Up>,
    __not_<is_same<__remove_cv<_Tp>, __remove_cv<_Up>>> >;

public:
  constexpr default_delete() noexcept = default;

  template<typename _Up, typename = typename
     enable_if<!__is_derived_Tp<_Up>::value>::type>
    default_delete(const default_delete<_Up[]>&) noexcept { }

  void
  operator()(_Tp* __ptr) const
  {
static_assert(sizeof(_Tp)>0,
      "can't delete pointer to incomplete type");
delete [] __ptr;
  }

  template<typename _Up>
typename enable_if<__is_derived_Tp<_Up>::value>::type
operator()(_Up*) const = delete;
};

```

本文仅讨论泛化版本。

首先看其**构造函数**:

```cpp
  explicit
  unique_ptr(pointer __p) noexcept
  : _M_t(__p, deleter_type())
  { static_assert(!is_pointer<deleter_type>::value,
     "constructed with null function pointer deleter"); }

  unique_ptr(pointer __p,
typename conditional<is_reference<deleter_type>::value,
  deleter_type, const deleter_type&>::type __d) noexcept
  : _M_t(__p, __d) { }

```

1. 直接使用默认deleter构造
2. 如果传入deleter为引用，则为引用类型的`__d`, 否则为 `const deleter_type&`的 `_d`

两个构造函数，均是初始化 `_M_t`。其定义为：

```cpp
      typedef std::tuple<typename _Pointer::type, _Dp>  __tuple_type;
      __tuple_type                                      _M_t;
```

`tuple_type`为一个二元元组类型。不过需要注意的是，若采用默认deleter, `_M_t`的大小仅为 8 Bytes(64位程序)。  **这是为何呢？按道理存储一个指针+deleter应该是大于一个指针的大小的， 就算deleter是一个empty class，其size也至少为1。**

这是因为tuple应用了 Empty Class Optimization (EBO) 技术。对于空基类其大小可以优化为 0 bytes. 具体可参考：[从tuple谈起-浅谈c++中空基类优化的使用 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/588929645)

但是若传入的deleter为一个函数指针，则 unique_ptr 为2个指针大小，若传入的为 lambda 表达式，则需视 lambda 的捕获列表情况而定， 若无任何捕获value，则 unique_ptr 依然为一个指针大小（因为 lambda 表达式本质上也是一个struct functor)

**再看析构函数:**

```cpp
      // Destructor.
      ~unique_ptr() noexcept
      {
	auto& __ptr = std::get<0>(_M_t);
	if (__ptr != nullptr)
	  get_deleter()(__ptr);
	__ptr = pointer();
      }
```

获取raw pointer, 若不为`nullptr`，调用`deleter`删除，同时将 raw pointer指向一个空`pointer`，通常也即为`nullptr`，避免野指针出现。

**unique_ptr如何实现对raw pointer的独占权？**，很简单，删除copy construct/assignment 即可：

```cpp
      // Disable copy from lvalue.
      unique_ptr(const unique_ptr&) = delete;
      unique_ptr& operator=(const unique_ptr&) = delete;

      // Disable construction from convertible pointer types.
      template<typename _Up, typename = _Require<is_pointer<pointer>,
	       is_convertible<_Up*, pointer>, __is_derived_Tp<_Up>>>
	unique_ptr(_Up*, typename
		   conditional<is_reference<deleter_type>::value,
		   deleter_type, const deleter_type&>::type) = delete;

      template<typename _Up, typename = _Require<is_pointer<pointer>,
	       is_convertible<_Up*, pointer>, __is_derived_Tp<_Up>>>
	unique_ptr(_Up*, typename
		   remove_reference<deleter_type>::type&&) = delete;

```

但**仍然保留`move`语义：**

```cpp
  // Move constructor.
  unique_ptr(unique_ptr&& __u) noexcept
  : _M_t(__u.release(), std::forward<deleter_type>(__u.get_deleter())) { }

        // Assignment.
    unique_ptr&
    operator=(unique_ptr&& __u) noexcept
    {
reset(__u.release());
get_deleter() = std::forward<deleter_type>(__u.get_deleter());
return *this;
    }

    template<typename _Up, typename _Ep>
typename
enable_if<__safe_conversion<_Up, _Ep>::value, unique_ptr&>::type
operator=(unique_ptr<_Up, _Ep>&& __u) noexcept
{
  reset(__u.release());
  get_deleter() = std::forward<_Ep>(__u.get_deleter());
  return *this;
}

    unique_ptr&
    operator=(nullptr_t) noexcept
    {
reset();
return *this;
    }

      void
      reset(pointer __p = pointer()) noexcept
      {
	using std::swap;
	swap(std::get<0>(_M_t), __p);
	if (__p != pointer())
	  get_deleter()(__p);
      }

      void
      swap(unique_ptr& __u) noexcept
      {
      using std::swap;
      swap(_M_t, __u._M_t);
      }


```

## shared_ptr

`std::shared_ptr` 是一种智能指针，它提供了对动态分配内存的共享所有权的特点包括：

- 引用计数：`std::shared_ptr` 使用引用计数来跟踪有多少个 `std::shared_ptr` 对象共享同一个内存块。当最后一个 `std::shared_ptr` 超出作用域时，它所管理的对象将被自动删除。
- 共享所有权：多个 `std::shared_ptr` 对象可以共享对同一个对象的所有权。这意味着，当其中一个 `std::shared_ptr` 超出作用域时，它所管理的对象不会被删除，只有当最后一个 `std::shared_ptr` 超出作用域时，对象才会被删除。
- 自定义删除器：`std::shared_ptr` 允许你指定自定义删除器来删除所管理的对象。这在管理非内存资源（如文件描述符或数据库连接）时非常有用。
- 弱指针支持：`std::shared_ptr` 支持弱指针（`std::weak_ptr`）。弱指针允许你观察共享对象，而不增加引用计数。这在解决循环引用问题时非常有用。

### 1. 类图

shared_ptr的相关类图为：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20230416212536528.png)

`shared_ptr` 继承自 `__shared_ptr`，`__shared_ptr` 内部有指向 要管理的指针`_M_ptr`和一个 `_shared_count`用于计数（实际上也就是shared_ptr的控制块)。 `_shared_count`实际组合了`_Sp_counted_base`， 该类为基类，有三个继承类：

- `_Sp_counted_deleter` 是一个辅助类，用于管理 `std::shared_ptr` 的自定义删除器。它负责在引用计数变为零时调用删除器来删除所管理的对象。
- `_Sp_counted_ptr_inplace` 是一个辅助类，用于在 `std::make_shared` 中分配内存。它负责在同一块内存中分配引用计数和所管理的对象。 （猜想为make_shared)
- `_Sp_counted_ptr` 是一个辅助类，用于管理 `std::shared_ptr` 的指针。它负责维护指向所管理对象的指针，并在引用计数变为零时调用删除器来删除所管理的对象。 （关注此类即可）

> 问题：shared_ptr的大小是多少？ (假设在64位平台)
>
> 答： 16Bytes， 一个 raw pointer, 一个指向 `_Sp_counted_base`的指针
>
> 问题： shared_ptr的控制块大小是多少？
>
> 答：8（vtable) + 4 (`_M_use_count`) + 4 (`_M_weak_count`) + 8 (`_M_ptr`)


### 2. 两种构造方式

`shared_ptr` 的使用有两种方式：

1. `shared_ptr<T>(new T)`
2. `make_shared<T>()`

两者实现不同, 一个个看：

**方式一：`shared_ptr<T>(new T)` 方式：**

```cpp
      template<typename _Tp1>
	explicit __shared_ptr(_Tp1* __p)
        : _M_ptr(__p), _M_refcount(__p) // 关注 _M_refcount 构造
	{
	  __glibcxx_function_requires(_ConvertibleConcept<_Tp1*, _Tp*>)
	  static_assert( sizeof(_Tp1) > 0, "incomplete type" );
	  __enable_shared_from_this_helper(_M_refcount, __p, __p);
	}
```

`_M_refcount`初始化:

```cpp
      template<typename _Ptr>
        explicit
	__shared_count(_Ptr __p) : _M_pi(0)
	{
	  __try
	    {
	      _M_pi = new _Sp_counted_ptr<_Ptr, _Lp>(__p);  // 此处
	    }
	  __catch(...)
	    {
	      delete __p;
	      __throw_exception_again;
	    }
	}

```

构造 `_Sp_counted_ptr` 时，需首先构造其父类：

```cpp
      _Sp_counted_base() noexcept
      : _M_use_count(1), _M_weak_count(1) { }
```

至此引用计数+1。

>**强调：此种方式构造，原有被管理指针和控制块分属两块内存。**

**方式二：`make_shared<T>()`**

`make_shared`相对复杂， 简化版本的 `make_shared` 可如下:

```cpp
template<typename T, typename... Args>
std::shared_ptr<T> make_shared(Args&&... args)
{
    // 分配内存，用于存储对象和控制块
    auto memory = new char[sizeof(T) + sizeof(std::shared_ptr<T>::_ControlBlock)];

    // 在分配的内存中构造对象
    T* object = new(memory) T(std::forward<Args>(args)...);

    // 在分配的内存中构造控制块
    auto controlBlock = new(memory + sizeof(T)) std::shared_ptr<T>::_ControlBlock(object);

    // 返回 shared_ptr 对象
    return std::shared_ptr<T>(object, controlBlock);
}
```

**即对象和控制块在连续内存空间中。**

实际源码如下：

```cpp
 
 /**
   *  @brief  Create an object that is owned by a shared_ptr.
   *  @param  __args  Arguments for the @a _Tp object's constructor.
   *  @return A shared_ptr that owns the newly created object.
   *  @throw  std::bad_alloc, or an exception thrown from the
   *          constructor of @a _Tp.
   */
  template<typename _Tp, typename... _Args>
    inline shared_ptr<_Tp>
    make_shared(_Args&&... __args)
    {
      typedef typename std::remove_const<_Tp>::type _Tp_nc;
      return std::allocate_shared<_Tp>(std::allocator<_Tp_nc>(),
				       std::forward<_Args>(__args)...);
    }
    
     /**
   *  @brief  Create an object that is owned by a shared_ptr.
   *  @param  __a     An allocator.
   *  @param  __args  Arguments for the @a _Tp object's constructor.
   *  @return A shared_ptr that owns the newly created object.
   *  @throw  An exception thrown from @a _Alloc::allocate or from the
   *          constructor of @a _Tp.
   *
   *  A copy of @a __a will be used to allocate memory for the shared_ptr
   *  and the new object.
   */
  template<typename _Tp, typename _Alloc, typename... _Args>
    inline shared_ptr<_Tp>
    allocate_shared(const _Alloc& __a, _Args&&... __args)
    {
      return shared_ptr<_Tp>(_Sp_make_shared_tag(), __a,
			     std::forward<_Args>(__args)...);  // 调用 shared_ptr 构造函数
    }

```

将走到这里：

```cpp
      template<typename _Alloc, typename... _Args>
	shared_ptr(_Sp_make_shared_tag __tag, const _Alloc& __a,
		   _Args&&... __args)
	: __shared_ptr<_Tp>(__tag, __a, std::forward<_Args>(__args)...) // __shared_ptr 构造
	{ }

```

而后：

```cpp
      template<typename _Alloc, typename... _Args>
	__shared_ptr(_Sp_make_shared_tag __tag, const _Alloc& __a,
		     _Args&&... __args)
    : _M_ptr(), _M_refcount(__tag, (_Tp*)0, __a,
				std::forward<_Args>(__args)...)  // 关注 _M_refcount 构造，待会马上分析
	{
	  // _M_ptr needs to point to the newly constructed object.
	  // This relies on _Sp_counted_ptr_inplace::_M_get_deleter.
	  void* __p = _M_refcount._M_get_deleter(typeid(__tag)); // 获取管理的对象内存地址， 后文分析
	  _M_ptr = static_cast<_Tp*>(__p);
	  __enable_shared_from_this_helper(_M_refcount, _M_ptr, _M_ptr);
	}
```

**内存分配逻辑在：**

```cpp
      template<typename _Tp, typename _Alloc, typename... _Args>
	__shared_count(_Sp_make_shared_tag, _Tp*, const _Alloc& __a,
		       _Args&&... __args)
	: _M_pi(0)
	{
	  typedef _Sp_counted_ptr_inplace<_Tp, _Alloc, _Lp> _Sp_cp_type;  //  采用_Sp_counted_ptr_inplace的控制块类型
	  typedef typename allocator_traits<_Alloc>::template
	    rebind_traits<_Sp_cp_type> _Alloc_traits;
	  typename _Alloc_traits::allocator_type __a2(__a);
	  _Sp_cp_type* __mem = _Alloc_traits::allocate(__a2, 1);
	  __try
	    {
	      _Alloc_traits::construct(__a2, __mem, std::move(__a),
		    std::forward<_Args>(__args)...); // 在 __mem 处构造 _Sp_counted_ptr_inplace 对象
	      _M_pi = __mem;   
	    }
	  __catch(...)
	    {
	      _Alloc_traits::deallocate(__a2, __mem, 1);
	      __throw_exception_again;
	    }
	}

```

上述函数实际上为构造 `_Sp_counted_ptr_inplace` 对象，该对象包含了所要分配的类型内存和控制块。

```cpp
    // class members
    private:
      _Impl _M_impl; // 指向所需要对象的内存地址
      typename aligned_storage<sizeof(_Tp), alignment_of<_Tp>::value>::type
	_M_storage;  // 所要构造的对象内存

    
    //  构造函数
    template<typename... _Args>
	_Sp_counted_ptr_inplace(_Alloc __a, _Args&&... __args)
	: _M_impl(__a)
	{
	  _M_impl._M_ptr = static_cast<_Tp*>(static_cast<void*>(&_M_storage));
	  // _GLIBCXX_RESOLVE_LIB_DEFECTS
	  // 2070.  allocate_shared should use allocator_traits<A>::construct
	  allocator_traits<_Alloc>::construct(__a, _M_impl._M_ptr,
	      std::forward<_Args>(__args)...); // might throw 
	}


```

从上述代码分析来看：

1. 所要构造的对象存储在 `_M_storage`， 同时`_M_impl`指向了该片内存
2. `_Sp_counted_ptr_inplace` 本身继承了 `_Sp_counted_base`, 则包含控制块信息(计数器)

回到此函数：

```cpp
    template<typename _Alloc, typename... _Args>
	__shared_ptr(_Sp_make_shared_tag __tag, const _Alloc& __a,
		     _Args&&... __args)
    : _M_ptr(), _M_refcount(__tag, (_Tp*)0, __a,
				std::forward<_Args>(__args)...)  // 关注 _M_refcount 构造，待会马上分析
	{
	  // _M_ptr needs to point to the newly constructed object.
	  // This relies on _Sp_counted_ptr_inplace::_M_get_deleter.
	  void* __p = _M_refcount._M_get_deleter(typeid(__tag)); // 获取管理的对象内存地址， 后文分析
	  _M_ptr = static_cast<_Tp*>(__p);
	  __enable_shared_from_this_helper(_M_refcount, _M_ptr, _M_ptr);
	}
```

`__shared_ptr` 中也有个需要指向所需对象内存的指针`_M_ptr`， 其调用了`_Sp_counted_ptr_inplace`的 `_M_get_deleter`:

```cpp
      void*
      _M_get_deleter(const std::type_info& __ti) const noexcept
      { return _M_pi ? _M_pi->_M_get_deleter(__ti) : 0; }

      // Sneaky trick so __shared_ptr can get the managed pointer
      virtual void*
      _M_get_deleter(const std::type_info& __ti) noexcept
      {
	return __ti == typeid(_Sp_make_shared_tag)
	       ? static_cast<void*>(&_M_storage) // 返回此地址
	       : 0;
      }
```

至此，`make_shared`的源码分析完成。 实际上总结一句话： **make_shared所构造的对象和其控制块绑定在一起，减少了一次内存分配，同时对cache更为友好**

**接下来分析当ref降为0时的逻辑**

析构流程： `shared_ptr` -> `__shared_ptr` -> `_M_refcount` :

```cpp
      ~__shared_count() // nothrow
      {
	if (_M_pi != 0)
	  _M_pi->_M_release();
      }

      void
      _M_release() noexcept
      {
        // Be race-detector-friendly.  For more info see bits/c++config.
        _GLIBCXX_SYNCHRONIZATION_HAPPENS_BEFORE(&_M_use_count);
	if (__gnu_cxx::__exchange_and_add_dispatch(&_M_use_count, -1) == 1)
	  {
            _GLIBCXX_SYNCHRONIZATION_HAPPENS_AFTER(&_M_use_count);
	    _M_dispose();
	    // There must be a memory barrier between dispose() and destroy()
	    // to ensure that the effects of dispose() are observed in the
	    // thread that runs destroy().
	    // See http://gcc.gnu.org/ml/libstdc++/2005-11/msg00136.html
	    if (_Mutex_base<_Lp>::_S_need_barriers)
	      {
	        _GLIBCXX_READ_MEM_BARRIER;
	        _GLIBCXX_WRITE_MEM_BARRIER;
	      }

            // Be race-detector-friendly.  For more info see bits/c++config.
            _GLIBCXX_SYNCHRONIZATION_HAPPENS_BEFORE(&_M_weak_count);
	    if (__gnu_cxx::__exchange_and_add_dispatch(&_M_weak_count,
						       -1) == 1)
              {
                _GLIBCXX_SYNCHRONIZATION_HAPPENS_AFTER(&_M_weak_count);
	        _M_destroy();
              }
	  }
      }
```

1. 如果`_M_use_count`减为0， 调用 `_M_dispose`释放对象内存
2. 如果 `_M_weak_count` 减为0， 调用 `_M_destroy`释放控制块

**对于第一种构造方式(new xxx)， 走 `_S_counted_ptr`**:

```cpp
      virtual void
      _M_dispose() noexcept
      { delete _M_ptr; } // 析构释放内存对象

      // Called when _M_weak_count drops to zero.
      virtual void
      _M_destroy() noexcept
      { delete this; }  // 析构释放控制块
```

**对于第二种构造方式（make_shared), 走 `_Sp_counted_ptr_inplace`** :

```cpp
      virtual void
      _M_dispose() noexcept
      { allocator_traits<_Alloc>::destroy(_M_impl, _M_impl._M_ptr); } // 析构内存对象

      // Override because the allocator needs to know the dynamic type
      virtual void
      _M_destroy() noexcept
      {
	typedef typename allocator_traits<_Alloc>::template
	  rebind_traits<_Sp_counted_ptr_inplace> _Alloc_traits;
	typename _Alloc_traits::allocator_type __a(_M_impl);
	_Alloc_traits::destroy(__a, this);  // 析构控制块
	_Alloc_traits::deallocate(__a, this, 1); // 释放整块内存
      }
```

### 3. 拷贝与线程安全性

**构造析构介绍完成，接着看看拷构造/赋值**，毕竟 `shared_ptr` 是共享的：

```cpp
shared_ptr(const shared_ptr&) noexcept = default;

shared_ptr& operator=(const shared_ptr&) noexcept = default;

```

走的默认，看其父类拷贝构造：

```cpp
__shared_ptr(const __shared_ptr&) noexcept = default;

template<typename _Tp1>  
__shared_ptr&  
operator=(const __shared_ptr<_Tp1, _Lp>& __r) noexcept  
{  
_M_ptr = __r._M_ptr;  
_M_refcount = __r._M_refcount; // __shared_count::op= doesn't throw  
return *this;  
}
```

也就是说 `__shared_ptr` 中的`_M_ptr`和`_M_refcount`会被分别拷贝。

**从这点可以看出， `shared_ptr` 本身的拷贝绝不是线程安全的。** 也就如下case是不正确的：

```cpp
std::shared_ptr<xx> g_ptr(new xx);

// thread 1
auto t1_ptr = g_ptr;

// thread 2
g_ptr = t2_ptr;

```

那对 **shared_ptr** 解引用是否线程安全？ 

```cpp
  // Allow class instantiation when _Tp is [cv-qual] void.
  typename std::add_lvalue_reference<_Tp>::type
  operator*() const noexcept
  {
_GLIBCXX_DEBUG_ASSERT(_M_ptr != 0);
return *_M_ptr;
  }

```

**答案依然是否**， 直接返回指向对象的引用。

那对 **控制块** 的操作是否线程安全？

**答案是 YES**， 控制块本身的++，--赋值等操作都是原子的。 具体给个hint，不做详细分析。

控制块基类为 `_Sp_counted_base`, 其内部成员为:

```
_Atomic_word  _M_use_count;     // #shared
_Atomic_word  _M_weak_count;    // #weak + (#shared != 0)
```

对它们的操作都是原子的。

至此 `shared_ptr`分析完成。

## 总结

本文较详细地分析了 `unique_ptr` 以及 `shared_ptr` 的关键源码，知道了 `unique_ptr` 大小默认情况下仅为 `raw_pointer` 的大小，这是因为编译器采用了EBO的优化技术（当然不同编译器可能优化方式不一）。 同时了解了 `shared_ptr` 的两种构造方式所带来的不同内存布局。 `new xx`方式，会导致两次内存申请，而 `make_shared` 方式，指针和控制块均在同一片内存，只有一次内存申请，最后还解释了为什么 `shared_ptr` 不是线程安全的。