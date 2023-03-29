---
title: STL源码阅读（二）： 泛型编程，iterator traits与iterator
categories: stl
date: 2023-03-29 15:34:31
tags:
---

这是stl源码阅读系列的第二篇，这一篇来看看stl中中迭代器traits的实现。

在stl中，有三个彼此关联的组件：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgstl%E5%AE%B9%E5%99%A8-%E8%BF%AD%E4%BB%A3%E5%99%A8-%E7%AE%97%E6%B3%95%E5%85%B3%E8%81%94.svg" alt="stl容器-迭代器-算法关联" style="zoom:150%;" />

Containers(即我们常用的vector, list, deque等)装载数据结构，Algorithms状态数据操作，两者相互独立，Iterator作为中间人，将两者联系起来。iterator提供公有访问容器数据成员的接口，隐藏了每个容器的具体存储实现（即迭代器设计模式）。每个容量都会提供一份iterator实现，所以在分析容器前，先看看iterator。

另外，整个stl可以看做泛型编程的最佳实践，所以在本篇中也会介绍一些泛型（模板）编程的概念。

<!--more-->

## 1. 什么是泛型/模板？

考虑一个场景，如果要实现一个取两个值中的最大值的函数，可以做如下实现：

```cpp
inline int Max(int a, int b) {
	return a < b ? b : a;
}
```

上面对int类型做了实现，如果现在的需求是对`int`,`float`,`double`等类型都实现该函数，若采用函数重载，那必然要写非常多的类似的代码。且每增一种类型，都要重新实现一次。

为了解决这种需求，可以采用泛型编程。如下例子：

```cpp
#include <iostream>
#include <string>
using namespace std;

template <typename T>
inline T const& Max (T const& a, T const& b) {
    return a < b ? b:a;
}

int main () {
    int i = 39;
    int j = 20;
    cout << "Max(i, j): " << Max(i, j) << endl;

    double f1 = 13.5;
    double f2 = 20.7;
    cout << "Max(f1, f2): " << Max(f1, f2) << endl;

    string s1 = "Hello";
    string s2 = "World";
    cout << "Max(s1, s2): " << Max(s1, s2) << endl;

   return 0;
}
```

当上面的代码被编译和执行时，它会产生下列结果：

```cpp
Max(i, j): 39
Max(f1, f2): 20.7
Max(s1, s2): World
```

在编译器编译时，发现调用Max模板函数的类型有 int, double , string, 则会自动生成这种重载函数，**这叫做模板实例化**：

```cpp
template<>
inline const int & Max<int>(const int & a, const int & b)
{
  return a < b ? b : a;
}

template<>
inline const double & Max<double>(const double & a, const double & b)
{
  return a < b ? b : a;
}

template<>
inline const std::basic_string<char> & Max<std::basic_string<char> >(const std::basic_string<char> & a, const std::basic_string<char> & b)
{
  return std::operator<(a, b) ? b : a;
}
```

如此一来，这些重复的工作由编译器实现，c++程序员们只需关注他们的核心代码即可。

## 2. 模板特化

前面对模板作了简单说明，但是现在思考问题，如果说我要对某种类型的模板做专门处理，该如何处理？比如我有个数据结构:

```cpp
struct Student{
	int age;
	int score;
  std::string name;
};
```

现在我想在这个 `Student`上做 `Max` 运算，即比较两个学生的大小，显然之前简单的 `a < b ? b : a`是不支持的，一种解决方式是对 `Student`重载`<`运算符，不过今天要介绍的是另一种方式，模板特化:

```cpp
template<>
inline const Student & Max<Student>(const Student & stu1, const Student & stu2)
{
  return stu1.name < st2.name;
}
```

是不是和前文的编译后的 **模板实例化** 很类似。是的，只不过是我们人为告诉编译器，对于这种`Student`类型，请按照这种方式处理。

实际上，这种方式的全称为 **全特化**, 还有种特化方式，称为 **偏特化**。首先模板参数可以不只一个，全特化代表指定了所有模板参数，而偏特化则是只指定了部分参数。例子如下:

