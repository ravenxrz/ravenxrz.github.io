---
title: leveldb源码阅读记录-DBIter
categories: leveldb
abbrlink: f4c6f796
date: 2020-10-12 18:57:40
tags:
---

# DBIter

Leveldb数据库的MemTable和sstable文件的存储格式都是InternalKey(userkey, seq, type) => uservalue。
**DBIter把同一个userkey在DB中的多条记录合并为一条，综合考虑了userkey的序号、删除标记、和写覆盖等等因素**。DBIter只会把userkey最新（seq最大的就是最新的，相同userkey的老记录（seq较小的）不会让上层看到）的一条记录展现给用户，另外如果这条最新的记录是删除类型，则会跳过该记录，否则，遍历时会把已删除的key列举出来。

<!--more-->

## 1. 创建DBIterator

```c++
Iterator* DBImpl::NewIterator(const ReadOptions& options) {
  SequenceNumber latest_snapshot;
  uint32_t seed;
  Iterator* iter = NewInternalIterator(options, &latest_snapshot, &seed);
  return NewDBIterator(this, user_comparator(), iter,
                       (options.snapshot != nullptr
                            ? static_cast<const SnapshotImpl*>(options.snapshot)
                                  ->sequence_number()
                            : latest_snapshot),
                       seed);
}

Iterator* NewDBIterator(DBImpl* db, const Comparator* user_key_comparator,
                        Iterator* internal_iter, SequenceNumber sequence,
                        uint32_t seed) {
  return new DBIter(db, user_key_comparator, internal_iter, sequence, seed);
}
```

## 2. DBIterator的基本定义

```c++
class DBIter: public Iterator {
  enum Direction {
    kForward,
    kReverse
  };
private:
  DBImpl* db_;
  const Comparator* const user_comparator_;	//比较iter间userkey
  Iterator* const iter_;				// 是一个MergingIterator
  SequenceNumber const sequence_;		// DBIter只能访问到比sequence_小的kv对，这就方便了老版本（快照）数据库的遍历

  Status status_;
  std::string saved_key_;     //用于方向遍历 direction_==kReverse时才有效
  std::string saved_value_;   //用于反向遍历 direction_==kReverse时才有效
  Direction direction_;
  bool valid_;

  Random rnd_;
  ssize_t bytes_counter_;
}
```

后面再说DBIter的作用，这里先介绍下sequence_的作用。

### 值得一提的sequence_

在创建DBIter时，我们将系统的一个snapshot的sequence_number传递给了DBIter：

```c++
  return NewDBIterator(this, user_comparator(), iter,
                       (options.snapshot != nullptr	//!! 这里
                            ? static_cast<const SnapshotImpl*>(options.snapshot)
                                  ->sequence_number()		
                            : latest_snapshot),
                       seed);
```

那它代表什么含义？

我们从一次WriteBatch出发，看看序列号是怎么使用及修改的。先给出结论图：

