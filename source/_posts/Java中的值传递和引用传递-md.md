---
title: Java中的值传递和引用传递
date: 2018-04-22 13:58:07
categories: 编程语言
toc: true
tags:
---


## 0.前言

被java中的“值传递”和“引用传递”困扰过一阵子，在实际代码中也犯过不少错，记录一下，方便查看。

<!-- more -->
## 1.问题

首先看看代码：

```java
public class Test {
	 public static void main(String[] args) {
	        String str = "123";
	        System.out.println(str);
	        change(str);
	        System.out.println(str);
	    }
    public static void change(String str){
    		str = "456";
    }
}
```

那么你觉得会输出多少呢？至少我曾经觉得是：

```
123
456
```

但是，正确答案是：

```
123
123
```

这是为什么呢？我相信答错的同学大都是受到了一些”java教材“的影响--java的参数传递有两种：

1. 值传递，传递值，在函数中形参发生的变化不影响实参。
2. 引用传递，传递对象引用，在函数中形参发生的变化影响实参。

**然而，实际上java参数传递只有一种情况，那就是值传递。所不同的是，一般说的"引用传递"，在实际中传递的不过是引用对象的地址值**

在解释上述代码前，先要在补充一点知识：

```
String a = new String("123");
String b;
b= new String("123");
```

两种形式的代码所形成的的结果是完全一致的，后面一种更容易理解java中的引用与对象的具体含义。**先声明一个String对象的引用，再new一个“123”对象，最后将这个对象赋值（等号=）给该引用**。

1. b：对象的引用
2. “123”：实际对象

好了，现在来具体解释一下**值传递**和**地址值**引用吧。

#### 1.值

```
int a = 1;
void f(int b){
  b =2;
}
```

a是多少呢？肯定是1嘛。

![image.png](https://pic.superbed.cn/item/5cfbb5df451253d178d9d315.png)

将a的值1传递给b，b的值为1.这样不论b如何更改，并不影响a的值。

#### 2.地址值

```
String a = new String("123";)
void main(String b){
  b = new String("345");
}
```


![image.png](https://pic.superbed.cn/item/5cfbb5e0451253d178d9d34b.png)
首先，**a将对象“123”的地址值传递给b，b指向"123"的地址**。

之后 new了一个"345"对象，**b重新指向了"345"**，也就是下图。
![image.png](https://pic.superbed.cn/item/5cfbb5e2451253d178d9d37a.png)
所以，仅仅是b的引用对象发生了变化，a的引用对象毫无改变。
**现在能够解释文首的问题了。**But,there is one more thing.

如下代码：

```
public class Test {
	 public static void main(String[] args) {
		 StringBuilder a = new StringBuilder("123");
		 change(a);
		 System.out.println(a);
	    }
    public static void change(StringBuilder str){
    		str.append("345");
    }
}

```

现在，a的值是多少？

答案是:

```
123345
```

这也就是平常我们所理解的，传递“引用”会影响原对象本身。还是画个图来解释：
![image.png](https://pic2.superbed.cn/item/5cfbb5e8451253d178d9d3f4.png)

首先a将“123”对象的地址传递给b，b指向“123”，接着通过“绿色”引用改变了“123”对象。因为a也是指向“123”对象，所以输出a也变为了"123345"。



That's all.Hope you have understood。


