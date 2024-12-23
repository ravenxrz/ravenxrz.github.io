---
title: STL源码阅读（十二）： std::thread源码分析
categories: stl
date: 2024-12-23 21:17:31
tags:
---

> 📌本文采用wolai制作，原文链接：[https://www.wolai.com/ravenxrz/wQ6VWz7rhYAK4NWReW5ziB](https://www.wolai.com/ravenxrz/wQ6VWz7rhYAK4NWReW5ziB "https://www.wolai.com/ravenxrz/wQ6VWz7rhYAK4NWReW5ziB")

`std::thread` 是c++11后引入的标准线程库。作为c++中最重要的线程封装库，有必要打开看看。

注：本文所有分析基于 gcc-8.5.0 源码

一些重要结论:

> 📌由此可以得到一个重要结论：给`std::thread`传入的参数是没有引用效果的。 如果要用引用，请使用`std::ref`

> 📌此处可知，`thread::get_id` 返回的是 `pthread_t`, 该值被`pthread_create`初始化,所以并不代表linux下的thread id

> 📌thread不支持copy语义，仅支持move语义, 且move operator=的前提是，当前线程已经不是`joinable`的，否则会有资源泄露。

<!--more-->

# 1 构造函数

```c++
template <typename _Tp>
using __not_same = __not_<is_same<
    typename remove_cv<typename remove_reference<_Tp>::type>::type, thread>>;

template <typename _Callable, typename... _Args,
          typename = _Require<__not_same<_Callable>>>
explicit thread(_Callable &&__f, _Args &&...__args) {
  static_assert(
      __is_invocable<typename decay<_Callable>::type,
                     typename decay<_Args>::type...>::value,
      "std::thread arguments must be invocable after conversion to rvalues");

#ifdef GTHR_ACTIVE_PROXY
  // Create a reference to pthread_create, not just the gthr weak symbol.
  auto __depend = reinterpret_cast<void (*)()>(&pthread_create);
#else
  auto __depend = nullptr;
#endif
   _M_start_thread(_S_make_state(__make_invoker(std::forward<_Callable>(__f),
                                               std::forward<_Args>(__args)...)),
                  __depend); 
}
```

逐行看:

1. 构造函数是一个模板构造函数，模板生效条件是传入的`_Callable`模板类型不是`thread`类型。
2. 最重要的是高亮的部分。

## `1.1 __make_invoker`函数

```c++
// A call wrapper that does INVOKE(forwarded tuple elements...)
template <typename _Tuple> struct  _Invoker  {
  _Tuple _M_t;

  template <size_t _Index>
  static __tuple_element_t<_Index, _Tuple> &&_S_declval();

  template <size_t... _Ind>
  auto _M_invoke(_Index_tuple<_Ind...>) noexcept(
      noexcept(std::__invoke(_S_declval<_Ind>()...)))
      -> decltype(std::__invoke(_S_declval<_Ind>()...)) {
    return std::__invoke(std::get<_Ind>(std::move(_M_t))...);
  }

  using _Indices =
      typename _Build_index_tuple<tuple_size<_Tuple>::value>::__type;

   auto operator()() noexcept(  // 重点：最终线程的起点
      noexcept(std::declval<_Invoker &>()._M_invoke(_Indices())))
      -> decltype(std::declval<_Invoker &>()._M_invoke(_Indices())) {
    return _M_invoke(_Indices());
  } 
};

template <typename... _Tp>
using __decayed_tuple = tuple<typename std::decay<_Tp>::type...>;

// Returns a call wrapper that stores
//  tuple{DECAY_COPY(__callable), DECAY_COPY(__args)...}. 
template <typename _Callable, typename... _Args>
static _Invoker<__decayed_tuple<_Callable, _Args...>>
__make_invoker(_Callable &&__callable, _Args &&...__args) {
  return {__decayed_tuple<_Callable, _Args...>{
      std::forward<_Callable>(__callable), std::forward<_Args>(__args)...}};
}
```

这里的invoker本质上是一个tuple，tuple的类型是对所有传入的类型的**decay**后的结果。

> decay的作用：
>
> - 移除引用。
> - 移除 `const` 和 `volatile` 修饰符。
> - 将数组类型转换为指针类型。
> - 将函数类型转换为指向函数的指针类型。

decay的主要作用是让传入的类型按照值传递。

> 📌由此可以得到一个重要结论：给`std::thread`传入的参数是没有引用效果的。 如果要用引用，请使用`std::ref`

`_Invoker`类是thread最重要类，线程最终将通过`pthread_create`创建(下文分析），入口函数将转入这里的`operator()`。

`operator()`重载，通过 `std::__invoke`调用`tuple`中存储的调用实体指针以及参数，这里用了`move`，所以如果有引用，实际上引用的是`tuple`中的变量。

## `1.2 _S_make_state`函数

将`__make_invoker`的结果转入`_S_make_state`

```c++
template <typename _Callable> static _State_ptr _S_make_state(_Callable &&__f) {
  using _Impl = _State_impl<_Callable>;
  return _State_ptr{new _Impl{std::forward<_Callable>(__f)}};
}

```

这里包含两个类:

1. `_Impl`
2. `_State_ptr`

先看`_Impl`

```c++
// Abstract base class for types that wrap arbitrary functors to be
// invoked in the new thread of execution.
struct _State {
  virtual ~_State();
  virtual void _M_run() = 0;
};

template <typename _Callable> struct _State_impl : public _State {
  _Callable _M_func;

  _State_impl(_Callable &&__f) : _M_func(std::forward<_Callable>(__f)) {}

  void _M_run() { _M_func(); }
};

```

前文中的`_Invoker`存放在此处的`_M_func`

`_State_ptr`本质上就是一个`unique_ptr`:

> [std::unique\_ptr](https://www.wolai.com/5kf4Mci5ETPySzSafqkAiS "std::unique_ptr")源码分析见此。

```c++
  using _State_ptr = unique_ptr<_State>;

```

所以`_S_make_state`即是一个对`_invoker `unique\_ptr包装。

## `1.3 _M_start_thread`函数

`_S_make_state`的结果又转入到`_M_state_thread`

```c++
void thread::_M_start_thread(_State_ptr state, void (*)()) {
  const int err =  __gthread_create(&_M_id._M_thread,
                                   &execute_native_thread_routine, state.get()); 
  if (err)
    __throw_system_error(err);
  state.release();
}
```

看起来第二个参数没用，也即:

```c++
auto __depend = reinterpret_cast<void (*)()>(&pthread_create);
```

没用。

此处的`_M_id`为:

```c++
typedef pthread_t __gthread_t;

typedef __gthread_t      native_handle_type;


// thread::id
class id {
   native_handle_type _M_thread; 

public:
  id() noexcept : _M_thread() {}

  explicit id(native_handle_type __id) : _M_thread(__id) {}

private:
  friend class thread;
  friend class hash<thread::id>;

  friend bool operator==(thread::id __x, thread::id __y) noexcept;

  friend bool operator<(thread::id __x, thread::id __y) noexcept;

  // 这个就是打印的c++ thread id(注意不是linux上的tid)
   template <class _CharT, class _Traits>
  friend basic_ostream<_CharT, _Traits> &
  operator<<(basic_ostream<_CharT, _Traits> &__out, thread::id __id); 
};

id _M_id;
```

### `1.3.1 __gthread_create`函数

```c++
static inline int __gthread_create(__gthread_t *__threadid,
                                   void *(*__func)(void *), void *__args) {
  return __gthrw_(pthread_create)(__threadid, NULL, __func, __args);
}

```

到这里就很明显了，调用`pthread_create`创建线程. 线程的routine为:`execute_native_thread_routine`. 另外 `thread_id`也被`pthread_create`初始化。

> 📌此处可知，`thread::get_id` 返回的是 `pthread_t`, 该值被`pthread_create`初始化,所以并不代表linux下的thread id

### `1.3.2 execute_native_thread_routine`

```c++
static void *execute_native_thread_routine(void *__p) {
  thread::_State_ptr __t{static_cast<thread::_State *>(__p)};
  __t->_M_run();
  return nullptr;
}

```

转为前面创建的`invoker`，并调用`_M_run`，最终进入如下函数:

```c++
// class _Invoker
auto operator()() noexcept( // 重点：最终线程的起点
    noexcept(std::declval<_Invoker &>()._M_invoke(_Indices())))
    -> decltype(std::declval<_Invoker &>()._M_invoke(_Indices())) {
  return _M_invoke(_Indices());
}
```

这里前文已经提到：

`_Invoker`类是thread最重要类，线程最终将通过`pthread_create`创建(下文分析），入口函数将转入这里的`operator()`。

# 2 joinable & join

`joinable`源码：

```c++
  bool joinable() const noexcept { return !(_M_id == id()); }

```

很简单，看`id`是否有被初始化。（即是否调用过`pthread_create`)

再看`join`

```c++
void thread::join() {
  int __e = EINVAL;

  if (_M_id != id())
    __e = __gthread_join(_M_id._M_thread, 0);

  if (__e)
    __throw_system_error(__e);

  _M_id = id();  // 标记成不可joinable的， 将对应析构函数处的检查
}

```

依然很简单， 如果thread是joinable的，则调用`__gthread_join`， 如果`join`成功，将`_M_id`置空。

> 析构函数:
>
> ```c++
> ~thread() {
>   if (joinable())
>     std::terminate();
> }
>
>
> ```
>
> 所以如果join后不置空，会抛异常。

回头看`__gthread_join`

```c++
static inline int __gthread_join(__gthread_t __threadid, void **__value_ptr) {
  return __gthrw_(pthread_join)(__threadid, __value_ptr);
}

```

转到`pthread_join`

# 3 detach

detach转到`pthread_detach`

```c++
void thread::detach() {
  int __e = EINVAL;

  if (_M_id != id())
    __e = __gthread_detach(_M_id._M_thread);

  if (__e)
    __throw_system_error(__e);

  _M_id = id();
}

static inline int __gthread_detach(__gthread_t __threadid) {
  return __gthrw_(pthread_detach)(__threadid);
}

```

# 4 move语义支持

```c++
thread(const thread &) = delete;

thread(thread &&__t) noexcept { swap(__t); }

thread &operator=(const thread &) = delete;

thread &operator=(thread &&__t) noexcept {
  if (joinable())
    std::terminate();
  swap(__t);
  return *this;
}
```

> 📌thread不支持copy语义，仅支持move语义, 且move operator=的前提是，当前线程已经不是`joinable`的，否则会有资源泄露。

# `5 this_thread`

还有个经常用到的功能是 `std::thread::this_thread`

```c++
namespace this_thread {
/// get_id
inline thread::id get_id() noexcept {
#ifdef __GLIBC__
  // For the GNU C library pthread_self() is usable without linking to
  // libpthread.so but returns 0, so we cannot use it in single-threaded
  // programs, because this_thread::get_id() != thread::id{} must be true.
  // We know that pthread_t is an integral type in the GNU C library.
  if (!__gthread_active_p())
    return thread::id(1);
#endif
  return thread::id(__gthread_self());
}

/// yield
inline void yield() noexcept {
#ifdef _GLIBCXX_USE_SCHED_YIELD
  __gthread_yield();
#endif
}

```

本质上`this_thread`是一个namespace, 提供了一些常用功能，如 `get_id` 、`yield`

`get_id`转到:

```c++
static inline __gthread_t __gthread_self(void) {
  return __gthrw_(pthread_self)();
}

```

`yield`实现转到

```c++
static inline int __gthread_yield(void) { return __gthrw_(sched_yield)(); }

```



# 6 总结

本文深入分析了C++11标准库中的`std::thread`类，基于GCC-8.5.0源码。

文中详细剖析了`std::thread`的内部工作原理，包括构造函数、`_Invoker`类、`_State_ptr`与`_State_impl`类等关键组成部分。这些细节揭示了线程是如何通过`pthread_create`系统调用来启动的，以及`_Invoker`如何作为线程执行的起点。同时，还讨论了`joinable()`和`join()`方法的工作机制，以及线程析构时的注意事项。

最后，介绍了`std::thread::this_thread`命名空间提供的功能。

总的来说， gnu下的thread库，实际上是`pthread_create`的一层封装。

