---
title: java函数接口&方法引用
tags: 
    - java
    - 函数接口
    - 方法引用
categories: java
abbrlink: 6bef376f
date: 2020-04-10 13:27:38
categoires: 编程语言
---

相信不少同学在学习java 1.8新特性时，都对java函数接口和方法引用有些困惑。在查询资料加上一些自己的理解后整理出本文，希望能够帮过理清这其中的“奥秘”。

<!-- more -->

## 1. 追追历史

一个东西的诞生是不会没有理由的。那为什么要使用函数接口呢？

我们都知道在jdk1.8之前，要让线程run一个task，我们可以通过一个匿名内部类来实现，如：

```java
// Java program to demonstrate functional interface 
  
class Test { 
    public static void main(String args[]) { 
        // create anonymous inner class object 
        new Thread(new Runnable() { 
            @Override
            public void run() { 
                System.out.println("New thread created"); 
            } 
        }).start(); 
    } 
}
```

可以看到，我们仅仅是为了让线程打印一条语句，却写了这么多行代码。有什么办法可以简化它呢？于是，**java引入了lambda表达式：**

```java
class Test {
    public static void main(String args[]) {
        // lambda expression to create the object 
        new Thread(() ->{
            System.out.println("New thread created");
        }).start();
    }
} 

```

现在代码看着舒服多了。不仅是代码行数的减少，也更直观的阐述了这个线程的工作任务。

**那我们想一下，什么情况下可以用lambda表达式来替代一个匿名内部类呢？**

答案是：如果如果这个匿名内部类只需要实现一个抽象方法。

## 2. 函数接口

紧跟前面的答案，如果一个方法所以来的接口只有一个抽象方法，那么我们就可以用lambda表达式去替代它。这些**“只有一个抽象方法”的接口**（这个说明其实略有出入，后文会有解释，现在这样就可以了）就是**函数接口**。并且，jdk中还提供了一个注解--@FunctionalInterface，用来编译器这个接口是函数接口。当然，只要接口满足函数接口的条件，即使不加这个注解，编译器还是会认为这个接口就是函数接口。

函数接口的关键：

1. 一个函数接口只能由一个抽象方法，但是可以有多个默认实现的方法。
2. @FunctionalInterface注解用来确保接口不能有多尔衮抽象方法。但是一个函数接口不必一定添加这个注解。
3. 函数接口可用lambda表达式或者方法引用来替代。

### 2.1 JDK中的函数接口包

JDK的函数接口包中大体上可以分成4类：

- Consumer： 消费器
- Supplier： 生产器
- Predicate： 判断器
- Function： 转换器

#### 2.1.1 Consumer

Consumer接口一个参数，并且返回一个void（即不返回），所以叫做“消费”。

其内部包含两个方法：

- accept 抽象方法
- andThen 默认实现方法

下面举个例子：

```java
public class ConsumerFunctionExample{
   public static void main(String args[]){
      List<Integer> integerList= Arrays.asList(1, 10, 200, 101, -10, 0);
      printList(integerList, (v -> System.out.println(v)));// 此处的lambda表达式，代替了printList方法中的函数接口
   }

   public static void printList(List<Integer> listOfIntegers, Consumer<Integer> consumer{
      for(Integer integer:listOfIntegers){
         consumer.accept(integer);
      }
   }
}
```

又或者：

```java
public class ConsumerFunctionExample{
   public static void main(String args[]){
      // 引用一个lambda表达式
      Consumer<Integer> consumer= i-> System.out.print(i);
      List<Integer> integerList= Arrays.asList(1, 10, 200, 101, -10, 0);
      printList(integerList, consumer);
      
   }

   public static void printList(List<Integer> listOfIntegers, Consumer<Integer> consumer){
      for(Integer integer:listOfIntegers){
         consumer.accept(integer);
      }
   }
}
```

含义一样。

**BiConsumer**，功能和Consumer一样，只不过Consumer吃一个参数，Biconsumer吃两个参数，然后两个接口都不吐任何东西。

#### 2.1.2 Supplier

和Consumer相反，Supplier不吃”任何东西“，返回给我们提供东西。所以它的内部只有一个get方法。

举个例子：

```java
public class SupplierFunctionExample {
    public static void main(String args[]) {

        //Supplier instance with lambda expression
        Supplier<String> helloStrSupplier = () -> new String("Hello");
        String helloStr = helloStrSupplier.get();
        System.out.println("String in helloStr is->" + helloStr + "<-");

        //Supplier instance using method reference to default constructor
        Supplier<String> emptyStrSupplier = String::new;
        String emptyStr = emptyStrSupplier.get();
        System.out.println("String in emptyStr is->" + emptyStr + "<-");

        //Supplier instance using method reference to a static method
        Supplier<Date> dateSupplier = SupplierFunctionExample::getSystemDate;
        Date systemDate = dateSupplier.get();
        System.out.println("systemDate->" + systemDate);
    }

    public static Date getSystemDate() {
        return new Date();
    }
}
```