```cpp
#include <iostream>
using namespace std;

// 模板定义
template <typename T1, typename T2>
class MyClass {
public:
    MyClass(T1 a, T2 b) {
        cout << "通用模板" << endl;
    }
};

// 全特化
template <>
class MyClass<int, int> {
public:
    MyClass(int a, int b) {
        cout << "全特化" << endl;
    }
};

// 偏特化
template <typename T1>
class MyClass<T1, int> {
public:
    MyClass(T1 a, int b) {
        cout << "偏特化" << endl;
    }
};

int main() {
    MyClass<double, double> c1(1.0, 2.0); // 输出：通用模板
    MyClass<int, int> c2(1, 2); // 输出：全特化
    MyClass<double, int> c3(1.0, 2); // 输出：偏特化
    return 0;
}
```

## 3. 什么是迭代器设计模式

**迭代器模式（Iterator Pattern）是一种行为设计模式，它允许客户端通过迭代器逐个访问聚合对象的元素，而不需要暴露其底层表示。**下面是一个例子：

```cpp
#include <iostream>
#include <vector>

class IntVector {
public:
    void push(int value) {
        data.push_back(value);
    }

    class Iterator {
    public:
        Iterator(const IntVector& iv) : intVector(iv), index(0) {}
        bool hasNext() const {
            return index < intVector.data.size();
        }
        int next() {
            return intVector.data[index++];
        }
    private:
        const IntVector& intVector;
        size_t index;
    };

    Iterator iterator() const {
        return Iterator(*this);
    }

private:
    std::vector<int> data;
};

int main() {
    IntVector intVector;
    intVector.push(1);
    intVector.push(2);
    intVector.push(3);

    for (IntVector::Iterator it = intVector.iterator(); it.hasNext();) {
        std::cout << it.next() << ' ';
    }
    std::cout << std::endl;

    return 0;
}
```

可以看到，采用Iterator的方式，无需关注IntVector内部是如何存储数据的(它可以是用数组，可以是链表，二叉树等任意数据结构），只用通过 `hasNext()`查看是否还有数据可访问， 通过`next()`得到一个`value`值。

## 4. Iterator Tratis

ok，前面把背景知识都铺垫了，现在开始步入正题。首先stl的每个容器都是模板容器，每个容器也实现了各自的迭代器。现在问一个问题：**算法族函数是如何透过迭代器知道容器内部元素的类型的？** 比如:

```cpp
#include <iostream>
#include <vector>

template<class RandomAccessIterator>
/* 这里该写什么? */ random_one(RandomAccessIterator begin, RandomAccessIterator end)
{
  int dist = end - begin;
  int idx = rand_between(0, dist); // 返回 [0, dist) 中的随机一个值
  return *(begin + idx);
}

int main() {
    std::vector<int> myVector = {1, 2, 3, 4, 5};
    std::cout << random_one(myVector.begin(), myVector.end());
    return 0;
}
```

对于random_one算法来说，它只知道自己接收的是一个iterator类，但是由于不知道Iterator内代表元素类型是什么，我们无法写出注释出的代码（实际上c++ 14/17之后是可以有办法写出的，后文会给出答案，这里专注老版本的写法)，那该怎么做？

### 1. 内嵌类型声明

实际上是利用到 **内嵌类型声明**的trick。听起来挺复杂的，给个例子：

```cpp
class MyClass {
public:
    typedef int MyInt;
};

int main() {
    MyClass::MyInt x = 10;
    std::cout << "x: " << x << std::endl;
    return 0;
}
```

实际上不过是利用了 `typedef` （新版可改用`using`关键字）在类中声明了一个类型罢了。现在把上面的例子换成模板：

```c++
#include <iostream>

template <typename T>
class Iterator {
public:
    typedef T value_type;
};

int main() {
    Iterator<int>::value_type x = 10;
  	std::cout << "x :" << x << std::endl;
    return 0;
}
```

现在 `Iterator<int>::value_type` 可以用来定义 `int`类型对象， `Iterator<Student>::value_type`可以用来定义`Student`类型对象。

有了 **内嵌类型声明**，算法自然就知道返回值类型了:

```cpp
template<RandomAccessIterator>
typename RandomAccessIterator::value_type random_one(RandomAccessIterator begin, RandomAccessIterator end)
{
  int dist = end - begin;
  int idx = rand_between(0, dist); // 返回 [0, dist) 中的随机一个值
  return *(begin + idx);
}
```

### 2. 特化

