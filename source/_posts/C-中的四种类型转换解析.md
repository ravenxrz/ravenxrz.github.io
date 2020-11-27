---
title: C++中的四种类型转换解析
categories: 编程语言
tags: C++
abbrlink: 714c450c
date: 2020-11-27 16:43:17
---

在阅读《Effective C++》的条款27：尽量少做转型动作时，书中对C++自带的四种转型做了解释，但是感觉还不是很理解，同时之前在leveldb中也看到了大量的转换代码。故google了一波加上自己做了点测试，写下本文做个记录。

**本文试图解释为什么C++要提供4种cast。**

主要翻译自：[How do you explain the differences among static\_cast, reinterpret\_cast, const\_cast, and dynamic\_cast to a new C++ programmer?](https://www.quora.com/How-do-you-explain-the-differences-among-static_cast-reinterpret_cast-const_cast-and-dynamic_cast-to-a-new-C++-programmer#:~:text=static_cast%20performs%20implicit%20conversions%2C%20the,their%20numerical%20(integer)%20values.)

<!--more-->

首先看看C Style的转型：

```c++
 (T) expression // 将expression转型为T
 或
 T(expression)	// 将expression转型为T
```

这种转型可能包含以下几种类型之间的转换：

1. 两个算数类型，如int到float，float到int等。
2. 指针到整形，整形到指针。
3. 指针到指针。
4. 常量类型到非常量类型
5. (4)和(1)或(2)或(3)的结合。

C++增加了继承和模板两种方式，使得C style的转型不太合适。

为什么？**先看继承**，假设D派生自B， 则 (DB*)pb会有哪些场景,（pb是指向B的指针）：

1. 返回一个指针，指向内存中**相同字节地址**，仅仅改变指针的类型。 （在C的指针转换就是这样的）
2. **检查**B*是否真的指向了一个D对象中的B部分。如果是，则返回一个指向D对象的指针，如果不是，则失败，返回一个null或抛出异常。
3. **直接假设**B*指针是指向D对象中的B部分，**并不作任何调整与检查**，如果有必要，调整指针的地址，使得它的确指向D对象（特别是在多重继承里，这几乎是肯定会调整指针地址的）。

为了做这三种转换，C++提供了三种cast：

1. 情况1， reinterpret_cast<D*>
2. 情况2，dynamic_cast<D*>
3. 情况3， static_cast<D*>

现在来看模板，因为转型时可能会依赖模板，如下：

```c++
template<typename T>
unsigned char* alias(T& x){
    return (unsigned char*)(&x);
}
```

咋一看似乎没什么，但是如果我传递的是一个常量呢？ 那么上面的转换会直接脱除常量性，导致使用这个函数返回的指针时，可以透过指针修改的value，这显然是不合理的。 C Style的cast可能改变类型，可能改变常量性，也可能同时改变两种。所以，在C++中可以这样写：

```c++
template <class T> 
unsigned char* safe_alias(T& x) { 
    return reinterpret_cast<unsigned char*>(&x); 
} 
```

这样，当客户传递一个const value，编译器会直接报错的，因为reinterpret_cast去除常量性。为了解决这个问题，C++提出了 const_cast ， 一个专门用于常量脱除的cast。

除了以上原因，由于模板的存在，有时候我们也必须显示转换类型，如下C代码：

```c
int* pi; 
void* pv = pi; // OK; implicit conversion 
void* pv2 = (void*)pi; // OK but unnecessary 
void f(void* pv); 
f(pi); // OK; implicit conversion 
f((void*)pi); // OK but unnecessary 
```

但是，在C++中，可能是这样的：

```c++
template <class T> void f(T* p); 
// ... 
int* pi; 
f(pi); // OK; calls f<int> 
f(static_cast<void*>(pi)); // OK; calls f<void> 
```

为了确保调用的是正确的类型(void *)， 有时候我们需要手动转换，而不是由编译器来决定。

到这儿，我们做个总结：

- static_cast 执行隐式转换，或者base到derived的转换（但是是不安全的，因为没有检查）
- reinterpret_cast 将一个指针转到另一个指针，同时不修改任何地址，同时可以将指针转换为数值。
- const_cast 只会用到常量性脱除上。
- dynamic_cast 可以向上或向下转型，并且检查转换是否成功。