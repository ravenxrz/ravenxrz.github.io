---
title: leveldb源码阅读记录-MemTableIterator
categories: leveldb
abbrlink: '51540516'
date: 2020-10-12 18:36:12
tags:
---

## MemTableIterator

读这篇文章前，请先阅读 skiplist实现和memtable两篇文章。

MemTableIteraotr是用来遍历MemTable的Iterator。

<!--more-->

## 1. 基本定义

下面是它的代码，代码很短：

```c++
class MemTableIterator : public Iterator {
 public:
  explicit MemTableIterator(MemTable::Table* table) : iter_(table) {}

  MemTableIterator(const MemTableIterator&) = delete;
  MemTableIterator& operator=(const MemTableIterator&) = delete;

  ~MemTableIterator() override = default;

  bool Valid() const override { return iter_.Valid(); }
  void Seek(const Slice& k) override { iter_.Seek(EncodeKey(&tmp_, k)); }
  void SeekToFirst() override { iter_.SeekToFirst(); }
  void SeekToLast() override { iter_.SeekToLast(); }
  void Next() override { iter_.Next(); }
  void Prev() override { iter_.Prev(); }
  Slice key() const override { return GetLengthPrefixedSlice(iter_.key()); }
  Slice value() const override {
    Slice key_slice = GetLengthPrefixedSlice(iter_.key());
    return GetLengthPrefixedSlice(key_slice.data() + key_slice.size());
  }

  Status status() const override { return Status::OK(); }

 private:
  MemTable::Table::Iterator iter_;
  std::string tmp_;  // For passing to EncodeKey
};

```

可以看到，MemTableIterator是对MemTable::Table::Iterator的简单封装，也就是说，真正的iterator是MemTable::Table::Iterator，

MemTable::Table其实是SkipList的别名，所以最终就是对SkipList的Iterator的封装。所以要求对MemTable的SkipList实现有所了解。

虽然MemTableIterator只是对SkipList的Iter的薄薄封装，但这里有两个地方特殊，那就是key和value。

如果你之前阅读了本系列的ksiplist文章， 那应该知道skiplist中只有key没有value， key里面存放的东西就是本文开篇中给出的图。它既包含了key也包含了value。下图是iter中的key的布局：

![](https://pic.downk.cc/item/5f8442441cd1bbb86b04da1b.png)

## 2. key

```c++
 Slice key() const override { return GetLengthPrefixedSlice(iter_.key()); }

static Slice GetLengthPrefixedSlice(const char* data) {
  uint32_t len;
  const char* p = data;
  p = GetVarint32Ptr(p, p + 5, &len);  // +5: we assume "p" is not corrupted
  return Slice(p, len);
}
```

先获取internal key 的长度，然后得到Internal key。

## 3. value

```c++
  Slice value() const override {
    Slice key_slice = GetLengthPrefixedSlice(iter_.key());
    return GetLengthPrefixedSlice(key_slice.data() + key_slice.size());
  }
```

获取internal key后，给第二个GetLengthPrefixedSlice传入的参数

```c++
key_slice.data() + key_slice.size()
```

是value size字段的首地址，所以这里返回的是实际的value。

**到这里我们可以补充一条内容，MemTable中存放的key是整个MemTable key（包括internel key size 和 internal key), 但是对外部呈现的却是一个internal key。**