![](https://pic.downk.cc/item/5f8440df1cd1bbb86b03d7eb.png)

**我们知道一个WriteBatch对一次批量写操作的封装。**先看看一个WriteBatch的结构：

```c++
class LEVELDB_EXPORT WriteBatch {
 public:
  class LEVELDB_EXPORT Handler {
   public:
    virtual ~Handler();
    virtual void Put(const Slice& key, const Slice& value) = 0;
    virtual void Delete(const Slice& key) = 0;
  };

    ...
    
  WriteBatch();
private:
  friend class WriteBatchInternal;

  std::string rep_;  // See comment in write_batch.cc for the format of rep_
};
```

其内部只有一个rep_的成员变量，其结构如下：

```c++
// WriteBatch::rep_ :=
//    sequence: fixed64
//    count: fixed32
//    data: record[count]
```

**由8字节序列号，4字节count，和实际数据record数组组成。**

![](https://pic.downk.cc/item/5f8440ec1cd1bbb86b03df83.png)

==一次Put操作：==

```c++
void WriteBatch::Put(const Slice& key, const Slice& value) {
  WriteBatchInternal::SetCount(this, WriteBatchInternal::Count(this) + 1);
  rep_.push_back(static_cast<char>(kTypeValue));
  PutLengthPrefixedSlice(&rep_, key);
  PutLengthPrefixedSlice(&rep_, value);
}
```

Put操作很简单，设置count+1， 然后将key value append 到rep_末尾。

**这里我们知道了count代表rep_中有效的kv pair数量。**

有了batch，我们如何应用batch（即如何将batch写入到leveldb系统中？）。对于一次写，除了写入到Log中来保证持久性以外，我们首先做的就是将数据插入到memtable中。这个过程如下函数：

**注意，这里终于要说到seequence_的意义了**

```c++
Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates) {
	...
	status = WriteBatchInternal::InsertInto(write_batch, mem_);
	...
}
```

```c++
Status WriteBatchInternal::InsertInto(const WriteBatch* b, MemTable* memtable) {
  MemTableInserter inserter;
  inserter.sequence_ = WriteBatchInternal::Sequence(b);	// !! 这里设置到了sequence number
  inserter.mem_ = memtable;
  return b->Iterate(&inserter);
}

SequenceNumber WriteBatchInternal::Sequence(const WriteBatch* b) {
  return SequenceNumber(DecodeFixed64(b->rep_.data()));		// 取出 batch的rep_的前8个字节作为sequence　number
}
```

现在在MemTableInserter中保存了一个 batch最开始的sequence number。（至于batch的sequence number是在哪儿初始化的，我们之后再说）。

再回到问题，如何将batch中的多个kv pair应用到系统中？核心就在b->Iterate(&inserter)中。

```c++
Status WriteBatch::Iterate(Handler* handler) const {
  ...
  while (!input.empty()) {
      ...
    switch (tag) {
      case kTypeValue:	// 普通值
        if (GetLengthPrefixedSlice(&input, &key) &&
            GetLengthPrefixedSlice(&input, &value)) {
          handler->Put(key, value);
        } else {
          return Status::Corruption("bad WriteBatch Put");
        }
        break;
      case kTypeDeletion:	// 删除值
        if (GetLengthPrefixedSlice(&input, &key)) {
          handler->Delete(key);
        } else {
          return Status::Corruption("bad WriteBatch Delete");
        }
        break;
      default:
        return Status::Corruption("unknown WriteBatch tag");
    }
  }
  if (found != WriteBatchInternal::Count(this)) {
    return Status::Corruption("WriteBatch has wrong count");
  } else {
    return Status::OK();
  }
}
```

注意到这就是一个循环遍历batch中的各kv pair。然后调用handler->Put或handler->Delete来应用。而handler就是MemTableInserter：

```c++
  void Put(const Slice& key, const Slice& value) override {
    mem_->Add(sequence_, kTypeValue, key, value);
    sequence_++;	// sequence的修改
  }
  void Delete(const Slice& key) override {
    mem_->Add(sequence_, kTypeDeletion, key, Slice());
    sequence_++; 	// sequence的修改
  }
```

**从这里终于可以看出，每插入一对kv，就会递增一次sequence**

现在还剩最后一个问题，batch的sequence number是如何初始化的？

在WriteBatchInternal类中有个SetSequence函数：

```c++
void WriteBatchInternal::SetSequence(WriteBatch* b, SequenceNumber seq) {
  EncodeFixed64(&b->rep_[0], seq);
}
```

在这里，将rep_的前8个字节设置为这个batch的起始sequence。那谁调用的这个函数?

在DBImpl::Wirte函数中：

```c++
Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates) {
	...
  // May temporarily unlock and wait.
  Status status = MakeRoomForWrite(updates == nullptr);
  uint64_t last_sequence = versions_->LastSequence();	// !! sequence从version中获得
  Writer* last_writer = &w;
  if (status.ok() && updates != nullptr) {  // nullptr batch is for compactions
    WriteBatch* write_batch = BuildBatchGroup(&last_writer);
    WriteBatchInternal::SetSequence(write_batch, last_sequence + 1);	// 注入sequence
```

当成这次batch的写入后，又会反过来更新version中的序列号：

```c++
    last_sequence += WriteBatchInternal::Count(write_batch);	// version版本号 = 旧版本号+此次写入的batch的kv数量

    ...
    if (write_batch == tmp_batch_) tmp_batch_->Clear();

    versions_->SetLastSequence(last_sequence);		// 应用
```

**所以，究其根本，系统中插入的每对kv的序列号，最初是由当前系统version的序列号指定的，然后在后序插入的kv中逐渐递增。**

有了上面对sequence的理解，我们就能更好的理解DBIter的作用了。

### DBIter的作用（功能）

**iter_是由NewInternalIterator创建的一个MergingIterator**，通过MergingIterator可以实现多个有序数据集合的归并操作。其中包含多个child iterator组成的集合。对MergingIterator的遍历会有序的遍历其child iterator中的每个元素。
因为**iter_遍历的是数据库的每一条记录。它是以InternalKey(userkey, seq, type)为遍历粒度的，只要InternalKey中任意一个组成元素不同，MergingIterator就认为他们是不同的kv对。**
而**DBIter是以userkey为遍历粒度的，只要记录的userkey相同，那么DBIter就认为他们是一条记录（不同版本），sqe越大代表该记录越新。每次迭代将跳到下一个不同userkey的记录，且==DBIter在遍历一个InternalKey时仅会检索InternalKey->seq 小于 DBIter创建时所初始化的seq号。==**举个例子：

![](https://pic.downk.cc/item/5f8440a11cd1bbb86b03a6c3.png)

上面表示了6个InternalKey，冒号前为user_key,冒号后为序列号。现假设创建DBIter时，所初始化的seq为2. 则DBIter在从前往后遍历时，将会直接跳过 key1:6,key1:5,key1:4 和key2:3.  只会从key2:2开始遍历。

下面我们来分析DBIter的各个操作。



## 3.FindNextUserEntry

```c++
void DBIter::FindNextUserEntry(bool skipping, std::string* skip) {
  // Loop until we hit an acceptable entry to yield
  assert(iter_->Valid());
  assert(direction_ == kForward);
  do {
    ParsedInternalKey ikey;	
    if (ParseKey(&ikey) && ikey.sequence <= sequence_) {	// 根据当前iter_的key封装成一个新ikey，同时要求ikey的序列号小于创建DBIter时传入的序列号
      switch (ikey.type) {
        case kTypeDeletion:
          // Arrange to skip all upcoming entries for this key since
          // they are hidden by this deletion.
          SaveKey(ikey.user_key, skip);		
          skipping = true;		// 如果当前ikey的类型是删除，将跳过整个 user_key = ikey.user_key()的nodes
          break;
        case kTypeValue:
          if (skipping &&	// 跳过
              user_comparator_->Compare(ikey.user_key, *skip) <= 0) {	// 旧条目不再需要
            // Entry hidden
          } else {	// 否则就是本 user_key 所在nodes中的最新节点
            valid_ = true;
            saved_key_.clear();
            return;
          }
          break;
      }
    }
    iter_->Next();	// 定位到下一个
  } while (iter_->Valid());
  saved_key_.clear();
  valid_ = false;
}
```

用一个例子来解释这个函数的作用：

![](https://pic.downk.cc/item/5f8440b51cd1bbb86b03b759.png)

上图中， 每个节点由3个字段组成，由冒号:分隔，第1个为user_key，第2个为版本号，第3个为节点类型（1代表普通值，0代表删除）。

假设当前要遍历所有key，且iter\_目前指向 key1:6:1. 则：

1. iter往下走到key1:5:1, 由于key1::5:1的user_key = key1:6:1的user_key即：

   ```c++
    if (skipping &&	// 跳过
                 user_comparator_->Compare(ikey.user_key, *skip) <= 0) {	// 旧条目不再需要
               // Entry hidden
    } 
   ```

   所以跳过这个节点。

2. iter往下走到key1:4:1, 由于key1::4:1的user_key = key1:6:1的user_key，所以跳过。

3. iter往下走到key2:3:0, 由于节点类型是0，删除节点。所以：

   ```c++
   case kTypeDeletion:
             // Arrange to skip all upcoming entries for this key since
             // they are hidden by this deletion.
             SaveKey(ikey.user_key, skip);		
             skipping = true;		// 如果当前ikey的类型是删除，将跳过整个 user_key = ikey.user_key()的nodes
             break;
   ```

   注意之类设置了skipping

4. iter往下走到key2:2:1，执行到：

   ```c++
   if (skipping &&	// 跳过
                 user_comparator_->Compare(ikey.user_key, *skip) <= 0) {	// 旧条目不再需要
               // Entry hidden
             }
   ```

   所以跳过。

5. 后面同理。

这次遍历的结果是，只得到了key:1:6:1(注意这是所有key1的节点中最新的节点)，所有key2都跳过了。

## 4.Seek

```c++
void DBIter::Seek(const Slice& target) {
  direction_ = kForward;
   // 首先在saved_key中封装一个internal key
  ClearSavedValue();
  saved_key_.clear();
  AppendInternalKey(&saved_key_,
                    ParsedInternalKey(target, sequence_, kValueTypeForSeek));
   // 注意之类的iter_是上一篇文章介绍过的InternalIterator.
  iter_->Seek(saved_key_);	// seek到合适位置
  if (iter_->Valid()) {		// 通过FindNextUserEntry找到当前最新key或者过滤掉已经被删除的key
    FindNextUserEntry(false, &saved_key_ /* temporary storage */);
  } else {
    valid_ = false;
  }
}
```

## 5. Next

```c++
void DBIter::Next() {
	...
    // Store in saved_key_ the current key so we skip it below.
    SaveKey(ExtractUserKey(iter_->key()), &saved_key_);	// 为后序FindNextUserEntry函数中进行跳过无用节点做铺垫

    // iter_ is pointing to current key. We can now safely move to the next to
    // avoid checking current key.
    iter_->Next();	// 定位到下一个节点
    if (!iter_->Valid()) {
        valid_ = false;
        saved_key_.clear();
        return;
    }
  
  	FindNextUserEntry(true, &saved_key_);
}
```

还是用例子来说：

![](https://pic.downk.cc/item/5f8440fa1cd1bbb86b03ea50.png)

假设当前iter_ 指向 key0:8:1。则Next函数的工作如下：

1. 在saved_key_中保存 key0:8:1
2. iter指向key0:7:1
3. 执行FindNextUserEntry
   1. 由于指定了FindNextUserEntry中的skipping为true，并且key0:7:1的user key = key0:8:1的user key，所以直接跳过key0:7:1。
   2. 现在iter来到了key1:6:1，有效，直接返回。

## 6. FindPrevUserEntry

向前遍历和向后遍历基本类似，但是由于是越靠前越新，所以向前遍历时，需要多往前尝试，直到找到一个新的不同的user_key.

```c++
void DBIter::FindPrevUserEntry() {
  assert(direction_ == kReverse);

  ValueType value_type = kTypeDeletion;
  if (iter_->Valid()) {
    do {
      ParsedInternalKey ikey;
      if (ParseKey(&ikey) && ikey.sequence <= sequence_) {
        if ((value_type != kTypeDeletion) &&		// 不是删除节点
            user_comparator_->Compare(ikey.user_key, saved_key_) < 0) {		// 遇到了新user_key
          // We encountered a non-deleted value in entries for previous keys,
          break;
        }
        value_type = ikey.type;
        if (value_type == kTypeDeletion) {	// 删除节点，清空saved_key和saved_value
          saved_key_.clear();
          ClearSavedValue();
        } else {			// 正常节点，保存saved_key和saved_value
          Slice raw_value = iter_->value();
          if (saved_value_.capacity() > raw_value.size() + 1048576) {
            std::string empty;
            swap(empty, saved_value_);
          }
          SaveKey(ExtractUserKey(iter_->key()), &saved_key_);
          saved_value_.assign(raw_value.data(), raw_value.size());
        }
      }
      iter_->Prev();
    } while (iter_->Valid());
  }

  if (value_type == kTypeDeletion) {	// 结束位置，无法再继续往前走
    // End
    valid_ = false;
    saved_key_.clear();
    ClearSavedValue();
    direction_ = kForward;
  } else {		// 正常情况走到这里
    valid_ = true;
  }
}
```



![](https://pic.downk.cc/item/5f8441241cd1bbb86b040b4b.png)

假设当前iter指向 key1:4:1, 则FindPrevUserEntry的工作为：

1. 保存key1:4:1,

   ```c++
   Slice raw_value = iter_->value();
             if (saved_value_.capacity() > raw_value.size() + 1048576) {
               std::string empty;
               swap(empty, saved_value_);
             }
             SaveKey(ExtractUserKey(iter_->key()), &saved_key_);
             saved_value_.assign(raw_value.data(), raw_value.size());
   }
   ```

2. iter指向key1:5:0，当前是删除节点，清空saved_key和saved_value:

   ```c++
   if (value_type == kTypeDeletion) {	// 删除节点，清空saved_key和saved_value
             saved_key_.clear();
             ClearSavedValue();
           } 
   ```

3. iter指向key1:6:1, 保存.

4. iter指向key0:7:1, 遇到不同user_key且不是删除几点,跳出：

   ```c++
     if ((value_type != kTypeDeletion) &&		// 不是删除节点
               user_comparator_->Compare(ikey.user_key, saved_key_) < 0) {		// 遇到了新user_key
             // We encountered a non-deleted value in entries for previous keys,
             break;
     }
   ```

**这样就从后往前遍历到当前user_key的最新版本。**

## 7. Prev

```c++
void DBIter::Prev() {
  assert(valid_);

  if (direction_ == kForward) {  // Switch directions?
    // iter_ is pointing at the current entry.  Scan backwards until
    // the key changes so we can use the normal reverse scanning code.
    assert(iter_->Valid());  // Otherwise valid_ would have been false
    SaveKey(ExtractUserKey(iter_->key()), &saved_key_);
    while (true) {
      iter_->Prev();
      if (!iter_->Valid()) {
        valid_ = false;
        saved_key_.clear();
        ClearSavedValue();
        return;
      }
      if (user_comparator_->Compare(ExtractUserKey(iter_->key()), saved_key_) <
          0) {
        break;
      }
    }
    direction_ = kReverse;
  }

  FindPrevUserEntry();
}

```

**注释说得很明白，首先从后往前遍历到和当前user_key不同的节点，设这个节点为节点k。然后通过FindPrevUserEntry找到节点k的最新版本。**

## 8. key & value

```c++
  Slice key() const override {
    assert(valid_);
    return (direction_ == kForward) ? ExtractUserKey(iter_->key()) : saved_key_;
  }
  Slice value() const override {
    assert(valid_);
    return (direction_ == kForward) ? iter_->value() : saved_value_;
  }
```

这个很简单了，不过这里也终于解释了saved_key_，saved_value_已经direction变量的作用。因为在反向遍历时，会出现这种的情况：

![](https://pic.downk.cc/item/5f8441351cd1bbb86b041853.png)

反向遍历时，需要的key应该时key1:6:1, 但是iter_已经指向了key0:7:1。 正向遍历则不存在这个问题，所以DBIter使用了三个额外的变量(direction,saved_key,saved_value)来区分正向遍历和反向遍历。

## 9. ParseKey

看完前面的几个函数分析，其实已经能够了解DBIter的所有操作了，但是这里还要提一下ParseKey这个函数：

```c++
inline bool DBIter::ParseKey(ParsedInternalKey* ikey) {
  Slice k = iter_->key();

  size_t bytes_read = k.size() + iter_->value().size();
  while (bytes_until_read_sampling_ < bytes_read) {
    bytes_until_read_sampling_ += RandomCompactionPeriod();
    db_->RecordReadSample(k);
  }
  assert(bytes_until_read_sampling_ >= bytes_read);
  bytes_until_read_sampling_ -= bytes_read;

  if (!ParseInternalKey(k, ikey)) {
    status_ = Status::Corruption("corrupted internal key in DBIter");
    return false;
  } else {
    return true;
  }
}
```

ParseKey的主要职责从iter_->key()返回的string中解析出一个InternalKey出来，封装在ikey中。这部分很简单，我想说的是:

```c++
  size_t bytes_read = k.size() + iter_->value().size();
  while (bytes_until_read_sampling_ < bytes_read) {
    bytes_until_read_sampling_ += RandomCompactionPeriod();
    db_->RecordReadSample(k);
  }
  assert(bytes_until_read_sampling_ >= bytes_read);
  bytes_until_read_sampling_ -= bytes_read;
```

这部分代码的意义是在 DBIter的scan过程中，对整个sstable进行compaction。当然这个compaction肯定有个执行间隔，这个间隔就是由bytes_until_read_sampling_来控制。它由一个随机数初始化：

```c++
// Approximate gap in bytes between samples of data read during iteration.
static const int kReadBytesPeriod = 1048576;

  // Picks the number of bytes that can be read until a compaction is scheduled.
  size_t RandomCompactionPeriod() {
    return rnd_.Uniform(2 * config::kReadBytesPeriod);
  }
```

每次增加的间隔也是也是一个基于均匀分布的随机数。

再看看db_->RecordReadSample(k);

```c++
void DBImpl::RecordReadSample(Slice key) {
  MutexLock l(&mutex_);
  if (versions_->current()->RecordReadSample(key)) {
    MaybeScheduleCompaction();
  }
}
```

MaybeScheduleCompaction是Compaction过程，详情参考Compaction章节，这里不赘述。重点看RecordReadSample_：

```c++
bool Version::RecordReadSample(Slice internal_key) {
  ParsedInternalKey ikey;
  if (!ParseInternalKey(internal_key, &ikey)) {
    return false;
  }

  struct State {
    GetStats stats;  // Holds first matching file
    int matches;

    static bool Match(void* arg, int level, FileMetaData* f) {
      State* state = reinterpret_cast<State*>(arg);
      state->matches++;
      if (state->matches == 1) {
        // Remember first match.
        state->stats.seek_file = f;
        state->stats.seek_file_level = level;
      }
      // We can stop iterating once we have a second match.
      return state->matches < 2;
    }
  };

  State state;
  state.matches = 0;
  ForEachOverlapping(ikey.user_key, internal_key, &state, &State::Match);

  // Must have at least two matches since we want to merge across
  // files. But what if we have a single file that contains many
  // overwrites and deletions?  Should we have another mechanism for
  // finding such files?
  if (state.matches >= 2) {
    // 1MB cost is about 1 seek (see comment in Builder::Apply).
    return UpdateStats(state.stats);
  }
  return false;
}
```

这里是不是看着比较熟悉？整体框架和基于Seek的Compaction完全相同。只不过Match函数采用了matches次数来记录seek_file指针。

一旦匹配到两次以上，就执行UpdateStats，在UpdateStats内部会对seek_file所指向的file的allowed_seek--。 当allowed_seek减到<=0时，就可以设置用来触发seek compaction的标志了。

## 总结

DBIter是对InternelIter的封装，两者针对的粒度不同。InternelIter以InternelKey为粒度，一个InternelKey由(user_key,sequence,type)组成，只要这三者有一个不同，InternelIter就认为它们是不同key。但对用户来说，只关心user_key是否相同，所以诞生了DBIter，用来在具有相同user_key的internelkey中，找到最新版本的那个并返回给用户。

到这里，leveldb中的所有iterator都已介绍完毕。