#### 2.1.3 Predicate

predicate接受一个input，然后返回一个boolean。通过会把这个input用来做某种条件判断。如：

```java
public class PredicateFunctionExample {
    public static void main(String args[]) {
        Predicate<Integer> positive = i -> i > 0;	// 条件判定的实体
        List<Integer> integerList;
        integerList = Arrays.asList(1, 10, 200, 101, -10, 0);
        List<Integer> filteredList = filterList(integerList, positive);
        
        filteredList.forEach(System.out::println);
    }

    public static List<Integer> filterList(List<Integer> listOfIntegers, Predicate<Integer> predicate) {
        List<Integer> filteredList = new ArrayList<Integer>();

        for (Integer integer : listOfIntegers) {
            if (predicate.test(integer)) {	
                filteredList.add(integer);
            }
        }
        return filteredList;
    }
}
```

**BiPredicate** ，接收两个参数来做判别。

#### 2.1.4 Function

Function函数接口，接受一个参数然后返回一个参数（注意接收和返回可以是相同的类型，也可以是不同的类型），我个人将其称作为转换器。

举个例子:

```java
public class FunctionExample {
    public static void main(String[] args) {
        Function<Integer,Double> converter = (t) -> (double)t;
        List<Integer> integerList = Arrays.asList(1, 10, 200, 101, -10, 0);
        // convert integer type to double type
        List<Double> doubleList = listConverter(integerList,converter);
        doubleList.forEach(System.out::println);
    }

    public static <T,U> List<U> listConverter(List<T> tList, Function<T,U> converter){
        List<U> uList = new ArrayList<>();
        for(T t: tList){
            uList.add(converter.apply(t));
        }
        return uList;
    }
}
```

现在，我们看看它的定义：

```java
@FunctionalInterface
public interface Function<T, R> {

    /**
     * Applies this function to the given argument.
     *
     * @param t the function argument
     * @return the function result
     */
    R apply(T t);
    ...
}
```

注意到Function的泛型T，R是只能接受**类类型**的。那如何接收基本类型呢(int double long等)。我们这样写吗：

```java
Function<int,double> function; // 显然是错的
```

考虑到这个问题，jdk里面给我们提供了一些方便接口：

- 基本类型转其它类型：命名方式：基本类型Function。如IntFunction,DoubleFunction等，就是接受一个int或double参数，然后一个泛型T类型。

  ```java
  @FunctionalInterface
  public interface IntFunction<R> {
  
      /**
       * Applies this function to the given argument.
       *
       * @param value the function argument
       * @return the function result
       */
      R apply(int value);
  }
  ```

- 其它类型转基本类型：命名方式：To基本类型Function。如ToIntFunction,ToDoubleFunction等。接受其它类型转为基本类型。

  ```java
  @FunctionalInterface
  public interface ToIntFunction<T> {
  
      /**
       * Applies this function to the given argument.
       *
       * @param value the function argument
       * @return the function result
       */
      int applyAsInt(T value);
  }
  
  ```

#### **额外的命名规则：**

jdk中的接口还有额外一种命名规则，叫做从左向右指定形参的参数类型。

如： ObjIntConsumer,其实Obj表示第一个参数是一个泛型参数，第二个参数时int：

```java
@FunctionalInterface
public interface ObjIntConsumer<T> {

    /**
     * Performs this operation on the given arguments.
     *
     * @param t the first input argument
     * @param value the second input argument
     */
    void accept(T t, int value);
}
```

又如， `IntToDoubleFunction`.表明Function的apply参数，参数时int，返回值是double。

### 2.2 另一些函数接口

除了jdk，java.util.function包下的函数接口外，还有一些函数也是函数接口。如本文最开始提到的：

Runnable,还有ActionListener, Comparable，Callable等，这些接口无一例外都只有一个抽象函数。

但是我在上文中曾提到，这样的说法其实是一点出入的。我们来看Comparator这个接口：

```java
@FunctionalInterface
public interface Comparator<T> {

    int compare(T o1, T o2);

    boolean equals(Object obj);
    
    ...
}
```

可以明显的看到，这个接口是有两个抽象函数的，但是它也用了@FunctionalInterface注解表明。

所以，这个说法是有一点出入，就我个人而言，应该是**在使用时能够使用labmda式或者方法引用去完全匹配一个接口中的某个抽象方法**，就算是函数接口。

如：

