---
title: 理解C++中的左值(lvalue)和右值(rvalue)
categories: 编程语言
toc: true
thumbnail: /thumbnails/desert.jpg
abbrlink: c49b3e12
date: 2019-10-13 13:59:01
tags:
	- C++
---

i> 翻译自:https://www.internalpointers.com/post/understanding-meaning-lvalues-and-rvalues-c

我一直苦于理解C++中的左值和右值。现在我感觉是时候分享出我对它们的理解了，因为它们在C++语言发展中扮演者越来越重要的角色。

一旦你理解了lvalues和rvalues的含义，你就可以更深入地研究高级c++特性，比如move语义和右值引用。

<!-- more -->

## 1.Lvalues and rvalues: a friendly definition

在C++中，**一个lvalue指的就是一个特定的内存位置**，**而rvalue则不指向任意位置**。通常，rvlaue是一种临时的变量却其生命周期较短，而lvalue则相反，lvalue具有更长的生命周期，因为它们以内存空间的变量的形式存在。你也可以把lvalue理解成容器，rvalue理解成容器中包含的东西，如果没有容器，rvalue很快就dead了。

现在来举点例子吧：

```c++
int x = 666;   // ok
```

这里**666是一个rvalue;** 数字(字面常量)没有特定的内存地址，它们仅仅在程序运行时使用了一些临时寄存器。这个数字被赋值给x, x是一个变量。**变量**有一个特定的内存位置，所以它**是一个lvalue**。c++声明一个赋值需要一个lvalue作为其左操作数。

x是一个lvalue，现在你可以这样做：

```c++
int* y = &x;   // ok
```

这里我获取了x的内存地址，并通过地址操作符&将其放入y中。地址操作符&接受一个lvalue参数并生成一个rvalue。而y则是一个指针型变量（占用内存空间），所以y是一个右值。

但是，我们不能像这样做：

```c++
int y;
666 = y; // error!
```

这当然是显然的啦。源于在于666只是一个字面常量--所以它是一个rvalue，没有一个特定的内存空间。既然没有内存空间，那我们把y的值赋值到哪儿呢？当然也就不行咯。

如果我们运行上述代码，gcc将提示我们下面的错误：

```c++
error: lvalue required as left operand of assignment
```

He is damn right.(我不知道怎么翻译，才有这种感觉，哈哈哈~~~)；左侧赋值语句必须要一个lvalue才行，但是在这里我使用了一个rvalue 666.

我也可以这样做。

```c++
int *y  = &666;
```

GCC 报错：

```c++
error: lvalue required as unary '&' operand
```

He is right agan. "&"操作符需要一个lvalue作为输入，因为&只能处理具有内存地址的量。

## 2. Functions returning lvalues and rvalues

我们都知道左赋值语句必须要一个lvalue。所以，如果像下面这个函数写，一定会报错：`lvalue required as left operand of assignment`:

```c++
int setValue()
{
    return 6;
}

// ... somewhere in main() ...

setValue() = 3; // error!
```

setValue() 返回了一个rvalue(临时数字6)，它不能用于左赋值语句。那么如果我们的函数返回一个lvalue呢？如下：

```c++
int global = 100;

int& setGlobal()
{
    return global;    
}

// ... somewhere in main() ...

setGlobal() = 400; // OK
```

这样就能正常工作了，因为setGlobal()返回了一个引用。引用就是指向内存空间（这里是global）的东西，所以引用能够作为lvalue，当然也就能用于赋值。

i> 没有引用符&，上述代码是不行的。这也是为什么C++的左值右值理解要比C语言难的原因之一。你永远不会在C语言中看到函数调用写在等号左边吧。

## 3. Lvalue to rvalue conversion

有事，lvalue是可以转换为rvalue的。我们以“+”来距离。“+”以两个rvalue作为参数，并返回一个rvalue。

如：

```c++
int x = 1;
int y = 2;
int z = x + y; //ok
```

等下，这里的x和y是lvalue啊，但是"+"需要的是两个rvalue作为参数。答案很简单，x和y都被隐式的转换为了rvalue。还有很多其它的操作符也有类似的转换-- "-","*","/"等。