有了内嵌类型声明，实际上我们的`random_one`算法已经能够处理一些标准容器的迭代器了。但是如果是如下代码：

```cpp
#include <iostream>
#include <vector>

template<RandomAccessIterator>
typename RandomAccessIterator::value_type random_one(RandomAccessIterator begin, RandomAccessIterator end)
{
  int dist = end - begin;
  int idx = rand_between(0, dist); // 返回 [0, dist) 中的随机一个值
  return *(begin + idx);
}

int main() {
    int arr[] = {1, 2, 3, 4, 5};
    std::cout << random_one(arr, arr + 5);
    return 0;
}
```

编译器依然会报错，为什么？因为编译将把`RandomAccessIterator`实例化为 `int *`, 而对于 `int *`这种类型来说，它没有 `value_type`的内嵌类型声明。此时该如何处理。

一种容易想到的解决方案是，使用特例化方式，为 `int *` 类型生成特例化版本的`random_one`算法。但是难道我们要为所有类型都生成一个重载吗? 这也是不现实的。

那stl是如何处理的？ stl的做法是引入了一个中间层，称为 `iterator_traits`

```cpp
template <class _Iterator>
struct iterator_traits {
  typedef typename _Iterator::value_type        value_type;  // 迭代器解除引用后所得到的值的类型
};
```

然后将 `random_one` 改写为：

```cpp
template<class RandomAccessIterator>
typename iterator_traits<RandomAccessIterator>::value_type random_one(RandomAccessIterator begin, RandomAccessIterator end)
{
  int dist = end - begin;
  int idx = rand_between(0, dist); // 返回 [0, dist) 中的随机一个值
  return *(begin + idx);
}
```

**是不是看起来没啥用，还引入了额外复杂度？** 毕竟现在写：

```cpp
int main() {
    int arr[] = {1, 2, 3, 4, 5};
    std::cout << random_one(arr, arr + 5);
    return 0;
}
```

编译器还是会报错。那是因为我只贴出了`iterator_traits`的模板定义，还有个特例化版本：

```cpp
// 针对原生指针(native pointer)而设计的 traits 偏特化版
template <class _Tp>
struct iterator_traits<_Tp*> {
  typedef _Tp                         value_type;
};
```

这种特例化匹配所有指针类型，这样当使用 `random_one(arr, arr + 5);`时， `_Tp` 为 `int`, 此时`value_type`也就为`int`。而如果是非指针类型，则按照普通模板处理，此时`value_type`也代表该类。至此， `random_one` 函数既可以接受`class`类型的迭代器，也可以接受 `pointer`类型的迭代器了。

然而事情还是没有结束，如果现在一个算法为:

```cpp
template<class Iterator>
typename iterator_traits<Iterator>::value_type foo(Iterator iter)
{
	typename iterator_traits<Iterator>::value_type tmp_value;
	tmp_value = *iter;
	return tmp_value;
}
```

这个函数没有什么功能，只是故意写了个`tmp_value`，使它等于迭代器的解引用值，然后返回。考虑如下代码:

```cpp
#include <iostream>
#include <vector>

template <class _Tp> struct iterator_traits {
  typedef _Tp value_type;
};

template <class _Tp> struct iterator_traits<_Tp *> {
  typedef _Tp value_type;
};

template <class Iterator>
typename iterator_traits<Iterator>::value_type foo(Iterator iter) {
  typename iterator_traits<Iterator>::value_type tmp_value;
  tmp_value = *iter;
  return tmp_value;
}

int main() {
  const int arr[]= {1,2,3,4};
  foo(arr);
  return 0;
}

```

此时`typename iterator_traits<Iterator>::value_type`被推导为`const int`, 而` tmp_value = *iter;`这句话是无法完成编译的，`typename iterator_traits<Iterator>::value_type tmp_value;`也必须完成处理化。

那该怎么办？老版本，还是特例化：

```cpp
// 针对原生之 pointer-to-const 而设计的 traits 偏特化版
template <class _Tp>
struct iterator_traits<const _Tp*> {
  typedef _Tp                         value_type;
};

```

完整代码如下:

