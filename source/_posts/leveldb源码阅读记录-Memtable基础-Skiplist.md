---
title: leveldb源码阅读记录-Memtable基础-Skiplist
categories: leveldb
abbrlink: 931e70da
date: 2020-10-12 11:11:33
tags:
---

在leveldb中，memtable的实现是采用skiplist的，所以本篇文章，我们就来分析以下leveldb的skiplist。

对于不了解skiplist，建议先学习以下skiplist的基本概念，最好能跟着实现一个skiplist。

<!--more-->

## 1. skiplist的定义

```c++
template <typename Key, class Comparator>
class SkipList {
 private:
  struct Node;

 public:
  // Create a new SkipList object that will use "cmp" for comparing keys,
  // and will allocate memory using "*arena".  Objects allocated in the arena
  // must remain allocated for the lifetime of the skiplist object.
  explicit SkipList(Comparator cmp, Arena* arena);

  SkipList(const SkipList&) = delete;
  SkipList& operator=(const SkipList&) = delete;

  // Insert key into the list.
  // REQUIRES: nothing that compares equal to key is currently in the list.
  void Insert(const Key& key);

  // Returns true iff an entry that compares equal to key is in the list.
  bool Contains(const Key& key) const;

  // Iteration over the contents of a skip list
  class Iterator {
     ...
  };

 private:
  enum { kMaxHeight = 12 };

   ...

  // Immutable after construction
  Comparator const compare_;
  Arena* const arena_;  // Arena used for allocations of nodes

  Node* const head_;

  // Modified only by Insert().  Read racily by readers, but stale
  // values are ok.
  std::atomic<int> max_height_;  // Height of the entire list

  // Read/written only by Insert().
  Random rnd_;
};
```

总体来看，这个skiplist只提供了, `Insert`和`Contains`两个接口。一个用于插入key，一个用于判断skiplist中是否存在具有和传入key相同的key的entry。为什么不提供delete接口？因为在leveldb中，一个删除操作就是一个插入一个具有“删除标记”的节点。所以删除即插入。

## 2. Insert

insert插入一个节点。

```c++
template <typename Key, class Comparator>
void SkipList<Key, Comparator>::Insert(const Key& key) {
  // TODO(opt): We can use a barrier-free variant of FindGreaterOrEqual()
  // here since Insert() is externally synchronized.
   // prev存放搜索路径
  Node* prev[kMaxHeight];
    // 设置搜索路径
  Node* x = FindGreaterOrEqual(key, prev);

    // 不允许出现相同节点
  // Our data structure does not allow duplicate insertion
  assert(x == nullptr || !Equal(key, x->key));

   // 采用随机height确定新节点能插入的最高高度
  int height = RandomHeight();
  if (height > GetMaxHeight()) {
    for (int i = GetMaxHeight(); i < height; i++) {
      prev[i] = head_;
    }
      // 更新最高高度
    // It is ok to mutate max_height_ without any synchronization
    // with concurrent readers.  A concurrent reader that observes
    // the new value of max_height_ will see either the old value of
    // new level pointers from head_ (nullptr), or a new value set in
    // the loop below.  In the former case the reader will
    // immediately drop to the next level since nullptr sorts after all
    // keys.  In the latter case the reader will use the new node.
    max_height_.store(height, std::memory_order_relaxed);
  }

    // 插入新节点
  x = NewNode(key, height);
  for (int i = 0; i < height; i++) {
    // NoBarrier_SetNext() suffices since we will add a barrier when
    // we publish a pointer to "x" in prev[i].
    x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));
    prev[i]->SetNext(i, x);
  }
}

```

插入节点逻辑很清晰, 主要就是三个工作：

1. 通过`FindGreaterOrEqual`函数确定搜索路径, 保存在 `prev`中
2. 生成本次插入节点的高度
3. 插入新节点

### 1. FindGreaterOrEqual

说一下`FindGreaterOrEqual`函数的作用：

```c++

template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node*
SkipList<Key, Comparator>::FindGreaterOrEqual(const Key& key,
                                              Node** prev) const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    Node* next = x->Next(level);
    if (KeyIsAfterNode(key, next)) {
      // Keep searching in this list
      x = next;
    } else {
      if (prev != nullptr) prev[level] = x;		// 保存路径到prev中
      if (level == 0) {
        return next;
      } else {
        // Switch to next list
        level--;
      }
    }
  }
}
```