```java
public class ComparatorExample {
    public static void main(String[] args) {
        Student[] stus = new Student[]{
                new Student(12,"法外狂徒张三"),
                new Student(123,"李四"),
                new Student(4,"王五")
        };
       	// 此处的lambda表达完全配置Comparator#compare方法
        Comparator<Student> comparator = (stu1,stu2)->{
            return stu1.getId() - stu2.getId();
        };
        Arrays.sort(stus,comparator);
        // print result
        Stream.of(stus).forEach(System.out::print);
    }
}

class Student{
    private int id;
    private String name;
    
    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Student{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }

    public Student(int id, String name) {
        this.id = id;
        this.name = name;
    }
}
```

下面我们再来谈谈和函数接口紧密相连的方法引用。

## 3. 方法引用

### 3.1 为什么要用方法引用

让我们再看看刚才那个根据学号排序学生的例子，只不过这次我在student类中添加一个新的方法。

```java
public class ComparatorExample {
    public static void main(String[] args) {
        Student[] stus = new Student[]{
                new Student(12,"法外狂徒张三"),
                new Student(123,"李四"),
                new Student(4,"王五")
        };
        Comparator<Student> comparator = (stu1,stu2)->{
            return Student.compareTo(stu1,stu2);
        };
        Arrays.sort(stus,comparator);
        
        Stream.of(stus).forEach(System.out::print);
    }
}

class Student{
    private int id;
    private String name;

    public static int compareTo(Student stu1, Student stu2){
        return stu1.getId() - stu2.getId();
    }

	...
}

```

这样同样能实现效果，现在我们仅看这一句话：

```java
        Comparator<Student> comparator = (stu1,stu2)->{
            return Student.compareTo(stu1,stu2);
        };
```
这一个labmda表达式非常简单，**当lambda内部出了调用一个函数以外什么也不做时，我们就可以考虑使用方法引用。**

使用方法引用来改造上面的例子：

```java
public class ComparatorExample {
    public static void main(String[] args) {
        Student[] stus = new Student[]{
                new Student(12,"法外狂徒张三"),
                new Student(123,"李四"),
                new Student(4,"王五")
        };
        Comparator<Student> comparator = Student::compareTo;
        Arrays.sort(stus,comparator);
        // 或者直接： Arrays.sort(stus,Student::compareTo);
        // print result
        Stream.of(stus).forEach(System.out::print);
    }
}
```

此时Student::compareTo等价于lambda表达式：

```java
 (stu1,stu2)->{
            return Student.compareTo(stu1,stu2);
        };
```

仔细思考一下它们之间的相似点。

yes，**他们的参数是相同**（数量，类型）。

上面我所展示的只不过是3种方法引用的其中一种，这3种分别是：

1. Reference to a static method. 静态方法引用
2. Reference to an instance method. 实例方法引用
3. Reference to a constructor. 构造器引用

![Types of Java Method References](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/types-of-method-references.jpg)



别被三种方式吓到了，看到**总结**后，你会发现其实很好区别。

### 3.2 静态方法引用

语法：

```
ContainingClass::staticMethodName  
```

现在我们来解释一下之前学生排序的例子：

```java
Comparator<Student> comparator = Student::compareTo;
Arrays.sort(stus,comparator);
// 等价于
Arrays.sort(stus,Student::compareTo);
```

我们来看看Arrays.sort的方法签名：

```java
public static <T> void sort(T[] a, Comparator<? super T> c)
```

Compator这里作为函数接口，它的签名是：

```java
int compare(T o1, T o2);
```

记住，它是两个参数。

再看看Student::compareTo,

```java
public static int compareTo(Student stu1, Student stu2)
```

也是两个参数。

也就是说静态方法引用，只要函数接口和静态方法的参数一一对应即可。

### 3.3 实例方法引用

实例方法引用又分两种：

#### 3.3.1 特定实例的方法引用

语法：

```java
containingObject::instanceMethodName  
```

1. 简单例子：

```java
class Filter{
    public boolean startWithB(String str){
        return str.startsWith("B");
    }
}

List<String> list = Arrays.asList( "Barbara", "James", "Mary", "John",
        "Patricia", "Robert", "Michael", "Linda");
Filter myFilter = new Filter();
List<String> collect = list.stream().filter(myFilter::startWithB).collect(Collectors.toList());
collect.forEach(System.out::println);
```

这里有两个地方都用到了实例方法引用：

1. list.stream().filter(myFilter::startWithB)。 myFilter::startWithB
2. collect.forEach(System.out::println);  

这里进说明第一个：

Filter是自定义的类，内部有一个方法。签名：

```java
 public boolean startWithB(String str)
```

filter:

```java
Stream<T> filter(Predicate<? super T> predicate);

public interface Predicate<T> {
    boolean test(T t);
}
```

所以也就是test(T t)中t传给了startWithB（String str)中的str。

#### 3.3.2 引用特定类型的任意对象的实例方法

