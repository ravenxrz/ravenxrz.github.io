---
title: leveldb源码阅读记录-TwoLevelIterator
categories: leveldb
abbrlink: c419bb35
date: 2020-10-12 18:52:46
tags:
---

TwoLevelIterator是用来访问可用 index域和 data域访问的数据。

如，sstable中的data block和index block。就是分别将index block的block iter和data block的block iter赋值给index iter和data iter.

<!--more-->

## 0. TwoLevelIterator内部成员

```c++
class TwoLevelIterator : public Iterator {
 ...

  BlockFunction block_function_;
  void* arg_;
  const ReadOptions options_;
  Status status_;
  IteratorWrapper index_iter_;	// 两个iter warpper， 
  IteratorWrapper data_iter_;  // May be nullptr
  // If data_iter_ is non-null, then "data_block_handle_" holds the
  // "index_value" passed to block_function_ to create the data_iter_.
  std::string data_block_handle_;
};
```

注意到TwoLevelIterator中的成员函数主要包括， `block_function_, index_iter和data_iter_`. 后面两个iter是一个包装类，主要提供了cache策略，减少由于虚函数调用带来的开销。

## 1. 创建一个TwoLevelIterator

```c++
Iterator* NewTwoLevelIterator(Iterator* index_iter,
                              BlockFunction block_function, void* arg,
                              const ReadOptions& options) {
    return new TwoLevelIterator(index_iter, block_function, arg, options);
}
```

一个实际调用的例子（这部分VersionSet::MakeInputIterator中，实际上这个函数是在DoCompaction过程中被调用）：

```c++
   list[num++] = NewTwoLevelIterator(
            new Version::LevelFileNumIterator(icmp_, &c->inputs_[which]),
            &GetFileIterator, table_cache_, options);
```

- index_iter等于我们上篇文章中说的LevelFileNumIterator。
- block_function = GetFileIterator
- table_cache_，options，作为block_function的参数

## 2. Seek

```c++
void TwoLevelIterator::Seek(const Slice& target) {
      index_iter_.Seek(target);
      InitDataBlock();
      if (data_iter_.iter() != nullptr) data_iter_.Seek(target);
      SkipEmptyDataBlocksForward();
}
```

首先将index_iter定位到target。

### InitDataBlock

```c++
void TwoLevelIterator::InitDataBlock() {
  if (!index_iter_.Valid()) {
    SetDataIterator(nullptr);
  } else {
     // handle = 文件序列号和文件size
    Slice handle = index_iter_.value();
    if (data_iter_.iter() != nullptr &&
        handle.compare(data_block_handle_) == 0) {
      // data_iter_ is already constructed with this iterator, so
      // no need to change anything
    } else {
      Iterator* iter = (*block_function_)(arg_, options_, handle);
      data_block_handle_.assign(handle.data(), handle.size());
      SetDataIterator(iter);
    }
  }
}
```

首次调用InitiDataBlock，会走到:

```c++
      Iterator* iter = (*block_function_)(arg_, options_, handle);
      data_block_handle_.assign(handle.data(), handle.size());
      SetDataIterator(iter);
```

**以下面的例子来说明这个创建过程。**

```c++
 list[num++] = NewTwoLevelIterator(
            new Version::LevelFileNumIterator(icmp_, &c->inputs_[which]),
            &GetFileIterator, table_cache_, options);
```

- block_function_是GetFileIterator
- arg\_ = table_cache_
- options_ = ReadOptions
- handle = 文件序列号和文件size

现在深入到GetFileIterator中：


```c++
static Iterator* GetFileIterator(void* arg, const ReadOptions& options,
                                 const Slice& file_value) {
   // 强转为table cache
  TableCache* cache = reinterpret_cast<TableCache*>(arg);
  if (file_value.size() != 16) {
    return NewErrorIterator(
        Status::Corruption("FileReader invoked with unexpected value"));
  } else {
      // 根据file number和file size ，构造一个cache iterator
    return cache->NewIterator(options, DecodeFixed64(file_value.data()),
                              DecodeFixed64(file_value.data() + 8));
  }
}
```

到这里可以得到结论， TwoLevelIterator由两个iterator组成，一个是LevelFileNumIterator， 用于提供数据索引需要的元信息。另一个是cache iterator, 用于实际访问数据。