举个例子：

![](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200909215139172.png)

现在一共有4层，level0,1,2,3. 最下面为level0. , 则prev数组的有效长度为4(实际上都为kmaxHeight),假设要插入87, 图中红色虚线为要插入时所经过的路线，prev数组的作用就是保存这些路线。 比如 prev[0] 代表的是level0层的86， prev[1]代表的是level1层的71,以此类推。

### 2.RandomHeight

紧接上面的例子，现在找到了要插入的地点（86的后一个节点），思考一个问题， skiplist的上层节点是怎么生成的？如果只在level0中插入87，那如何生成上层的87？ 这部分就是通过RandomHeight来实现的，每次找到要插入的节点的位置时，就为这个节点生成一个高度，从0到这个高度都要插入这个节点，而高度是随机生成的，我们看看这个函数：

```c++
template <typename Key, class Comparator>
int SkipList<Key, Comparator>::RandomHeight() {
  // Increase height with probability 1 in kBranching
  static const unsigned int kBranching = 4;
  int height = 1;
  while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
```

rnd_是一个随机对象，leveldb作者自己实现的随机数生成器，这部分我们不管，从这段代码来看， height是随机递增且不能超过预设的最大高度。

## 3. Contains

这个函数没什么好说的，在skiplist中找到节点，如果存在这个节点，且key相同，则返回。

```c++
template <typename Key, class Comparator>
bool SkipList<Key, Comparator>::Contains(const Key& key) const {
  Node* x = FindGreaterOrEqual(key, nullptr);
  if (x != nullptr && Equal(key, x->key)) {
    return true;
  } else {
    return false;
  }
}
```

## 4. Iterator

前面说了skiplist的两个接口，但是skiplist还提供了一个重要的功能，那就是迭代器，迭代器是用来遍历skiplist的内部节点的。我们一起来分析以下：

### 1. 基本定义

```c++
  // Iteration over the contents of a skip list
  class Iterator {
   public:
    // Initialize an iterator over the specified list.
    // The returned iterator is not valid.
    explicit Iterator(const SkipList* list);

    // Returns true iff the iterator is positioned at a valid node.
    bool Valid() const;

    // Returns the key at the current position.
    // REQUIRES: Valid()
    const Key& key() const;

    // Advances to the next position.
    // REQUIRES: Valid()
    void Next();

    // Advances to the previous position.
    // REQUIRES: Valid()
    void Prev();

    // Advance to the first entry with a key >= target
    void Seek(const Key& target);

    // Position at the first entry in list.
    // Final state of iterator is Valid() iff list is not empty.
    void SeekToFirst();

    // Position at the last entry in list.
    // Final state of iterator is Valid() iff list is not empty.
    void SeekToLast();

   private:
    const SkipList* list_;
    Node* node_;
    // Intentionally copyable
  };
```

可以看到，这个迭代器提供的接口和普通迭代器基本一样，**但没有value**，内部私有成员包含一个skiplist和一个node，毕竟是要遍历skiplist，所以这也很好理解。

### 2.Seek

```c++

template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::Seek(const Key& target) {
  node_ = list_->FindGreaterOrEqual(target, nullptr);
}

```

seek内部直接调用了skiplist的FindGreaterOrEqual函数，FindGreaterOrEqual除了保存搜索路径外，最终还会返回找到的节点，正好可以用来做seek，并将结果保存在node_

### 3. Next

Next函数很简单，移动到下一个节点。

```c++
template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::Next() {
  assert(Valid());
  node_ = node_->Next(0);
}
```

### 4. Prev

prev内部调用了FindLessThan，它和FindGreaterOrEqual类似。其实可以双链表优化，但是双链表会多占用一个prev指针，这里应该算是时间换空间吧。

```c++
template <typename Key, class Comparator>
inline void SkipList<Key, Comparator>::Iterator::Prev() {
  // Instead of using explicit "prev" links, we just search for the
  // last node that falls before key.
  assert(Valid());
  node_ = list_->FindLessThan(node_->key);
  if (node_ == list_->head_) {
    node_ = nullptr;
  }
```

### 5. key 

```c++
template <typename Key, class Comparator>
inline const Key& SkipList<Key, Comparator>::Iterator::key() const {
  assert(Valid());
  return node_->key;
}
```

到这里，我们就说完skiplist了，有了skiplist的基础，下文我们将介绍memtable。