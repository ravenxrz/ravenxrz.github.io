---
title: leveldb源码阅读记录-BlockIter
abbrlink: 6f1eb48f
date: 2020-10-12 18:50:12
categories: leveldb
tags:
---

这篇文章我们一起来看看Block::Iter, 这个Iter顾名思义使用来访问一个block的Iter，block可以是sstable中的data block,或者index block。还记得data block的布局吗：

<!--more-->

一个data block由多条record组成，且内部采用了前缀压缩机制：

![](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/data_block_of_sstable.png)

一条Record的layout如下：

![](https://pic.downk.cc/item/5f8444041cd1bbb86b061575.png)

ok，有了block的认识，再看Block::Iter就很简单了。

## 1. 基本定义

```c++
class Block::Iter : public Iterator {
 private:
  const Comparator* const comparator_;
  const char* const data_;       // underlying block contents 	底层块数据
  uint32_t const restarts_;      // Offset of restart array (list of fixed32) 重启点offset
  uint32_t const num_restarts_;  // Number of uint32_t entries in restart array	有多少个重启点

  // current_ is offset in data_ of current entry.  >= restarts_ if !Valid
  uint32_t current_;		// 标明当前iterator走到哪个位置
  uint32_t restart_index_;  // Index of restart block in which current_ falls, current_落入到哪个restart区间
  std::string key_;
  Slice value_;
  Status status_;
	...
  }
```

## 2. 创建一个Iter

通过Block::NewIterator函数，可以创建一个Block::Iter

```c++
Iterator* Block::NewIterator(const Comparator* comparator) {
  if (size_ < sizeof(uint32_t)) {
    return NewErrorIterator(Status::Corruption("bad block contents"));
  }
  const uint32_t num_restarts = NumRestarts();
  if (num_restarts == 0) {
    return NewEmptyIterator();
  } else {		// 正常情况走这里
    return new Iter(comparator, data_, restart_offset_, num_restarts);
  }
}

```

## 3. Seek

Seek定位，看代码：

```c++
  void Seek(const Slice& target) override {
      // 通过二分查找，找到距离target最近的key，且key <target
    // Binary search in restart array to find the last restart point
    // with a key < target
    uint32_t left = 0;
    uint32_t right = num_restarts_ - 1;
    while (left < right) {
      uint32_t mid = (left + right + 1) / 2;
      uint32_t region_offset = GetRestartPoint(mid);
      uint32_t shared, non_shared, value_length;
      const char* key_ptr =		
          DecodeEntry(data_ + region_offset, data_ + restarts_, &shared,
                      &non_shared, &value_length);		// 解析当前重启点对应的key，shared，non_shared和value_length
      if (key_ptr == nullptr || (shared != 0)) {	// 重启点对应到的key，是没有前缀压缩的，所以shared 应该 = 0
        CorruptionError();
        return;
      }
      Slice mid_key(key_ptr, non_shared);
        // 二分查找
      if (Compare(mid_key, target) < 0) {
        // Key at "mid" is smaller than "target".  Therefore all
        // blocks before "mid" are uninteresting.
        left = mid;
      } else {
        // Key at "mid" is >= "target".  Therefore all blocks at or
        // after "mid" are uninteresting.
        right = mid - 1;
      }
    }
	
     // 上面的代码确定了target在哪个重启点的区间，在这个区间中进行线性查找即可。
    // Linear search (within restart block) for first key >= target
    SeekToRestartPoint(left);
    while (true) {
      if (!ParseNextKey()) {
        return;
      }
      if (Compare(key_, target) >= 0) {
        return;
      }
    }
  }

```

总体上，这里分为了两步：

1. 通过restart_point，进行二分查找，查找目标key(target)所在哪个重启点区间。所谓的重启点区间，举个例子：

   ![](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/data_block_of_sstable.png)

   上面由restart_[0] 到 restart\_[1]之间的区间，也即Record0 – Record 16就是一个区间。

2. 找到这个区间后，线性搜索即可找到目标key。

简单看一下如何解析一条record：

### 1. DecodeEntry

```c++
// Helper routine: decode the next block entry starting at "p",
// storing the number of shared key bytes, non_shared key bytes,
// and the length of the value in "*shared", "*non_shared", and
// "*value_length", respectively.  Will not dereference past "limit".
//
// If any errors are detected, returns nullptr.  Otherwise, returns a
// pointer to the key delta (just past the three decoded values).
static inline const char* DecodeEntry(const char* p, const char* limit,
                                      uint32_t* shared, uint32_t* non_shared,
                                      uint32_t* value_length) {
  if (limit - p < 3) return nullptr;
  *shared = reinterpret_cast<const uint8_t*>(p)[0];
  *non_shared = reinterpret_cast<const uint8_t*>(p)[1];
  *value_length = reinterpret_cast<const uint8_t*>(p)[2];
  if ((*shared | *non_shared | *value_length) < 128) {
    // Fast path: all three values are encoded in one byte each
    p += 3;
  } else {
    if ((p = GetVarint32Ptr(p, limit, shared)) == nullptr) return nullptr;
    if ((p = GetVarint32Ptr(p, limit, non_shared)) == nullptr) return nullptr;
    if ((p = GetVarint32Ptr(p, limit, value_length)) == nullptr) return nullptr;
  }

  if (static_cast<uint32_t>(limit - p) < (*non_shared + *value_length)) {
    return nullptr;
  }
  return p;
}
```

注释说得很清楚，这个函数就是用来解析一条record的 shared, non-shared和value_length字段。

### 2. SeekToRestartPoint

根据restart_index索引到具体data block的位置。这里的value\_ 虽然长度未0，但是相当于起到了一个位置标记的作用，为后续ParseKey做铺垫。

```c
  void SeekToRestartPoint(uint32_t index) {
    key_.clear();
    restart_index_ = index;
    // current_ will be fixed by ParseNextKey();

    // ParseNextKey() starts at the end of value_, so set value_ accordingly
    uint32_t offset = GetRestartPoint(index);
    value_ = Slice(data_ + offset, 0);
  }

```

## 4. Next

```c++
  void Next() override {
    assert(Valid());
    ParseNextKey();
  }
```

### 1. ParseNextKey

重点在ParseNextKey：

```c++
  bool ParseNextKey() {
    current_ = NextEntryOffset();		// 解析当前entry的位置， current_保存的是当前entry距离 data_的offset
    const char* p = data_ + current_;	// 指向当前entry
    const char* limit = data_ + restarts_;  // Restarts come right after data // data_ + restarts_所指向的位置是 有效data的下边界
    if (p >= limit) {
      // No more entries to return.  Mark as invalid.
      current_ = restarts_;
      restart_index_ = num_restarts_;
      return false;
    }

    // Decode next entry
    uint32_t shared, non_shared, value_length;
    p = DecodeEntry(p, limit, &shared, &non_shared, &value_length);
    if (p == nullptr || key_.size() < shared) {
      CorruptionError();
      return false;
    } else {
      key_.resize(shared);		// 首先设置key的前缀， 这里只用resize就能得到前缀，因为对于重启点对应的key来说，没有共享前缀，所以非重启点对应的key的前缀都是来自 重启点对应的key
      key_.append(p, non_shared);		// 添加非共享长度，得到最终key
      value_ = Slice(p + non_shared, value_length);		// 得到value
      while (restart_index_ + 1 < num_restarts_ &&
             GetRestartPoint(restart_index_ + 1) < current_) {	// 移动到之后的重启点，保证这个重启点对应的entry位置在current_之下
        ++restart_index_;
      }
      return true;
    }
  }
```

这个函数算是Block::Iter的核心了，主要工作是：

1. 定位目标entry（record）所在位置
2. 根据前缀压缩机制，解析并组装key，得到最后的value_
3. 更新restart_index

## 5. Prev

```c++
  void Prev() override {
    assert(Valid());

     // 定位到上一个重启点，保证上一个key在这个重启点和下个重启点的区间内
    // Scan backwards to a restart point before current_
    const uint32_t original = current_;
    while (GetRestartPoint(restart_index_) >= original) {
      if (restart_index_ == 0) {
        // No more entries
        current_ = restarts_;
        restart_index_ = num_restarts_;
        return;
      }
      restart_index_--;
    }

     // 定义
    SeekToRestartPoint(restart_index_);
      // 向下线性扫描，直到找到上一个key
    do {
      // Loop until end of current entry hits the start of original entry
    } while (ParseNextKey() && NextEntryOffset() < original);
  }

```

理解了前面的Seek和Next函数，这个Prev就很简单了。注释有说明，我们再结合图说明一下，加深印象：

![](https://pic.downk.cc/item/5f845d5e1cd1bbb86b1598f9.png)

## 6. key & value

```c++
  Slice key() const override {
    assert(Valid());
    return key_;
  }
  
    Slice value() const override {
    assert(Valid());
    return value_;
  }
```

这两个函数没什么好说的，前期的Seek，Next，Prev中会设置两个成员变量。

## 7. 总结

本文我们分析了Block::Iter的设计与实现。其实只是了解了它使用来访问Block的，那么掌握了Block的存储布局方式，就非常好理解了。重点函数是ParseNextKey，里面反解析了“前缀压缩机制”，拼装出最后的key，得到value。另外，由于一个block内的数据是有序的，所以可以通过 restart_point来二分查找，从而加速访问性能。

