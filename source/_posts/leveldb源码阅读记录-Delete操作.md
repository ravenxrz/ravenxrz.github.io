---
title: leveldb源码阅读记录-Delete操作
categories: leveldb
abbrlink: 81b3993e
date: 2020-10-12 19:00:00
tags:
---

前面我们已经分析完了Put操作，本篇再来看看Delete操作。 在leveldb中，一次Delete并不是去找到原kv数据然后从数据库中删除，而是将Delete看成一种特殊的Put，标志某个key已经被删除。

<!--more-->

## 1. Delete函数

```c++
Status DB::Delete(const WriteOptions& opt, const Slice& key) {
  WriteBatch batch;
  batch.Delete(key);
  return Write(opt, &batch);
}
```

代码是不是非常简单，而Write函数前面我们已经做了分析，所以只用看下 `batch.Delete(key);`

```c++
void WriteBatch::Delete(const Slice& key) {
  WriteBatchInternal::SetCount(this, WriteBatchInternal::Count(this) + 1);
  rep_.push_back(static_cast<char>(kTypeDeletion));
  PutLengthPrefixedSlice(&rep_, key);
}
```

首先加入一个 `kTypeDeletion`标记，这个标记用于表明本次操作是一次对key的删除操作。 后面将整个key封装到rep_中。  而`PutLengthPrefixedSlice`我们也在[编码](https://www.ravenxrz.ink/archives/8f115521.html)中讲过了，仅仅是将一个字符串的长度和字符串本身编码在一起。

## 2. 为什么要这样做 & 为什么可以这样做？

为什么要这样做？ leveldb是一种随机写友好的数据库，如果我们每次Delete都去找到原数据（原数据可能存在多层sstable中）再删除，这样的效率非常低。严重影响写性能。

为什么可以这样做？ leveldb是有新旧数据分层的概念的， 在低层的(level0)数据始终比高层(level 1-6)的数据新。当然了，memtable中的数据又会比sstable中的数据新。所以当插入一条删除记录，它肯定是最新的，位于低层，leveldb在后序的读取操作中，也是由低层到高层的读取，一旦读取到删除标志就意味着更高层的数据都是无效的了。这一点在做[DBIter](https://www.ravenxrz.ink/archives/f4c6f796.html)的分析时也说过。

