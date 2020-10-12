---
title: leveldb源码阅读记录-Memtable
categories: leveldb
abbrlink: aba77258
date: 2020-10-12 11:12:58
tags:
---

上文我们介绍了Skiplist，它是leveldbMemtable的底层数据结构，这篇文章，我们一起来分析MemTable。

<!--more-->

## 1. MemTable定义

```c++
class MemTable {
 public:
  // MemTables are reference counted.  The initial reference count
  // is zero and the caller must call Ref() at least once.
  explicit MemTable(const InternalKeyComparator& comparator);

  MemTable(const MemTable&) = delete;
  MemTable& operator=(const MemTable&) = delete;

  // Increase reference count.
  void Ref() { ++refs_; }

  // Drop reference count.  Delete if no more references exist.
  void Unref() {
    --refs_;
    assert(refs_ >= 0);
    if (refs_ <= 0) {
      delete this;
    }
  }

  // Returns an estimate of the number of bytes of data in use by this
  // data structure. It is safe to call when MemTable is being modified.
  size_t ApproximateMemoryUsage();

  // Return an iterator that yields the contents of the memtable.
  //
  // The caller must ensure that the underlying MemTable remains live
  // while the returned iterator is live.  The keys returned by this
  // iterator are internal keys encoded by AppendInternalKey in the
  // db/format.{h,cc} module.
  Iterator* NewIterator();

  // Add an entry into memtable that maps key to value at the
  // specified sequence number and with the specified type.
  // Typically value will be empty if type==kTypeDeletion.
  void Add(SequenceNumber seq, ValueType type, const Slice& key,
           const Slice& value);

  // If memtable contains a value for key, store it in *value and return true.
  // If memtable contains a deletion for key, store a NotFound() error
  // in *status and return true.
  // Else, return false.
  bool Get(const LookupKey& key, std::string* value, Status* s);

 private:
  friend class MemTableIterator;
  friend class MemTableBackwardIterator;

  struct KeyComparator {
    const InternalKeyComparator comparator;
    explicit KeyComparator(const InternalKeyComparator& c) : comparator(c) {}
    int operator()(const char* a, const char* b) const;
  };
	
   // 注意skiplist存放的key是 const char *类型
  typedef SkipList<const char*, KeyComparator> Table;

  ~MemTable();  // Private since only Unref() should be used to delete it

  KeyComparator comparator_;
  int refs_;
  Arena arena_;
  Table table_;
};

```

MemTable里面有3个重要的数据结构：

1. KeyComparator ，比较器，在Get操作中会用到。
2. Arena，内存分配器，在Add操作中用到。
3. SkipList，核心数据结构，管理数据。 **注意key就是一个char *， 内部比较器采用的是InternalKeyComparator.**

## 2. Key类型分类

在说memtable的get和add操作前，我们先了解一下 **leveldb 中**的几种key类型。

在leveldb中做search操作时，search的过程大概是:

memtable->immutable memtable -> sstable

这里涉及到2个search用到的key，一个在memtable中用，一个在sstable中用的key。

其实还有1个key，那就是用户自己输入的key，user-key。

总结起来就3种key，如下：

```
memtable: 逻辑上称为memtable_key

sstalbe: 逻辑上称为internal_key

key: 用户提供的key，我们称之为user_key
```

leveldb是如何表示这3种key的？看下图：

![img](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb_key.png)

leveldb用一种类LookupKey包含了这3种key，我们要用的memtable_key其实就是Lookupkey。这一点，也可以从MemTable类的成员函数中可以看出，如Get操作：

```c++
// If memtable contains a value for key, store it in *value and return true.
// If memtable contains a deletion for key, store a NotFound() error
// in *status and return true.
// Else, return false.
bool Get(const LookupKey& key, std::string* value, Status* s);
```

## 3. Add 操作

对key分类有了初步的认识后 ，我们就来看MemTable是如何将一个user key封装成一个memtable_key，然后将key value插入到skiplist中的。

```c++
  // Add an entry into memtable that maps key to value at the
  // specified sequence number and with the specified type.
  // Typically value will be empty if type==kTypeDeletion.
  void Add(SequenceNumber seq, ValueType type, const Slice& key,
           const Slice& value);
```

上面是Add的函数签名，注意到函数参数中有一个ValueType类型，我们知道leveldb中删除一个key并不会inplace update,而是插入一个带有删除标记的key。ValueType就是表示当前插入的是一个正常key还是一个删除key:

```c++
// Value types encoded as the last component of internal keys.
// DO NOT CHANGE THESE ENUM VALUES: they are embedded in the on-disk
// data structures.
enum ValueType { kTypeDeletion = 0x0, kTypeValue = 0x1 };
```

