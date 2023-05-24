---
title: STL源码阅读（八）： std::function源码分析
categories: stl
date: 2023-05-24 20:11:31
tags:
---



本篇将分析c++11引入的 `std::function` 源码。

<!--more-->


## TODO:

分析 std::ref 以及 reference_wrapper 相关部分的源码。



## 构造函数

```cpp
    template<typename _Res, typename... _ArgTypes>
    template<typename _Functor>
    function<_Res(_ArgTypes...)>::
    function(_Functor __f,
             typename __gnu_cxx::__enable_if<
                     !is_integral<_Functor>::value, _Useless>::__type)
            : _Function_base() {
        typedef _Function_handler<_Signature_type, _Functor> _My_handler;

        if (_My_handler::_M_not_empty_function(__f)) {
            _My_handler::_M_init_functor(_M_functor, __f);
            _M_invoker = &_My_handler::_M_invoke;
            _M_manager = &_My_handler::_M_manager;
        }
    }

```

简单几行代码，包含了较多类的概念，下面一点点分析。

### `_Function_handler`

这里的关键是`_Function_handler`：

```c++
	// 下面两种重载形式对应 普通函数指针、类实例或者Lambda表达式 ， 区别在于是否有返回值
template<typename _Res, typename _Functor, typename... _ArgTypes>
    class _Function_handler<_Res(_ArgTypes...), _Functor>
            : public _Function_base::_Base_manager<_Functor> {
        typedef _Function_base::_Base_manager<_Functor> _Base;

    public:
        static _Res
        _M_invoke(const _Any_data &__functor, _ArgTypes... __args) {
            return (*_Base::_M_get_pointer(__functor))(__args...);
        }
    };

    template<typename _Functor, typename... _ArgTypes>
    class _Function_handler<void(_ArgTypes...), _Functor>
            : public _Function_base::_Base_manager<_Functor> {
        typedef _Function_base::_Base_manager<_Functor> _Base;

    public:
        static void
        _M_invoke(const _Any_data &__functor, _ArgTypes... __args) {
            (*_Base::_M_get_pointer(__functor))(__args...);
        }
    };


	// 下面两种重载形式针对 reference_wrapper 的偏特化，区别在于是否有返回值
    template<typename _Res, typename _Functor, typename... _ArgTypes>
    class _Function_handler<_Res(_ArgTypes...), reference_wrapper<_Functor> >
            : public _Function_base::_Ref_manager<_Functor> {
        typedef _Function_base::_Ref_manager<_Functor> _Base;

    public:
        static _Res
        _M_invoke(const _Any_data &__functor, _ArgTypes... __args) {
            return
                    __callable_functor(**_Base::_M_get_pointer(__functor))(__args...);
        }
    };

    template<typename _Functor, typename... _ArgTypes>
    class _Function_handler<void(_ArgTypes...), reference_wrapper<_Functor> >
            : public _Function_base::_Ref_manager<_Functor> {
        typedef _Function_base::_Ref_manager<_Functor> _Base;

    public:
        static void
        _M_invoke(const _Any_data &__functor, _ArgTypes... __args) {
            __callable_functor(**_Base::_M_get_pointer(__functor))(__args...);
        }
    };

		// 下面两种针对类成员函数的偏特化
    template<typename _Class, typename _Member, typename _Res,
            typename... _ArgTypes>
    class _Function_handler<_Res(_ArgTypes...), _Member _Class::*>
            : public _Function_handler<void(_ArgTypes...), _Member _Class::*> {
        typedef _Function_handler<void(_ArgTypes...), _Member _Class::*>
                _Base;

    public:
        static _Res
        _M_invoke(const _Any_data &__functor, _ArgTypes... __args) {
            return tr1::
            mem_fn(_Base::_M_get_pointer(__functor)->__value)(__args...);
        }
    };

    template<typename _Class, typename _Member, typename... _ArgTypes>
    class _Function_handler<void(_ArgTypes...), _Member _Class::*>
            : public _Function_base::_Base_manager<
                    _Simple_type_wrapper<_Member _Class::*> > {
        typedef _Member _Class::* _Functor;
        typedef _Simple_type_wrapper<_Functor> _Wrapper;
        typedef _Function_base::_Base_manager<_Wrapper> _Base;

    public:
        static bool
        _M_manager(_Any_data &__dest, const _Any_data &__source,
                   _Manager_operation __op) {
            switch (__op) {
#ifdef __GXX_RTTI
                case __get_type_info:
                  __dest._M_access<const type_info*>() = &typeid(_Functor);
                  break;
#endif
                case __get_functor_ptr:
                    __dest._M_access<_Functor *>() =
                            &_Base::_M_get_pointer(__source)->__value;
                    break;

                default:
                    _Base::_M_manager(__dest, __source, __op);
            }
            return false;
        }

        static void
        _M_invoke(const _Any_data &__functor, _ArgTypes... __args) {
            tr1::mem_fn(_Base::_M_get_pointer(__functor)->__value)(__args...);
        }
    };



```

每种重载所做的工作都是： **取出实际的调用实体，然后传入参数予以调用**



### `_Any_data`

```cpp
 union _Any_data
  {
    void*       _M_access()       { return &_M_pod_data[0]; }
    const void* _M_access() const { return &_M_pod_data[0]; }

    template<typename _Tp>
      _Tp&
      _M_access()
      { return *static_cast<_Tp*>(_M_access()); }

    template<typename _Tp>
      const _Tp&
      _M_access() const
      { return *static_cast<const _Tp*>(_M_access()); }

    _Nocopy_types _M_unused;
    char _M_pod_data[sizeof(_Nocopy_types)];
  };
```

