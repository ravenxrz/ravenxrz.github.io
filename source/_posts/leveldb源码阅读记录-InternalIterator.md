---
title: leveldb源码阅读记录-InternalIterator
categories: leveldb
abbrlink: 615b88c9
date: 2020-10-12 18:56:54
tags:
---

# InternalIterator

在LevelDB中并没有InternalIterator这个类，它是一个功能Iterator，实现的功能是遍历整个leveldb的key value.从本质上来看就是一个MergingIterator.

<!--more-->

## 1. 创建一个InternalIterator

创建InternalIterator的入口为`DBImpl::NewInternalIterator:`

```c++
Iterator* DBImpl::NewInternalIterator(const ReadOptions& options,
                                      SequenceNumber* latest_snapshot,
                                      uint32_t* seed) {
  mutex_.Lock();
  *latest_snapshot = versions_->LastSequence();

  // Collect together all needed child iterators
  std::vector<Iterator*> list;
   // 加入mem的iterator
  list.push_back(mem_->NewIterator());
  mem_->Ref();
  if (imm_ != nullptr){
      // 加入imm_的iterator
    list.push_back(imm_->NewIterator());
    imm_->Ref();
  }
   // 加入外存levels的iterator
  versions_->current()->AddIterators(options, &list);
    // 组合成一个MergingIterator
  Iterator* internal_iter =
      NewMergingIterator(&internal_comparator_, &list[0], list.size());
  versions_->current()->Ref();

   // 注册清理函数
  IterState* cleanup = new IterState(&mutex_, mem_, imm_, versions_->current());
  internal_iter->RegisterCleanup(CleanupIteratorState, cleanup, nullptr);

  *seed = ++seed_;
  mutex_.Unlock();
  return internal_iter;
}

```

整个过程很明确：

1. 加入mem的iterator
2. 加入imm_的iterator
3. 加入外存levels的iterator
4. 组合成一个MergingIterator（最终的iter）

我们直到MeringIterator中含有多个children iterator, meringiterator在遍历的过程中，相当于把所有children iterator进行""排序".

mem和imm的iterator都很简单，是对skiplist的iter的封装。我们重点再看看levels的iter，==AddIterators函数==：

## 2. AddIterators

```c++
void Version::AddIterators(const ReadOptions& options,
                           std::vector<Iterator*>* iters) {
  // Merge all level zero files together since they may overlap
   // 加入所有level0的files iterator
  for (size_t i = 0; i < files_[0].size(); i++) {
    iters->push_back(vset_->table_cache_->NewIterator(
        options, files_[0][i]->number, files_[0][i]->file_size));
  }

   // 对于levels>0, 每层加入一个concatenating iterator即可。
  // For levels > 0, we can use a concatenating iterator that sequentially
  // walks through the non-overlapping files in the level, opening them
  // lazily.
  for (int level = 1; level < config::kNumLevels; level++) {
    if (!files_[level].empty()) {
      iters->push_back(NewConcatenatingIterator(options, level));
    }
  }
}
```

这里的工作也很简单：

1. 加入所有level0的files iterator
2. 对于levels>0, 每层加入一个concatenating iterator

那什么是concatenating iterator？

## 3. ConcatenatingIterator

==NewConcatenatingIterator==

```c++
Iterator* Version::NewConcatenatingIterator(const ReadOptions& options,
                                            int level) const {
  return NewTwoLevelIterator(
      new LevelFileNumIterator(vset_->icmp_, &files_[level]), // index iter 为 LevelFileNumIterator, 即存放的是SStable的FileMetaData。 key是sstable的最大key，value是sstable的 file number和file size
      &GetFileIterator, vset_->table_cache_, options);	// data iter是 sstable的iterator，即TwoLevelIterator
}
```

ConcatenatingIterator是用来遍历出level0的其他level的iterator。 依然是一个组合iterator，如下图：

![](https://pic.downk.cc/item/5f8441891cd1bbb86b045762.png)

## 4. 总结

InternalIterator是一个功能Iterator，功能是遍历整个leveldb系统。包括mem,imm, 以及各level的sstable。本质上是一个MergingIterator.如下图：

![](https://pic.downk.cc/item/5f8441901cd1bbb86b045c7f.png)