```cpp
#include <iostream>
#include <vector>

template <class _Tp> struct iterator_traits {
  typedef _Tp value_type;
};

template <class _Tp> struct iterator_traits<_Tp *> {
  typedef _Tp value_type;
};

template <class _Tp>
struct iterator_traits<const _Tp*> {
  typedef _Tp                         value_type;
};

template <class Iterator>
typename iterator_traits<Iterator>::value_type foo(Iterator iter) {
  typename iterator_traits<Iterator>::value_type tmp_value ;
  tmp_value = *iter;
  return tmp_value;
}

int main() {
  const int arr[]= {1,2,3,4};
  foo(arr);
  return 0;
}
```

Ok. 支持，一个能够handle各种情况的`iterator_traits`完成了。来回忆一遍：

1. 为了让算法知道iterator解引用后的类型，引入了 **内嵌类型声明**
2. 为了解决指针类型无内嵌类型声明问题，引入了中间层`iterator_traits`和指针版本特例化。
3. 为了解决`const pointer`所带来的问题，引入了`const pointer`特例化。

这就是`iterator_traits`, 借用侯捷老师的图再次说明其作用。

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgv2-aedff7a9a120c1653a4817cfdf015611_1440w.jpg)

### 3.stl iterator traits定义

前面介绍完了`iterator traits`的由来，来看看stl内是怎么定义的吧。

```cpp
// traits 获取各个迭代器的特性(相应类型)-----类型特性类
template <class _Iterator>
struct iterator_traits {
  typedef typename _Iterator::iterator_category iterator_category; // 迭代器类别
  typedef typename _Iterator::value_type        value_type;  // 迭代器解除引用后所得到的值的类型
  typedef typename _Iterator::difference_type   difference_type; // 两个迭代器之间的距离
  typedef typename _Iterator::pointer           pointer;      // 指向被迭代类型的指针
  typedef typename _Iterator::reference         reference;   // 被迭代类型的引用类型
};

// 针对原生指针(native pointer)而设计的 traits 偏特化版
template <class _Tp>
struct iterator_traits<_Tp*> {
  typedef random_access_iterator_tag iterator_category;
  typedef _Tp                         value_type;
  typedef ptrdiff_t                   difference_type;  // C++ 内建的 ptrdiff_t 类型
  typedef _Tp*                        pointer;
  typedef _Tp&                        reference;
};

// 针对原生之 pointer-to-const 而设计的 traits 偏特化版
template <class _Tp>
struct iterator_traits<const _Tp*> {
  typedef random_access_iterator_tag iterator_category;
  typedef _Tp                         value_type;
  typedef ptrdiff_t                   difference_type;
  typedef const _Tp*                  pointer;
  typedef const _Tp&                  reference;
};

```

看起来多了不少内嵌类型声明。分别解释下这几个类型声明：

- `value_type`：迭代器所指对象的类型，原生指针也是一种迭代器，对于原生指针 `int*`，`int` 即为指针所指对象的类型，也就是所谓的 `value_type` 。

- `difference_type`： 用来表示两个迭代器之间的距离，对于原生指针，STL 以 C++ 内建的 `ptrdiff_t` 作为原生指针的 difference_type。

- `reference`： 是指迭代器所指对象的类型的引用，`reference` 一般用在迭代器的 * 运算符重载上，如果 `value_type` 是 `T`，那么对应的 `reference` 就是 `T&`；如果 `value_type` 是 `const T`，那么对应的reference_type 就是 `const T&`。

- `pointer`： 就是相应的指针类型。

