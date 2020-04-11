---
title: JAVA IO流总结
categoires: 编程语言
tags: 
    - java
    - IO流
abbrlink: a068fc76
date: 2020-03-05 08:50:06
---

转载自：

1. [Java基础知识](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/Java%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86.md#35-java-%E4%B8%AD-io-%E6%B5%81)
2. [java I/O流详解](https://juejin.im/post/5d4ee73ae51d4561c94b0f9d)

### Java 中 IO 流分为几种?

- 从流的方向划分：分为 `输入流`， `输出流`
- 从流的传输单位划分： 分为 `字节流`（8位字节），`字符流`（16位的字符）
- 从流的角色划分： 分为 `节点流`，`处理流`
  - 节点流是直接连接`数据源`的流，可以`直接向数据源（特定的IO设备，如硬盘，网络，其他程序）读写数据`。
  - 处理流通过`构造方法接收一个节点流`，对节点流使用`装饰者模式`增加更多的功能，处理流`必须依赖于一个节点流`，因为只有节点流最终可以将数据流输入输出到IO设备中。

<!-- more -->

Java Io 流共涉及 40 多个类，这些类看上去很杂乱，但实际上很有规则，而且彼此之间存在非常紧密的联系， Java I0 流的 40 多个类都是从如下 4 个抽象类基类中派生出来的。

- InputStream/Reader: 所有的输入流的基类，前者是字节输入流，后者是字符输入流。
- OutputStream/Writer: 所有输出流的基类，前者是字节输出流，后者是字符输出流。

按操作方式分类结构图：

![](https://pic.downk.cc/item/5e604d9498271cb2b85af6ee.jpg)

按操作对象分类结构图：

![](https://pic.downk.cc/item/5e604db798271cb2b85b0aef.jpg)

这里再解释一下“节点流”和“处理流”：

### 节点流--直接连接数据源的流

![](https://ae01.alicdn.com/kf/Hed5828b4792e48b198efef07eaf06183m.jpg)



### 常见的节点流

![](https://pic.downk.cc/item/5e604eab98271cb2b85b3407.jpg)



1. File 文件流。对文件进行读、写操作 ：FileReader、FileWriter、FileInputStream、FileOutputStream。
2. 从/向内存数组读写数据: CharArrayReader与 CharArrayWriter、ByteArrayInputStream与ByteArrayOutputStream。
3. 从/向内存字符串读写数据 StringReader、StringWriter、StringBufferInputStream。
4. Pipe管道流。 实现管道的输入和输出（进程间通信）: PipedReader与PipedWriter、PipedInputStream与PipedOutputStream。

### 处理流--连接已存在的流

![](https://pic.downk.cc/item/5e604ee398271cb2b85b3d71.jpg)

### 常见的处理流

![](https://pic.downk.cc/item/5e604ef298271cb2b85b3fed.jpg)

1. Buffering缓冲流：在读入或写出时，对数据进行缓存，以减少I/O的次数：BufferedReader与BufferedWriter、BufferedInputStream与BufferedOutputStream。
2. Filtering 滤流：在数据进行读或写时进行过滤：FilterReader与FilterWriter、FilterInputStream与FilterOutputStream。
3. Converting between Bytes and Characters 转换流：按照一定的编码/解码标准将字节流转换为字符流，或进行反向转换（Stream到Reader）：InputStreamReader、OutputStreamWriter。
4. Object Serialization 对象流 ：ObjectInputStream、ObjectOutputStream。
5. DataConversion数据流： 按基本数据类型读、写（处理的数据是Java的基本类型（如布尔型，字节，整数和浮点数））：DataInputStream、DataOutputStream 。
6. Counting计数流： 在读入数据时对行记数 ：LineNumberReader、LineNumberInputStream。
7. Peeking Ahead预读流： 通过缓存机制，进行预读 ：PushbackReader、PushbackInputStream。  8 .Printing打印流： 包含方便的打印方法 ：PrintWriter、PrintStream。

### FileInputStream & FileOutputStream

1. FileInputStream & FileOutputStream 可以从文件系统中 读取/写入 诸如图像数据之类的原始字节流。
2. 以下是 使用 FileInputStream & FileOutputStream 实现文件拷贝的案例

```java
// 使用文件字节流 一次拷贝一个字节
    private static void copyFile1(String src, String dest) throws IOException {
        //1. 创建流
        InputStream in = new FileInputStream(src);
        OutputStream os = new FileOutputStream(dest);

        //2. 读写数据
        int data = in.read();
        while (data != -1) {
            os.write(data);
            data = in.read();
        }

        //3. 关闭流
        in.close();
        os.close();
    }

    // 使用文件字节流 一次拷贝一个字节数组
    private static void copyFile2(String src, String dest) throws IOException {
        //1. 创建流
        InputStream in = new FileInputStream(src);
        OutputStream os = new FileOutputStream(dest);

        //2. 读写数据
        byte[] buffer = new byte[2048];
        int len = in.read(buffer);
        while (len != -1) {
            os.write(buffer, 0, len);
            len = in.read(buffer);
        }
        //3. 关闭流
        in.close();
        os.close();
   }

```

### DataInputStream & DataOutputStream

1. DataInputStream & DataOutputStream，是处理流，改构造方法接收一个已存在的输入输出流， 允许程序从读取方便快捷 操作java的基本数据类型。

```java
// 向文件中写入 java基本数据类型
    private static void write(String dest) throws IOException {
        //1. 创建流对象
        DataOutputStream os = new DataOutputStream(new FileOutputStream(dest));
        //2. 写入数据
        os.writeInt(10);
        os.writeChar('a');
        os.writeChar('b');
        os.writeDouble(12.83);
        //3. 关闭流
        os.close();
}

    // 从文件中读取 java基本数据类型，要和写入的顺序保持一致
    private static void read(String src) throws IOException {
        //1. 创建数据流对象
        DataInputStream in = new DataInputStream(new FileInputStream(src));
        //2. 读取数据
        int a = in.readInt();
        char b = in.readChar();
        char c = in.readChar();
        double d = in.readDouble();
        //3. 关闭流
        in.close();
    }

```

### BufferedInputStream & BufferedOutputStream

1. BufferedInputStream & BufferedOutputStream 为另一个输入输出流流添加一些功能，即缓冲区的作用。在创建 BufferedInputStream & BufferedOutputStream 时，会创建一个内部缓冲区数组。

```java
// 为文件字节流 添加缓冲区功能， 一次读写一个字节数据，但内部缓冲区数组已经填满
    private static void copyFile1(String src, String dest) throws IOException {
        //1. 创建流
        InputStream in = new BufferedInputStream(new FileInputStream(src));
        OutputStream os = new BufferedOutputStream(new FileOutputStream(dest));

        //2. 读写数据
        int data = in.read();
        while (data != -1) {
            os.write(data);
            data = in.read();
        }

        //3. 关闭流
        in.close();
        os.close();
    }

    // 为文件字节流 添加缓冲区功能， 一次读写一个字节数组数据，但内部缓冲区数组已经填满
    private static void copyFile2(String src, String dest) throws IOException {
        //1. 创建流
        InputStream in = new BufferedInputStream(new FileInputStream(src));
        OutputStream os = new BufferedOutputStream(new FileOutputStream(dest));

        //2. 读写数据
        byte[] buffer = new byte[2048];
        int len = in.read(buffer);
        while (len != -1) {
            os.write(buffer, 0, len);
            len = in.read(buffer);
        }
        //3. 关闭流
        in.close();
        os.close();
    }

```

### PrintStream

1. PrintStream 为其他输出流添加了功能，使它们能够方便地打印各种数据值表示形式。

```java
public static void main(String[] args) throws FileNotFoundException {

        PrintStream ps = new PrintStream("test.txt");
        ps.println(5);
        ps.print("Aaaa");
        ps.print(false);
        ps.println("hahah");

        ps.flush();
        ps.close();
    }

```

### ByteArrayInputStream & ByteArrayOutputStream

ByteArrayInputStream & ByteArrayInputStream 包含一个内部缓冲区(实际上就是把数据写入内存，然后再读取)，该缓冲区包含从流中读取的字节。内部计数器跟踪 read 方法要提供的下一个字节。

有人说这种写入到内存又读到内存的意义是什么，举个不常用的例子，假设我们对一个对象进行序列化，当从序列化后的文件中读回时，我们可以得到这个对象的深拷贝，但是如果写入到文件的话就会特别耗时，这时采用这ByteArrayOuputStream就可以把序列化写入到内存中，这样速度就会加快很多了。

```java
public static void main(String[] args) throws IOException {

        //1. 写入数据
        ByteArrayOutputStream os = new ByteArrayOutputStream();
        os.write("hello".getBytes());
        os.close(); // 没有写任何实现

        //2. 读取输出
        byte[] bytes = os.toByteArray();
        ByteArrayInputStream in = new ByteArrayInputStream(bytes);

        byte[] data = new byte[1014];
        int len = in.read(data);
        System.out.println(new String(data, 0 ,len));

    }

```

### InputStreamReader & OutputStreamWriter

这两个比较特殊，它是字符流和字节流之间的桥梁，先看看官网是如何描述它们的吧：

> An InputStreamReader is a bridge from byte streams to character streams: It reads bytes and decodes them into characters using a specified [`charset`](https://devdocs.io/openjdk~8/java/nio/charset/charset). The charset that it uses may be specified by name or may be given explicitly, or the platform's default charset may be accepted.
>
> Each invocation of one of an InputStreamReader's read() methods may cause one or more bytes to be read from the underlying byte-input stream. To enable the efficient conversion of bytes to characters, more bytes may be read ahead from the underlying stream than are necessary to satisfy the current read operation.
>
> For top efficiency, consider wrapping an InputStreamReader within a BufferedReader. For example:
>
> ```java
> BufferedReader in
>    = new BufferedReader(new InputStreamReader(System.in)); // 这也是从控制台读入数据的第二种方法，第一种是使用Scanner类
> ```
>
> 

这就是说，InputStreamReader/OutputStreamWriter可以将字节流的读写转化为字符流的读写。只要我们提供一个指定的charset即可（当然它有默认的charset的）。

```java
// 使用 字符转换流， 实现文本文件的 拷贝
    // 一次拷贝一个 字符
    private static void copyFile1(String src, String dest) throws IOException {
        //1. 创建转换流
        Reader reader = new InputStreamReader(new FileInputStream(src));
        Writer writer = new OutputStreamWriter(new FileOutputStream(dest));

        //2. 拷贝数据
        int data = reader.read();
        while (data != -1) {
            writer.write(data);
            data = reader.read();
        }
        //3.关闭流
        reader.close();
        writer.close();
    }

    // 一次拷贝一个 字符数组
    private static void copyFile2(String src, String dest) throws IOException {
        //1. 创建转换流
        Reader reader = new InputStreamReader(new FileInputStream(src));
        Writer writer = new OutputStreamWriter(new FileOutputStream(dest));

        //2. 拷贝数据
        char[] buffer = new char[2048];
        int len = reader.read(buffer);
        while (len != -1) {
            writer.write(buffer, 0 , len);
            len = reader.read(buffer);
        }
        //3.关闭流
        reader.close();
        writer.close();
    }

```

### FileReader & FileWriter

在只针对将字符写入文件的时候，因为 每次使用 转换流，对字节流进行包装，写法太麻烦，所以jdk 提供了 字节转换流子类FileReader & FileWriter，方便的进行字符文件的IO操作

```java
// 使用 字符转换流， 实现文本文件的 拷贝
    // 一次拷贝一个 字符
    private static void copyFile1(String src, String dest) throws IOException {
        //1. 创建转换流
        Reader reader = new FileReader(src);
        Writer writer = new FileWriter(dest);

        //2. 拷贝数据
        int data = reader.read();
        while (data != -1) {
            writer.write(data);
            data = reader.read();
        }
        //3.关闭流
        reader.close();
        writer.close();
    }

    // 一次拷贝一个 字符数组
    private static void copyFile2(String src, String dest) throws IOException {
        //1. 创建转换流
        Reader reader = new FileReader(src);
        Writer writer = new FileWriter(dest);

        //2. 拷贝数据
        char[] buffer = new char[2048];
        int len = reader.read(buffer);
        while (len != -1) {
            writer.write(buffer, 0 , len);
            len = reader.read(buffer);
        }
        //3.关闭流
        reader.close();
        writer.close();
    }

```

### BufferedReader & BufferedWriter

```java
// 一次拷贝一个 字符
    private static void copyFile1(String src, String dest) throws IOException {
        //1. 创建转换流
        Reader reader = new BufferedReader(new FileReader(src));
        Writer writer = new BufferedWriter(new FileWriter(dest));

        //2. 拷贝数据
        int data = reader.read();
        while (data != -1) {
            writer.write(data);
            data = reader.read();
        }
        //3.关闭流
        reader.close();
        writer.close();
    }

    // 一次拷贝一个 字符数组
    private static void copyFile2(String src, String dest) throws IOException {
        //1. 创建转换流
        Reader reader = new BufferedReader(new FileReader(src));
        Writer writer = new BufferedWriter(new FileWriter(dest));

        //2. 拷贝数据
        char[] buffer = new char[2048];
        int len = reader.read(buffer);
        while (len != -1) {
            writer.write(buffer, 0 , len);
            len = reader.read(buffer);
        }
        //3.关闭流
        reader.close();
        writer.close();
    }

    // 一次拷贝一个一整行的 字符串
    private static void copyFile3(String src, String dest) throws IOException {
        //1. 创建转换流
        BufferedReader reader = new BufferedReader(new FileReader(src));
        BufferedWriter writer = new BufferedWriter(new FileWriter(dest));

        //2. 拷贝数据
        String data = reader.readLine();
        while (data != null) {
            writer.write(data);
            writer.newLine();
            data = reader.readLine();
        }
        //3.关闭流
        reader.close();
        writer.close();
    }

```

### StringReader & StringWriter

方便快捷的将字符串写入内存，或从内存读取

```java
public static void main(String[] args) throws IOException, NoSuchFieldException {
        //1. 向内存写入数据，其内部提供了一个缓冲区
        StringWriter writer = new StringWriter();
        writer.write("hello world");

        //2. 读取数据
        StringReader reader = new StringReader(writer.toString());

        char[] data = new char[1024];
        int len =  reader.read(data);

        System.out.println(new String(data,0,len));
    }
```

