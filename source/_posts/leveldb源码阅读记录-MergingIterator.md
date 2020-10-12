---
title: leveldb源码阅读记录-MergingIterator
categories: leveldb
abbrlink: b85343e6
date: 2020-10-12 18:54:37
tags:
---

MergingIterator是用于merge sort的iterator。mergeiterator内部有多个有序块，这些有序块分别有一个iterator来遍历，mergeiterator每次访问具有最小key的块。

举个例子：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb源码阅读-copy-第 22 页.png" style="zoom:33%;" />

如果从小到大访问，则首先通过iter2访问块2的0， 接着通过iter1访问块1的1，iter2-块2-2， iter3-块3-3....

<!--more-->

## 0. 成员变量

```c++
 private:
  // Which direction is the iterator moving? 
  enum Direction { kForward, kReverse };

  void FindSmallest();
  void FindLargest();

  // We might want to use a heap in case there are lots of children.
  // For now we use a simple array since we expect a very small number
  // of children in leveldb.
  const Comparator* comparator_;
  IteratorWrapper* children_;	// children数组
  int n_;	// n个children
  IteratorWrapper* current_;
  // 方向
  Direction direction_;
```

## 1. 创建一个MergingIterator

```c++
Iterator* NewMergingIterator(const Comparator* comparator, Iterator** children,
                             int n) {
  assert(n >= 0);
  if (n == 0) {
    return NewEmptyIterator();
  } else if (n == 1) {
    return children[0];
  } else {
    return new MergingIterator(comparator, children, n);
  }
}
```

参数包含多个children iterators，n表示children的个数.

## 2. FindSmallest操作

```c++
void MergingIterator::FindSmallest() {
  IteratorWrapper* smallest = nullptr;
  for (int i = 0; i < n_; i++) {
    IteratorWrapper* child = &children_[i];
    if (child->Valid()) {
      if (smallest == nullptr) {
        smallest = child;
      } else if (comparator_->Compare(child->key(), smallest->key()) < 0) {
        smallest = child;
      }
    }
  }
  current_ = smallest;
}
```

代码很简单，在所有iterator中找到目前key最小的那个， current_指向具有最小key的iterator。

FindLargets和FIndSamllest是对称操作，这里略。

## 3. Seek操作

```c++
  void Seek(const Slice& target) override {
    for (int i = 0; i < n_; i++) {
      children_[i].Seek(target);
    }
    FindSmallest();
    direction_ = kForward;
  }
```

确定当前最接近target的具有最小key的iterator，同时将其他iter放在合适的位置（通过seek）。

## 4. Next 

```c++
  void Next() override {
    assert(Valid());

    // Ensure that all children are positioned after key().
    // If we are moving in the forward direction, it is already
    // true for all of the non-current_ children since current_ is
    // the smallest child and key() == current_->key().  Otherwise,
    // we explicitly position the non-current_ children.
    if (direction_ != kForward) {
      for (int i = 0; i < n_; i++) {
        IteratorWrapper* child = &children_[i];
        if (child != current_) {
          child->Seek(key());
          if (child->Valid() &&
              comparator_->Compare(key(), child->key()) == 0) {
            child->Next();
          }
        }
      }
      direction_ = kForward;
    }

    current_->Next();
    FindSmallest();
  }
```

工作如下：

1. next是为了保证所有的children位置都在 key() 之后，而因为current_->key()已经是当前的最小child，所以non-child的iter不用做next操作，除非non-child的key=current\_->key. 
2. next当前current\_的key
3. 重新在所有iter中找到最小的child

由于Prev和Next的反操作，且操作对称，所以这里不再赘述。

## 5. Key & Value

如果方向是forward，则current_代表最小child，返回该最小child的key和value即可。

如果方向是reverse，则current_代表最大child，返回该最大child的key和value即可。

```c++
 Slice key() const override {
    assert(Valid());
    return current_->key();
  }

  Slice value() const override {
    assert(Valid());
    return current_->value();
  }
```

## 6. 总结

MergingIterator是一个组合iter，内部含有多个iter，每个iter指向一个“有序”的存储结构，MergingIterator每次都返回这些存储结构中具有小（大）的数据。 这和归并排序的思想是完全相同的，类比学习即可。

MergingIterator也被用于compaction， 参考 copaction章节的DoCompactionWork部分。