- `iterator_category`： 的作用是标识迭代器的移动特性和可以对迭代器执行的操作，从 `iterator_category` 上，可将迭代器分为 `Input Iterator、Output Iterator、Forward Iterator、Bidirectional Iterator、Random Access Iterator` 五类，**为什么需要定义这么多种类的迭代器？主要是为了算法可以针对特定的迭代器使用特定的算法，从而提高算法效率**。在stl中，分别定义为：

  ```cpp
  // iterator_category 五种迭代器类型
  // 标记
  struct input_iterator_tag {};
  struct output_iterator_tag {};
  struct forward_iterator_tag : public input_iterator_tag {};
  struct bidirectional_iterator_tag : public forward_iterator_tag {};
  struct random_access_iterator_tag : public bidirectional_iterator_tag {};
  
  // The base classes input_iterator, output_iterator, forward_iterator,
  // bidirectional_iterator, and random_access_iterator are not part of
  // the C++ standard.  (They have been replaced by struct iterator.)
  // They are included for backward compatibility with the HP STL.
  
  template <class _Tp, class _Distance> struct input_iterator {
    typedef input_iterator_tag iterator_category;
    typedef _Tp                value_type;
    typedef _Distance          difference_type;
    typedef _Tp*               pointer;
    typedef _Tp&               reference;
  };
  
  struct output_iterator {
    typedef output_iterator_tag iterator_category;
    typedef void                value_type;
    typedef void                difference_type;
    typedef void                pointer;
    typedef void                reference;
  };
  
  template <class _Tp, class _Distance> struct forward_iterator {
    typedef forward_iterator_tag iterator_category;
    typedef _Tp                  value_type;
    typedef _Distance            difference_type;
    typedef _Tp*                 pointer;
    typedef _Tp&                 reference;
  };
  
  
  template <class _Tp, class _Distance> struct bidirectional_iterator {
    typedef bidirectional_iterator_tag iterator_category;
    typedef _Tp                        value_type;
    typedef _Distance                  difference_type;
    typedef _Tp*                       pointer;
    typedef _Tp&                       reference;
  };
  
  template <class _Tp, class _Distance> struct random_access_iterator {
    typedef random_access_iterator_tag iterator_category;
    typedef _Tp                        value_type;
    typedef _Distance                  difference_type;
    typedef _Tp*                       pointer;
    typedef _Tp&                       reference;
  };
  
  ```

  简单说明这几个 iterator 的用途与区别：

  - `Input Iterator`： 此迭代器不允许修改所指的对象，是只读的。支持 ==、!=、++、*、-> 等操作。
  - `Output Iterator`：允许算法在这种迭代器所形成的区间上进行只写操作。支持 ++、* 等操作。
  - `Forward Iterator`：允许算法在这种迭代器所形成的区间上进行读写操作，但只能单向移动，每次只能移动一步。支持 Input Iterator 和 Output Iterator 的所有操作。
  - `Bidirectional Iterator`：允许算法在这种迭代器所形成的区间上进行读写操作，可双向移动，每次只能移动一步。支持 Forward Iterator 的所有操作，并另外支持 – 操作。
  - `Random Access Iterator`：包含指针的所有操作，可进行随机访问，随意移动指定的步数。支持前面四种 Iterator 的所有操作，并另外支持 [n] 操作符等操作。

  一张图总结：

  ![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img011.png)



## 5. iterator

说了这么多的iterator traits, 最终都是为了`iterator`服务，stl的`iterator`定义为：

```cpp
// 用户自己的写的迭代器最好继承此 std::iterator
template <class _Category, class _Tp, class _Distance = ptrdiff_t,
          class _Pointer = _Tp*, class _Reference = _Tp&>
struct iterator {
  typedef _Category  iterator_category;
  typedef _Tp        value_type;
  typedef _Distance  difference_type;
  typedef _Pointer   pointer;
  typedef _Reference reference;
};
```

这是迭代器的基类， 以看到这里就定义了`iterator_traits`中要使用的各种类型，符合前文的分析。同时注意，`Distance = ptrdiff_t, class _Pointer = _Tp*, class _Reference = _Tp&`这三个模板是有默认值的。所以之后使用时，可以只传入 `_Category`与`_Tp`即可。



## 6. 总结

本文介绍了什么是模板，模板特例化，同时从 **算法从如何透过迭代器知道迭代器解引用后的元素类型** 问题出发，一步步引出了 `iterator traits`的实现。



## 补充

前文提到了如下模板在c++ 14/17后可以直接书写出来:

```cpp
template<class RandomAccessIterator>
/* 这里该写什么? */ random_one(RandomAccessIterator begin, RandomAccessIterator end)
{
  int dist = end - begin;
  int idx = rand_between(0, dist); // 返回 [0, dist) 中的随机一个值
  return *(begin + idx);
}
```

答案是：

```cpp
#include <iostream>
#include <vector>

template<class RandomAccessIterator>
auto random_one(RandomAccessIterator begin, RandomAccessIterator end) -> decltype(*begin)
{
  int dist = end - begin;
  /int idx = rand_between(0, dist); // 返回 [0, dist) 中的随机一个值
  return *(begin + 0);
}
```

