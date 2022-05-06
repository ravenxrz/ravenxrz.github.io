---
title: CMU-15445-proj1(BufferPool)
abbrlink: 8983c10b
date: 2021-10-06 18:44:23
categories: CMU15445
tags:
---

## 1. 要求

原文要求参考：https://15445.courses.cs.cmu.edu/fall2020/project1/

这是cmu15445的第一个proj，目标是实现一个简单的buffer pool。具体来看，整个proj需要实现两个类：一个LRUReplacer和一个BufferPoolManager。

<!--more-->

- LRUReplacer需要实现以下四个methods:
  - `Victim(T*)` : Remove the object that was accessed the least recently compared to all the elements being tracked by the `Replacer`, store its contents in the output parameter and return `True`. If the `Replacer` is empty return `False`.
  - `Pin(T)` : This method should be called after a page is pinned to a frame in the `BufferPoolManager`. It should remove the frame containing the pinned page from the `LRUReplacer`.
  - `Unpin(T)` : This method should be called when the `pin_count` of a page becomes 0. This method should add the frame containing the unpinned page to the `LRUReplacer`.
  - `Size()` : This method returns the number of frames that are currently in the `LRUReplacer`.

额外需要注意的是，LRUReplacer是需要 保证**线程安全**的。

- BufferPoolManager相对复杂一些，并且依赖于LRUReplacer，核心需要实现以下几个methods:
  - `FetchPageImpl(page_id)`
  - `NewPageImpl(page_id)`
  - `UnpinPageImpl(page_id, is_dirty)`
  - `FlushPageImpl(page_id)`
  - `DeletePageImpl(page_id)`
  - `FlushAllPagesImpl()`

同样，BufferPoolManager需要保证线程安全。

**测试方式：**

基本测试：

```
mkdir build
cmake ..
# 测试lru replacer
make lru_replacer_test -j
./test/lru_replacer_test
# 测试buffer_pool_manager
make buffer_pool_manager_test -j  
./test/buffer_pool_manager_test 
```

全测试需要去：https://www.gradescope.com/courses/195440， 具体请参考 [这里](https://ravenxrz.github.io/archives/1b7fd99d.html)

## 2. 题解（不含任何代码）

这里遵守课程要求，不会公开任何代码。只记录解题思路和坑点。

### 1. LRUReplacer

LRUReplacer相对简单，只要是刷过leetcode的[LRU缓存机制](https://leetcode-cn.com/problems/lru-cache/)应该都知道如何做。核心为用一个map做index，一个list做data container。至于线程安全如何保证，加大锁即可。解释下Pin和Unpin函数:

- Pin函数，当线程在访问某个数据时，只要该线程没有释放该数据的句柄，所以该数据是不应该存放在replacer中的，其实就是引用计数，换句话说，Pin函数执行时，需要将对应数据从replacer中移除。
- Unpin函数，线程释放该数据句柄时，需要该数据放入replacer中，放在哪儿？根据LRU算法规则放置即可。

> 坑记录：对于一个已经Unpin的数据来说，如果再次调用Unpin，此时应该do nothing, 而不是将数据放入到MRU位置处。

### 2. BufferPoolManager

要做这个题，那要理解BufferPool是什么。BufferPool就是disk上数据在内存中的缓存。那么有以下几个关键的数据结构需要注意：

1. 既然是缓存，那么容量肯定是不如disk的，需要合适的替换算法（上面的replacer）。

2. disk中的数据和memory中的数据，数据交换的粒度是多少？本proj中，这个数据粒度称为一个page（默认是4k)。

3. memory中的pages/buffer是如何组织的？本proj中，buffer就是一个一维page数组, 这个page数组中的每一个page又被称为一个frame。

   > 概念理清：在memory中，buffer=一组page，一个page也可以被称为一个frame

4. 如何找到memory中空闲的page？关注 free\_list数据结构，只要free\_list不为空，那么应该从free\_list中去拿free page，否则使用replacer替换一个free page出来。

5. 对于上游业务来说，是看不到memory中的page的（buffer pool对于上游业务透明），上游业务只会拿到数据的page_id，不知道这个数据page在内存中的哪个frame。所以BufferPoolManager还需要一个page\_table来存储 `page_id - frame_id`的映射。

ok, 至此所有关键的数据结构都已经理清，剩下的就是数据处理逻辑了，以一张图来看：

![结构图](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/结构图-1.svg)

frame状态图：

![frame状态图-bufferpool-状态机](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/frame状态图-bufferpool-状态机-1.svg)

1. free -> Pin: 从free_list中获取一个frame

2. Pin -> Pin: FetchPage一个已经Pin的frame，只会增加pin\_count; UnpinPage只要pin\_count没有变为0，都处于Pin状态
3. Pin -> Unpin: UnpinPage直到`pin_count`为0为止。
4. Unpin -> Pin:  FetchPage和NewPage都可能造成pin\_count不为0.
5. Unpin -> free: DeletePage时，需要清空frame，并将frame归还到free\_list中。

核心部分至此结束，**下面说明一些坑点**：

1. NewPage时，不要从disk中ReadData()。 NewPage的作用仅仅是让disk分配一个page\_id, 将这个page\_id返回给上游。 如果此时调用ReadData(), disk上的文件对应page id处的page实际上还未分配，此时会报 `I/O error reading past end of file`错误。

2. dirty位的处理：在UnpinPage的参数中，可以设置对应page id的dirty flag。 然而并不是简单的调用 `page->is_drity_ = dirty`. 正确的应该是 `page->is_drity_ |= dirty`.  

   > 这里是最坑的，也是让我卡了很久很久的地方，甚至重写了整个实验。原因在于UnpinPage不应该负责flush dirty page。 所以dirty的属性应该一直保持。

ok。以上是proj1的解析。

