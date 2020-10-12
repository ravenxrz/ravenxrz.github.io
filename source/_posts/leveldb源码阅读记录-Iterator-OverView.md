---
title: leveldb源码阅读记录-Iterator-OverView
categories: leveldb
abbrlink: 2b8370d3
date: 2020-10-12 18:31:40
tags:
---

通过之前对LevelDB的整体流程，数据存储以及元信息管理的介绍，我们已经基本完整的了解了LevelDB。从这篇文章开始，我们将重心转移到Iterator上，这是一种统一的访问底层数据的设计模式，用户不用关心底层数据是如何存储，只需使用Iterator提供的几种操作接口，即可访问。本文是leveldb众多iterator的一个总览，后文再对各个Iterator单独分析。

本文转载自：https://www.jianshu.com/p/7fe24a77002a

<!--more-->

## **作用**

正如[庖丁解LevelDB之数据存储](https://link.jianshu.com?t=http://catkang.github.io/2017/01/17/leveldb-data.html)中介绍的，LevelDB各个组件用不同的格式进行数据存取。在LevelDB内部外部，各个不同阶段又不可避免的需要从不同的视角遍历这些数据。如果每一个层次的数据遍历都需要详细的关心全部数据存储格式，无疑将使得整个过程变得无比的冗余复杂。Iterator的出现正式为了解决这个问题，Iterator在各个层次上，向上层实现提供了：

**无须了解下层存储细节的情况下，通过统一接口对下层数据进行遍历的能力。**

## **接口**

Iterator用确定的遍历接口将上层需求和下层实现解耦和。熟悉STL的同学一定不会陌生Iterator的使用方式，这里LevelDB的Iterator接口包括：

*   Seek到某一位置：Seek，SeekToFirst，SeekToLast；
*   访问前驱后继：Next，Prev；
*   判断当前位置是否有效：Valid；
*   获取当前位置数据信息：key，value，status；
*   可以注册多个Cleanup方法，当Iterator析构前做一些清理操作。

### Iterator虚基类定义

下面给出，Iterator虚基类的定义：

```c++
class LEVELDB_EXPORT Iterator {
 public:
  Iterator();

  Iterator(const Iterator&) = delete;
  Iterator& operator=(const Iterator&) = delete;

  virtual ~Iterator();

  // An iterator is either positioned at a key/value pair, or
  // not valid.  This method returns true iff the iterator is valid.
  virtual bool Valid() const = 0;

  // Position at the first key in the source.  The iterator is Valid()
  // after this call iff the source is not empty.
  virtual void SeekToFirst() = 0;

  // Position at the last key in the source.  The iterator is
  // Valid() after this call iff the source is not empty.
  virtual void SeekToLast() = 0;

  // Position at the first key in the source that is at or past target.
  // The iterator is Valid() after this call iff the source contains
  // an entry that comes at or past target.
  virtual void Seek(const Slice& target) = 0;

  // Moves to the next entry in the source.  After this call, Valid() is
  // true iff the iterator was not positioned at the last entry in the source.
  // REQUIRES: Valid()
  virtual void Next() = 0;

  // Moves to the previous entry in the source.  After this call, Valid() is
  // true iff the iterator was not positioned at the first entry in source.
  // REQUIRES: Valid()
  virtual void Prev() = 0;

  // Return the key for the current entry.  The underlying storage for
  // the returned slice is valid only until the next modification of
  // the iterator.
  // REQUIRES: Valid()
  virtual Slice key() const = 0;

  // Return the value for the current entry.  The underlying storage for
  // the returned slice is valid only until the next modification of
  // the iterator.
  // REQUIRES: Valid()
  virtual Slice value() const = 0;

  // If an error has occurred, return it.  Else return an ok status.
  virtual Status status() const = 0;

  // Clients are allowed to register function/arg1/arg2 triples that
  // will be invoked when this iterator is destroyed.
  //
  // Note that unlike all of the preceding methods, this method is
  // not abstract and therefore clients should not override it.
  using CleanupFunction = void (*)(void* arg1, void* arg2);
  void RegisterCleanup(CleanupFunction function, void* arg1, void* arg2);

 private:
  // Cleanup functions are stored in a single-linked list.
  // The list's head node is inlined in the iterator.
  struct CleanupNode {
    // True if the node is not used. Only head nodes might be unused.
    bool IsEmpty() const { return function == nullptr; }
    // Invokes the cleanup function.
    void Run() {
      assert(function != nullptr);
      (*function)(arg1, arg2);
    }

    // The head node is used if the function pointer is not null.
    CleanupFunction function;
    void* arg1;
    void* arg2;
    CleanupNode* next;
  };
  CleanupNode cleanup_head_;
};
```

可以看到Iterator定义了一个迭代器的通用接口，同时还包含了一个cleanup链表，可以向iterator注册多个cleanup函数，这些cleanup函数会在iterator析构时被调用。

## **分类**

LevelDB大量使用了各种Iterator，根据Iterator的实现和层次，我们将其分为三种类型：

*   基本Iterator：最原子的Iterator，针对相应的数据结构实现Iterator接口；
*   组合Iterator：通过各种方式将多个基本Iterator组合起来，向上层提供一致的Iterator接口。
*   功能Iterator：某种或多种组合Iterator的联合使用，附加一些必要的信息，实现某个过程中的遍历操作。

## **基本Iterator**

LevelDB中包括三种基本Iterator，他们分别针对Memtable，Block以及Version中的文件索引格式，实现了最原子的Iterator：

#### **1，MemTableIterator**

在Memtable Skiplist的格式上的Iterator实现。Memtable格式见[庖丁解LevelDB之数据存储](https://link.jianshu.com?t=http://catkang.github.io/2017/01/17/leveldb-data.html)。

#### **2，Block::Iter**

针对SST文件Block存储格式的Iterator实现。遍历的过程中解析重启点，拼接key的共享部分和特有部分，获取对应的value值。Block详细格式见[庖丁解LevelDB之数据存储](https://link.jianshu.com?t=http://catkang.github.io/2017/01/17/leveldb-data.html)。

#### **3，Version::LevelFileNumIterator**

[庖丁解LevelDB之版本控制](https://link.jianshu.com?t=http://catkang.github.io/2017/02/03/leveldb-version.html)中介绍了Version中记录了当前所有文件按层次划分的二维数组。其中Level1层之上的文件由于相互之间没有交集且有序，可以利用文件信息中的最大最小Key来进行二分查找。LevelFileNumIterator就是利用这个特点实现的对文件元信息进行遍历的Iterator。其中每个项记录了当前文件最大key到文件元信息的映射关系。这里的文件元信息包含文件号及文件长度。

## **组合Iterator**

组合Iterator由上述多个基本Iterator或组合Iterator组合而成，LevelDB中包含两种组合Iterator

#### **1，TwoLevelIterator**

TwoLevelIterator实现逻辑上有层次关系的数据的遍历操作。组合了**index iterator**和**data iterator**两层迭代器，其中index iterator记录从数据key值到data iterator的映射，而data iterator则负责真正数据key到value的映射。生成TwoLevelIterator时，需要提供index Iterator及BlockFunction函数，其中BlockFunction实现了index iterator value值的反序列化以及对应的data iterator的生成。

#### **2，MergingIterator**

通过MergingIterator可以实现多个有序数据集合的归并操作。其中包含多个**child iterator**组成的集合。对MergingIterator的遍历会有序的遍历其child iterator中的每个元素。

## **功能Iterator**

为了实现不同场景下不同层次的数据遍历操作，可以联合一种或多种组合Iterator，这里称为功能Iterator，按其所负责的数据层次由下自上进行介绍：

#### **1，Table::Iterator**

对SST文件的遍历，通过[庖丁解LevelDB之数据存储](https://link.jianshu.com?t=http://catkang.github.io/2017/01/17/leveldb-data.html)可知，这里有明显的层级关系，可以利用上面介绍的TwoLevelIterator，其**index iterator**为Index Block的Block::Iter，**data iterator**为Data Block的Block::Iter

![](https://pic.downk.cc/item/5f8442be1cd1bbb86b053032.jpg)

Table::Iterator

#### **2，Compaction过程Iterator**

Compaction过程中需要对多个文件进行归并操作，并将结果输出到新的下层文件。LevelDB用MergingIterator来实现这个过程，其**clild iterator**包括[庖丁解LevelDB之版本控制](https://link.jianshu.com?t=http://catkang.github.io/2017/02/03/leveldb-version.html)中提到的要Compaction的多个文件对应的Iterator：

*   如果有Level0文件，则包含所有level0文件的Table::Iterator

*   其他Level文件，包含文件索引的TwoLevelIterator，由Version::LevelFileNumIterato作为index iterator，Table::Iterator作为data iterator

![](https://pic.downk.cc/item/5f8442c81cd1bbb86b0537ed.jpg)

Compaction过程Iterator

#### **3，NewInternalIterator**

LevelDB作为整体同样通过Iterator向外部用户提供遍历全部数据的能力。这里使用MergingIterator将Memtable，Immutable memtable及各层SST文件的Iterator归并起来，使得外部使用者不用关心具体的内部实现而有序的循环LevelDB内部的数据，LevelDB首先实现了NewInternalIterator：

![](https://pic.downk.cc/item/5f8442d21cd1bbb86b053fa4.jpg)

在NewInternalIterator的基础上，LevelDB又封装了DBIter来处理快照，过滤已删除key。

## **参考**

Source Code：[https://github.com/google/leveldb](https://link.jianshu.com?t=https://github.com/google/leveldb)

庖丁解LevelDB之概览: [http://catkang.github.io/2017/01/07/leveldb\-summary.html](https://link.jianshu.com?t=http://catkang.github.io/2017/01/07/leveldb-summary.html)

庖丁解LevelDB之数据管理: [http://catkang.github.io/2017/01/07/leveldb\-summary.html](https://link.jianshu.com?t=http://catkang.github.io/2017/01/07/leveldb-summary.html)

庖丁解LevelDB之版本控制：[http://catkang.github.io/2017/02/03/leveldb\-version.html](https://link.jianshu.com?t=http://catkang.github.io/2017/02/03/leveldb-version.html)