## 4. Lvalue references

既然lvalue可以转化为rvalue，但rvalue可以转化为lvalue吗？Nope。

在C++，你可以这样做：

```c++
int y = 10;
int& yref = y;
yref++;        // y is now 11
```

你声明了一个引用yref，并指向了y，yref是一个lvalue。我们都知道一个引用必须指向一个有内存空间的变量，也即lvalue。这里y的确是有内存空间的变量，所以上述代码没有问题。

现在，我们考虑直接将一个字面常量10直接赋值给我们的引用呢？

```c++
int &yref = 10;
```

在等号的右边，我们有一个rvalue，rvalue是没有内存空间的，他只是一种临时变量。而yref必须指向具有内存空间的变量，这就产生了冲突。

也许你会这样想，为了使用引用，一个易失的字面常量（rvalue）可以自动转化为lvalue才对。如果语言允许这样做，那之后你就可以通过引用来改变一个rvalue。但是rvalue是临时的，是易失的，那一旦rvalue消失了，引用又指到哪儿呢？

下面的代码会因为相同的原因而无法编译通过：

```c++
void fnc(int& x)
{
}

int main()
{
    fnc(10);  // Nope!
    // This works instead:
    // int x = 10;
    // fnc(x);
}
```

我们将一个临时rvalue 10 传递给一个函数，这个函数使用了引用作为形参。**rvalue是没法自动转化为lvalue的。**注释的代码是可以通过的，因为已经在10赋值给了一个lvalue x。

## 5. Const lvalue reference to the rescue

上面的代码片段在编译时，会报如下错误：

```c++
error: invalid initialization of non-const reference of type 'int&' from an rvalue of type 'int'
```

GCC指出这个常量不是一个const。难道将因为增添const修饰就可以通过了？的确是这样。根据C++语言规范，你的确可以将一个const lvalue绑定到一个rvalue上。所以下面这两段代码可以通过：

```c++
const int &ref = 10;
```

```c++
void fnc(const int& x)
{
}

int main()
{
    fnc(10);  // OK!
}
```

这背后的思想相当直接。字面常量10是易失的，所以使用可修改值的引用是没有意义的。但是当引用无法修改rvalue的value，那就可以了。所以增添了const修饰的引用是可以指向一个rvalue的。但其实也不算真正的指向了一个rvalue，只是我们可以这样使用。主要的原因是编译器帮我们创建了隐式的lvalue变量。如：

```c++
// the following...
const int& ref = 10;

// ... would translate to:
int __internal_unique_name = 10;
const int& ref = __internal_unique_name;
```

现在你可以正常的使用引用了。当然了，这种情况下你是没办法通过引用更改指向的值的。

```c++
const int& ref = 10;
std::cout << ref << "\n";   // OK!
std::cout << ++ref << "\n"; // error: increment of read-only reference ‘ref’
```



## 6. Conclusion

最后一段，不打算翻译。

Understanding the meaning of lvalues and rvalues has given me the chance to figure out several of the C++'s inner workings. C++11 pushes the limits of rvalues even further, by introducing the concept of rvalue references and move semantics, where — surprise! — rvalues too are modifiable. I will restlessly dive into that minefield in one of my [next articles]( https://www.internalpointers.com/post/c-rvalue-references-and-move-semantics-beginners ).

## Sources

Thomas Becker's Homepage \- *C++ Rvalue References Explained* ([link](http://thbecker.net/articles/rvalue_references/section_01.html))
Eli Bendersky's website \- *Understanding lvalues and rvalues in C and C++* ([link](http://eli.thegreenplace.net/2011/12/15/understanding-lvalues-and-rvalues-in-c-and-c))
StackOverflow \- *Rvalue Reference is Treated as an Lvalue?* ([link](http://stackoverflow.com/questions/28483250/rvalue-reference-is-treated-as-an-lvalue))
StackOverflow \- *Const reference and lvalue* ([link](http://stackoverflow.com/questions/22845167/const-reference-and-lvalue))
CppReference.com \- *Reference declaration* ([link](http://en.cppreference.com/w/cpp/language/reference))
