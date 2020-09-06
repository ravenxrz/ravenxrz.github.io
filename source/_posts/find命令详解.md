---
title: find命令详解
categories: Linux
tags:
  - linux
  - find命令
abbrlink: be26ae40
date: 2020-04-11 20:53:30
---

## 1. Q&A

- 我几天前下的pdf文档，txt文件等放在哪儿了？
- 今天我创建了哪些文件？
- 某个目录下的markdown文件一共有多少个？
- 如何批量删除符合条件的文件?

上面这些问题通通都可以使用**find命令**来解决。

> find命令在类unix系统下都可以使用，windows平台可通过安装cygwin等仿linux环境来使用。

<!--more-->
## 2.  find命令

## 2.1 what is find command?

![find man page](https://shapeshed.com/images/articles/find.png)

## 2.2 通过文件名来搜索文件

可以通过`-name`参数来完全匹配单个文件，或使用通配符来批量匹配一批文件。

实验环境：

```
.
├── bar
├── baz
│   ├── file1
│   ├── file10
│   ├── file2
│   ├── file3
│   ├── file4
│   ├── file5
│   ├── file6
│   ├── file7
│   ├── file8
│   └── file9
└── bop
```

1. 完全匹配单个文件

假如我们要找到file1这个文件的路径，则可以这样使用:

```shell
find . -name "file1"
```

结果:

```java
./baz/file1
```

2. 批量匹配

我们要找到filex{x<10}的文件，则可以使用占位符?,如:

```shell
find . -name "file?"
```

结果：

```powershell
./baz/file1
./baz/file2
./baz/file3
./baz/file4
./baz/file5
./baz/file6
./baz/file7
./baz/file8
./baz/file9
```

假如我们要找所有以file开头的文件，则可以使用通配符*，如：

```shell
find . -name "file*"
```

结果:

```
./baz/file1
./baz/file10
./baz/file2
./baz/file3
./baz/file4
./baz/file5
./baz/file6
./baz/file7
./baz/file8
./baz/file9
```

## 2.3 删除符合条件的文件

假如我们要删除所有以file开头的文件：

```shell
find . -name "file*" -delete
```

## 2.4 指定搜索类型（目录or文件？)

我们可以通过-type参数，来指定我们搜索的文件是目录还是文件：

- -type f 文件
- -type d 目录

假设当前我们的实验环境为：

```
├── bar
├── baz	-- 下面的filex 全是文件
│   ├── file1
│   ├── file10
│   ├── file2
│   ├── file3
│   ├── file4
│   ├── file5
│   ├── file6
│   ├── file7
│   ├── file8
│   └── file9
└── bop	-- 下面的filex 全是目录
    ├── file1
    ├── file10
    ├── file2
    ├── file3
    ├── file4
    ├── file5
    ├── file6

```

则，我们要找所有的文件类型file时:

```shell
find . -name "file*" -type f
```

```
./baz/file1
./baz/file10
./baz/file2
./baz/file3
./baz/file4
./baz/file5
./baz/file6
./baz/file7
./baz/file8
./baz/file9
```

要找所有目录类型的file时：

```shell
find . -name "file*" -type d
```

```
./bop/file1
./bop/file10
./bop/file2
./bop/file3
./bop/file4
./bop/file5
./bop/file6
./bop/file7
./bop/file8
./bop/file9
```

## 2.5 如何根据时间戳寻找文件

每个文件都有三个 time stamps. 用来记录当对文件进行某种操作后的结果。这三个分别是：

- [a] access( 读取文件内容)时间 - atime（天） amin（分钟）
- [m] modify (修改了文件的内容)时间 - mtime mmin
- [c] change( 修改了文件内容或它的属性（如何文件权限）) - ctime cmin

-axxx -mxxx -cxxx这三种参数后也接一个数n：

- 写作-n时， 表示n天（分钟）前置今。 如 -atime -10 十天前至今。
- 写作+n时，表示n天以前， 如 -atime +10 十天以前
- 写作n时，表示n当天

假如我要找10天前到今天，“Downlaods“目录下所下过的文件,可执行：

```shell
find ~/Downlaods -mtime -10
```

又如找到3天前-1天前的下载文件：

```shell
find ~/Downlaods -mtime -3 -mtime 1
```

## 2.6 通过文件权限找文件

这个不太常用，如找文件权限为777的文件:

```shell
find . -perm 777
```

## 2.7 -newer参数

-newer参数，可以用来找比某个文件新的文件，如找比file1新的文件:

```shell
find . -newer baz/fiel1
```

## 2.8 重点！ 找到符合条件的文件并执行某种参数

我们可以通过在find命令后添加` -exec 操作命令 {} \; `来执行某种命令,

如删除所有比file1新的文件，则:

```shell
find . -newer file1 -exec rm {} \;
// 另外可写为
find . -newer file1 -delete	# 这种方式在删除大量文件时，效率更高
```

又如，常用的批量替换某些文件的单词。

说一个我个人的案例，我在使用hexo静态博客写文件时，将categories标签，在几乎所有的文档中都写成了categrioes。

有了find指令再加上sed指令，我们就可以实现批量替换:

```shell
find . -name "*.md" -exec sed -i 's/categrioes/categories/g' {} \;
```

OK, find的常用使用方法就总结到这里。更多的使用方法请参照`man find`

## 参考：

- [Find Files By Access, Modification Date / Time Under Linux or UNIX](https://www.cyberciti.biz/faq/howto-finding-files-by-date/)
- [Linux and Unix find command tutorial with examples](https://shapeshed.com/unix-find/)

