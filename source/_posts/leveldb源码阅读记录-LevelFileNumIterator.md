---
title: leveldb源码阅读记录-LevelFileNumIterator
categories: leveldb
abbrlink: 14e901ed
date: 2020-10-12 18:50:44
tags:
---

## LevelFileNumIterator

LevelFileNumIterator用于访问一个level中的file。

其中**ke存放的是 指定要访问的file的最大key。 value为16字节数据，用于存放文件序列号和文件size**。

> 原文注释
>
> // An internal iterator.  For a given version/level pair, yields
>
> // information about the files in the level.  For a given entry, key()
>
> // is the largest key that occurs in the file, and value() is an
>
> // 16-byte value containing the file number and file size, both
>
> // encoded using EncodeFixed64.

LevelFileNumIterator代码很短，直接上所有代码吧。

<!--more-->

```c++
class Version::LevelFileNumIterator : public Iterator {
 public:
  LevelFileNumIterator(const InternalKeyComparator& icmp,
                       const std::vector<FileMetaData*>* flist)
      : icmp_(icmp), flist_(flist), index_(flist->size()) {  // Marks as invalid
  }
  bool Valid() const override { return index_ < flist_->size(); }
  void Seek(const Slice& target) override {
     // 定位到的key，要不等于target，要不是第一个大于target的key
    index_ = FindFile(icmp_, *flist_, target);
  }
  void SeekToFirst() override { index_ = 0; }
  void SeekToLast() override {
    index_ = flist_->empty() ? 0 : flist_->size() - 1;
  }
  void Next() override {
    assert(Valid());
    index_++;
  }
  void Prev() override {
    assert(Valid());
    if (index_ == 0) {
      index_ = flist_->size();  // Marks as invalid
    } else {
      index_--;
    }
  }
  Slice key() const override {
    assert(Valid());
    return (*flist_)[index_]->largest.Encode();
  }
  Slice value() const override {
    assert(Valid());
    EncodeFixed64(value_buf_, (*flist_)[index_]->number);
    EncodeFixed64(value_buf_ + 8, (*flist_)[index_]->file_size);
    return Slice(value_buf_, sizeof(value_buf_));
  }
  Status status() const override { return Status::OK(); }

 private:
  const InternalKeyComparator icmp_;
  const std::vector<FileMetaData*>* const flist_;
  uint32_t index_;

  // Backing store for value().  Holds the file number and size.
  mutable char value_buf_[16];
};
```

理解这个iterator主要是只用知道 flist_ 存放的是一堆sstable元数据文件即可。这个iterator相当于是对一个sstable的数组的遍历的封装。

Seek函数中FindFile依然用了二分查找了确定位置。其余都比较简单。不再赘述。

## 在哪里用到了LevelFileNumIterator?

就目前的分析来看，leveldb并没有直接使用levelFileNumberIterator. 而是将它进一步封装在TwoLevelIterator中，所以下一篇我们就分析一下**TwoLevelIterator**。

```c++
Iterator* VersionSet::MakeInputIterator(Compaction* c) {
	xxx		
    // Create concatenating iterator for the files from this level
    list[num++] = NewTwoLevelIterator(
      new Version::LevelFileNumIterator(icmp_, &c->inputs_[which]),
       &GetFileIterator, table_cache_, options);
    xxx
  return result;
}
```