**这是方法引用中最难理解的一种。**

现在看这个例子：

```java
String[] stringArray = { "Barbara", "James", "Mary", "John",
    "Patricia", "Robert", "Michael", "Linda" };
Arrays.sort(stringArray, String::compareToIgnoreCase);// String::xx 这里不是静态方法，而是作为方法实例在使用
```

现在我们来比较一下参数：

```java
// Comparator
int compare(T o1, T o2);
// String::compareToIgnoreCase
 public int compareToIgnoreCase(String str)
```

这种情况是怎么回事？

这种方式等价于:

```java
int compareToIgnoreCase(String str2) <==> (String str1, String str2) -> { return str1.compareToIgnoreCase(str2);}
// 传入的第一个参数作为调用者，第二个参数作为真正的compareToIngnoreCase的参数
```

为什么会这样呢?

这其实又是一个很大的话题。总结下来，就两句话：

-  第一点：**接口方法的参数比引用方法的参数多一个**

- 第二点：**接口方法的第一个参数恰巧是调用引用方法的对象（其引用方法所在类或其父类的实例）**

具体可参考：[点这里](https://blog.csdn.net/weixin_41126303/article/details/81187002)

**实际开发中，不推荐这种方式！**

### 3.4 构造方法

```java
// 这里对应的是Student的默认构造函数
Supplier<Student> supplier = Student::new; 
// 下面看一个非默认构造函数
interface MyFunction<T,U,R>{
    R apply(T t,U u);
}

class Student{
    private int id;
    private String name;
    public Student(int id, String name) {
        this.id = id;
        this.name = name;
    }
}
// 此时对应的是Student(int id, String name)构造函数
 MyFunction<Integer,String,Student> myFunction = Student::new;

// 使用：
MyFunction<Integer,String,Student> myFunction = Student::new;
Student stu = myFunction.apply(1,"raven");
System.out.println(stu);
```

## 3.5 方法引用总结

1. 静态方法引用 参数一一对应
2. 实例方法引用
   1. 特定实例的方法引用， 参数一一对应
   2. 引用特定类型的任意对象，函数接口应该比方法引用多一个参数，且这个参数是对应的类型（或父类型）。很难理解，很不推荐使用。另一种理解方式
3. 构造器，参数一一对应



再引用知乎某大佬的理解方式：

原文链接:https://zhuanlan.zhihu.com/p/69985120 

**在评论中找下面的话，原文不要看。**

> lambda 内部使用 MethodHandle，所以其实方法引用的处理规则跟 MethodHandle 是一模一样的。
>
> 
>
> 以 PrintStream::println 这个方法引用为例，它实际上调用的方法是 PrintStream 里的 void println(String)，但因为它是成员方法，所以内部处理时的使用的方法签名实际上是 (PrintStream, String)void，即把 this 作为第一个传入参数。
>
> 
>
> 因此知道这个规律后，就能明白 String::compareToIgnoreCase 是怎么回事了——它的方法签名为 (String, String)int，参数列表增加了 this（在这里就是一个 String）后刚好符合了 Comparator<String> 接口里 int compareTo(String, String) 的要求，因此可以作为 Arrays.sort(String[], Comparator<String>) 的参数。
>
> 
>
> 静态方法没有 this，所以方法签名该是怎么样还是怎么样。比如 String valueOf(int) 的签名就是 (int)String。
>
> 
>
> 当 :: 前面不是一个类型，而是一个对象，比如 "a"::equals 的时候，JDK 在生成 lambda 实例时会自动把目标对象给 bindTo 到对应的 MethodHandle 上。意思是：原来 String::equals 的方法签名根据上一段提到的规则是 (String, Object)boolean，但因为 :: 前面是一个 String 实例 "a"，因此方法引用的第一个参数（在这里就是 this）被绑定为了 "a"，方法签名变成了 (Object)boolean（因为原先第一个 String 参数被绑定成 "a" 了，它的值永远不会改变了，所以就只剩下 (Object)boolean 了）。这个改变后的签名就符合了 Predicate 接口里 boolean test(Object) 的要求。
>
> 
>
> 总结一下的话，三条规则就能说清楚了：
>
> 1. 成员方法的方法签名，前面会追加 this 的类型。
> 2. 静态方法的方法签名，因为没有 this, 不会追加任何东西。
> 3. 当 :: 前是一个实例时，这个实例会作为第一个参数给绑定到目标方法签名上。

## 参考：

- https://blog.csdn.net/weixin_41126303/article/details/81187002

- http://zyzhang.github.io/blog/2013/06/15/java8-preview-method-reference/
- http://moandjiezana.com/blog/2014/understanding-method-references/
- https://docs.oracle.com/javase/tutorial/java/javaOO/methodreferences.html
- https://www.baeldung.com/java-method-references