`_M_pod_data` 和 `_M_unused` 占用一样大的空间。从上述的几种`_M_access`方法来看，可以直接获取底层存储指针，也可以转换为实际的可调用实体类型指针。

`_Nocopy_types`定义如下：

```cpp
  union _Nocopy_types
  {
    void*       _M_object;					// 类实例或者lambda
    const void* _M_const_object;		// 类实例或者lambda
    void (*_M_function_pointer)();	// 函数指针
    void (_Undefined_class::*_M_member_pointer)();		// 成员函数指针
  };

```



### `_Function_base`

```cpp
    /// Base class of all polymorphic function object wrappers.
    class _Function_base {
    public:
        static const std::size_t _M_max_size = sizeof(_Nocopy_types);
        static const std::size_t _M_max_align = __alignof__(_Nocopy_types);

       	...

        bool _M_empty() const { return !_M_manager; }

        typedef bool (*_Manager_type)(_Any_data &, const _Any_data &,
                                      _Manager_operation);

        _Any_data _M_functor;
        _Manager_type _M_manager;
    };

```

有两个成员变量: `_M_functor` 即存储了底层的调用实体。  `_M_manager` 一个函数指针。

#### ` _Function_base::_Base_manager`

`_Function_base` 内部还有两个内部类，其中一个为`_Base_manager`:

```c++
        template<typename _Functor>
        class _Base_manager {
        protected:
            static const bool __stored_locally =
                    (__is_location_invariant<_Functor>::value
                     && sizeof(_Functor) <= _M_max_size
                     && __alignof__(_Functor) <= _M_max_align
                     && (_M_max_align % __alignof__(_Functor) == 0));

            typedef integral_constant<bool, __stored_locally> _Local_storage;

            // Retrieve a pointer to the function object
            static _Functor *
            _M_get_pointer(const _Any_data &__source) {
                const _Functor *__ptr =
                        __stored_locally ? std::__addressof(__source._M_access<_Functor>())
                        /* have stored a pointer */ : __source._M_access<_Functor *>();
                return const_cast<_Functor *>(__ptr);
            }

        		...
        public:
						...

            static void
            _M_init_functor(_Any_data &__functor, const _Functor &__f) {
                _M_init_functor(__functor, __f, _Local_storage());
            }

           ...

        private:
            static void
            _M_init_functor(_Any_data &__functor, const _Functor &__f, true_type) {
                new(__functor._M_access()) _Functor(__f);
            }

            static void
            _M_init_functor(_Any_data &__functor, const _Functor &__f,
                            false_type) { __functor._M_access<_Functor *>() = new _Functor(__f); }
        };

```

这里可以看出：

1. `_Local_storage` 类型表示`_Functor`的size 是否小于`_M_max_size` （即前面`_Any_data`的大小), 是否按照`_M_max_align`对齐。如果满足，则为`true`否则为`false`。
2. `_M_init_functor` 根据`_Local_storage`决定将`_Functor f`存放于何处。 如果`_Local_storage`为true， 则直接存放在(使用placement new) `_M_functor` 所在的8个字节处（64位平台），否则则直接`new`一个heap内存，然后用 `_M_functor` 指向这片内存。
3. `_M_get_pointer` 根据`__stored_locally`决定如何取`_Functor`, 如果为true， 直接使用`_source`作为可调用实体（这里使用了`__addressof`, 其作为是即使`_Functor`重载了`&`符号，也能取到实际地址，可参考：[这里](https://en.cppreference.com/w/cpp/memory/addressof)）；否则，直接取`__source`所指向的内存作为 `_Functor`指针。

有了如上知识支撑，再看 `_Function_handler` 的`_M_invoke`函数就很简单：

```c++
 static _Res
        _M_invoke(const _Any_data &__functor, _ArgTypes... __args) {
            return (*_Base::_M_get_pointer(__functor))(__args...);
        }
```

#### `_Function_base::_Ref_manager`

略，该类是针对 引用包装类 而实现的。

### 再看构造函数

```c++
    template<typename _Res, typename... _ArgTypes>
    template<typename _Functor>
    function<_Res(_ArgTypes...)>::
    function(_Functor __f,
             typename __gnu_cxx::__enable_if<
                     !is_integral<_Functor>::value, _Useless>::__type)
            : _Function_base() {
        typedef _Function_handler<_Signature_type, _Functor> _My_handler;

        if (_My_handler::_M_not_empty_function(__f)) {  // 判定传入`__f`是否为空，如函数指针是否为nullptr，lambda表达式则直接返true
            _My_handler::_M_init_functor(_M_functor, __f); // 已介绍过，即将__f存储于_M_functor, 可能直接存在_M_functor底层的8字节，也可能new一个heap内存， 依据为__f的大小和对齐
            _M_invoker = &_My_handler::_M_invoke;   // 赋值静态成员函数，用于调用_M_functor
            _M_manager = &_My_handler::_M_manager;   // 可忽略，也是绑定静态成员函数
        }
    }

```

## operator() 调用

```cpp
bool _M_empty() const { return !_M_manager; }

template<typename _Res, typename... _ArgTypes>
    _Res
    function<_Res(_ArgTypes...)>::
    operator()(_ArgTypes... __args) const
    {
      if (_M_empty())
	_GLIBCXX_THROW_OR_ABORT(bad_function_call());
      return _M_invoker(_M_functor, __args...);
    }

```

很简单，调用`_M_functor`存储的可调用实体。



## 总结

function 源码相对简单，重点关注的有两点：

1. 可调用实体`_Functor`的存储方式，直接存在预分配的8字节 or  new 一个堆
2. ref 相关的调用实现