好了，现在来正式看看Add函数:

```c++
void MemTable::Add(SequenceNumber s, ValueType type, const Slice& key,
                   const Slice& value) {
  // Format of an entry is concatenation of:
  //  key_size     : varint32 of internal_key.size()
  //  key bytes    : char[internal_key.size()]
  //  value_size   : varint32 of value.size()
  //  value bytes  : char[value.size()]
  size_t key_size = key.size();
  size_t val_size = value.size();
  size_t internal_key_size = key_size + 8;
  const size_t encoded_len = VarintLength(internal_key_size) +
                             internal_key_size + VarintLength(val_size) +
                             val_size;
  char* buf = arena_.Allocate(encoded_len);
  char* p = EncodeVarint32(buf, internal_key_size);
  std::memcpy(p, key.data(), key_size);
  p += key_size;
  EncodeFixed64(p, (s << 8) | type);
  p += 8;
  p = EncodeVarint32(p, val_size);
  std::memcpy(p, value.data(), val_size);
  assert(p + val_size == buf + encoded_len);
  table_.Insert(buf);
}
```

代码很短，基本就是申请一个buf，然后填充数据,最后将buf插入到skiplist中。具体填充的字段如下：

![](https://pic.downk.cc/item/5f8442441cd1bbb86b04da1b.png)

## 4. Get操作

```c++
// If memtable contains a value for key, store it in *value and return true.
// If memtable contains a deletion for key, store a NotFound() error
// in *status and return true.
// Else, return false.
bool MemTable::Get(const LookupKey& key, std::string* value, Status* s) {
  Slice memkey = key.memtable_key();
  Table::Iterator iter(&table_);
    // 在skip中找到相应的memkey对应的node
  iter.Seek(memkey.data());
  if (iter.Valid()) {
    // entry format is:
    //    klength  varint32
    //    userkey  char[klength]
    //    tag      uint64
    //    vlength  varint32
    //    value    char[vlength]
    // Check that it belongs to same user key.  We do not check the
    // sequence number since the Seek() call above should have skipped
    // all entries with overly large sequence numbers.
    const char* entry = iter.key();
    uint32_t key_length;
      // 提取memtable key。包括 klength + userkey + tag 字段
    const char* key_ptr = GetVarint32Ptr(entry, entry + 5, &key_length);
    if (comparator_.comparator.user_comparator()->Compare(
            Slice(key_ptr, key_length - 8), key.user_key()) == 0) {	// 如果找到的key和需要get的key相同
      // Correct user key
        // 获取序列号+type字段
      const uint64_t tag = DecodeFixed64(key_ptr + key_length - 8);
      switch (static_cast<ValueType>(tag & 0xff)) {	// & 0xff 取最后1个字节
        case kTypeValue: {	// 有value
          Slice v = GetLengthPrefixedSlice(key_ptr + key_length);
          value->assign(v.data(), v.size());
          return true;
        }
        case kTypeDeletion:	// deletion 操作
          *s = Status::NotFound(Slice());
          return true;
      }
    }
  }
  return false;
}
```

get的流程：

1. 根据传入key，到skiplist中找到相应的node
2. 提取 node中的 memtable_key, 进一步从memtable_key中提取 user_key, 比较这个user_key和用户传入的user_key是否相同
3. 提取 memtable_key中的tag(包括序列号和type)
4. 查看type是哪种类型：
   1. 正常有value，保存value
   2. 是删除的key，在Status s中保存NotFound的结果。

## 5. 在skiplist如何进行key的比较 & 如果key相同，新旧数据如何确定？

通过InternalKeyComparator::Compare来确定。

规则：

1. 按照key的升序
2. key相同，按照序列号降序
3. 序列号相同，按照type降序

```c++
int InternalKeyComparator::Compare(const Slice& akey, const Slice& bkey) const {
  // Order by:
  //    increasing user key (according to user-supplied comparator)
  //    decreasing sequence number
  //    decreasing type (though sequence# should be enough to disambiguate)
  int r = user_comparator_->Compare(ExtractUserKey(akey), ExtractUserKey(bkey));
  if (r == 0) {
    const uint64_t anum = DecodeFixed64(akey.data() + akey.size() - 8);
    const uint64_t bnum = DecodeFixed64(bkey.data() + bkey.size() - 8);
    if (anum > bnum) {
      r = -1;
    } else if (anum < bnum) {
      r = +1;
    }
  }
  return r;
}
```

**同时，seq number越大，代表数据越新。**

## 6. 迭代器Iterator

这部分请看 MemTableIterator 。

## 参考：

1. [Leveldb源码笔记之读操作](http://blog.1feng.me/2016/09/10/leveldb-read/)