![](https://pic.downk.cc/item/5f8441d31cd1bbb86b048b26.png)

注意：这里LevelFileNumIterator和Table Iterator实际上又会被IteratorWrapper warp起来。（上文也提到了）。

==上一篇文章已经说过了LevelFileNumIterator, 之后再说说Table Iterator==

## 3. Next & Prev

一旦有了data_iter， next和prev就简单了。

```c++
void TwoLevelIterator::Next() {
  assert(Valid());
  data_iter_.Next();
   // 跳过empty data block
  SkipEmptyDataBlocksForward();
}

void TwoLevelIterator::Prev() {
  assert(Valid());
  data_iter_.Prev();
  SkipEmptyDataBlocksBackward();
}
```

ok,其余几个seek操作都比较简单。

## 4. key & value

```c++
 Slice key() const override {
    assert(Valid());
    return data_iter_.key();
  }
  Slice value() const override {
    assert(Valid());
    return data_iter_.value();
  }
```

## 5. TwolevelIterator总结

TwoLevelIterator是一种组合的Iterator，其内部由两个iter组成：index_iter和data_iter. index_iter用于索引，它包含了用来创建data_iter的元数据，实际上真实的key value存放在data_iter中。  在TwolevelIterator的各个seek操作中，都是先由index_iter找到合适的位置（索引），得到用来创建data_iter的元数据（sstable的file  number和file size), 然后创建data_iter, 最终data_iter才seek到真实数据的位置。

## 6. TwolevelIterator的应用--Table::Iterator

之前通过Compaction过程中用到的iterator介绍了TwoleveIIterator。现在用Table的Iterator来进一步理解TwolevelIterator。先看看入口：

```c++
Iterator* Table::NewIterator(const ReadOptions& options) const {
  return NewTwoLevelIterator(
      rep_->index_block->NewIterator(rep_->options.comparator),	// index iter
      &Table::BlockReader, const_cast<Table*>(this), options);	// data iter
}
```

首先分析用来创建index iter的Block::NewIterator:

```c++
Iterator* Block::NewIterator(const Comparator* comparator) {
  if (size_ < sizeof(uint32_t)) {
    return NewErrorIterator(Status::Corruption("bad block contents"));
  }
  const uint32_t num_restarts = NumRestarts();
  if (num_restarts == 0) {
    return NewEmptyIterator();
  } else { // 正常情况下会走到这里
    return new Iter(comparator, data_, restart_offset_, num_restarts);
  }
}
```

从这里可以看到index iter本质上就是一个Block::Iter, 是用来遍历index_block. 我们曾在SSTable章节中说过，index_block和data_block采用完全相同的结构。所以可以用Block::Iter进行遍历。

现在，看TwoLevelIterator的data iter部分，data iter主要由NewTwoLevelIterator参数中 `&Table::BlockReader, const_cast<Table*>(this), options`构成。在`TwoLevelIterator::InitDataBlock`函数中会初始化data iter:

```c++
 Iterator* iter = (*block_function_)(arg_, options_, handle);
```

此时：

- block_function_ = BlockReader
- arg_ = const_cast<Table*>(this)
- options_ = options
- handle = index_iter->value(), 即index_block的value，也即index_block对应data_block的block_handle(offset和size）。

再看BlockReader：

```c++
// Convert an index iterator value (i.e., an encoded BlockHandle)
// into an iterator over the contents of the corresponding block.
Iterator* Table::BlockReader(void* arg, const ReadOptions& options,
                             const Slice& index_value) {
 
    // 这里省略的代码的工作为，根据index_value(即data_block位置信息)从block_cache中获取block
   ...


  Iterator* iter;
  if (block != nullptr) {
     // 创建Block::Iter
    iter = block->NewIterator(table->rep_->options.comparator);
    ...
  }
  return iter;
}
```

丛这里可以看到，BlockReader返回的也是一个Block::Iter，所以这个TwoleveIIterator的data iter就是data block的Block::Iter.

### 总结

Table的Iter，是一个TwoLevelIterator，这个Iterator由index iter和data iter组成。 index iter为Block::iter, datablock iter也为Block::Iter.