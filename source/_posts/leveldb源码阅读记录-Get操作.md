---
title: leveldb源码阅读记录-Get操作
categories: leveldb
abbrlink: 6349fde1
date: 2020-10-12 18:59:42
tags:
---

上文我们说了leveldb的Put操作。简单来说就是先向log写入一条记录，用于保证本条记录的持久性，然后向memtable插入本条记录。当然这个过程还可能牵涉到compaction，但从宏观上就是这么简单的两步。

今天我们再来谈谈leveldb的Get操作。

<!--more-->

## 1. DBImpl::Get

先看下Get的函数签名：

```c++
Status DBImpl::Get(const ReadOptions& options, const Slice& key,
                   std::string* value) ;
```

第一个是读取操作的options，紧跟一个本次Get的key，将get到的value保存在最后一个参数中。

简单说一下第一个参数:

### 1.  ReadOptions

```c++
// Options that control read operations
struct LEVELDB_EXPORT ReadOptions {
  ReadOptions() = default;

   // 如果true，所有从底层读取回来的数据都需要进行校验
  // If true, all data read from underlying storage will be
  // verified against corresponding checksums.
  bool verify_checksums = false;

   // 本次read的record是否需要缓存到内存？
  // Should the data read for this iteration be cached in memory?
  // Callers may wish to set this field to false for bulk scans.
  bool fill_cache = true;

    // 如果提供快照(non-null),则从快照种读。如果不提供(null)，则从implicit的快照种读
  // If "snapshot" is non-null, read as of the supplied snapshot
  // (which must belong to the DB that is being read and which must
  // not have been released).  If "snapshot" is null, use an implicit
  // snapshot of the state at the beginning of this read operation.
  const Snapshot* snapshot = nullptr;
};
```

注释中说得很明确了，不再赘述。

### 2. Get函数

吸纳再正式看看Get函数：

```c++
Status DBImpl::Get(const ReadOptions& options, const Slice& key,
                   std::string* value) {
  Status s;
  MutexLock l(&mutex_);
  SequenceNumber snapshot;
    // 确定是从哪个snapshot种读取
  if (options.snapshot != nullptr) {	// 提供了snapshot，则从提供的snapshot中读
    snapshot =
        static_cast<const SnapshotImpl*>(options.snapshot)->sequence_number();
  } else {			// 否则从当前最新版本读
    snapshot = versions_->LastSequence();
  }

  MemTable* mem = mem_;
  MemTable* imm = imm_;
  Version* current = versions_->current();
    // 增加引用计数
  mem->Ref();
  if (imm != nullptr) imm->Ref();
  current->Ref();

  bool have_stat_update = false;
  Version::GetStats stats;

  // Unlock while reading from files and memtables
  {
    mutex_.Unlock();
    // First look in the memtable, then in the immutable memtable (if any).
     // 将user key封装到LookupKey中，LookupKey是对整个leveldb所能用的key的封装，可向外提供memtable使用的key，sstable使用的key，user的key
    LookupKey lkey(key, snapshot);
    if (mem->Get(lkey, value, &s)) {	// 先向memtable中查询
      // Done
    } else if (imm != nullptr && imm->Get(lkey, value, &s)) {	// 再向imm查询
      // Done
    } else {		// 最后到外存的sstables中查询
      s = current->Get(options, lkey, value, &stats);
      have_stat_update = true;
    }
    mutex_.Lock();
  }

  if (have_stat_update && current->UpdateStats(stats)) {	// 更新状态，可能会触发基于seek的compaction
    MaybeScheduleCompaction();
  }
    // 减少引用计数
  mem->Unref();
  if (imm != nullptr) imm->Unref();
  current->Unref();
  return s;
}

```

整体代码逻辑也非常清晰。 先查memtable，再查imm，再查外存的sstables。

memtable和imm的查询都是基于SkipList的Iterator来查询。在memtable章节中已经说过，这里不再赘述。主要说一下

```c++
 s = current->Get(options, lkey, value, &stats);
```

```c++
Status Version::Get(const ReadOptions& options, const LookupKey& k,
                    std::string* value, GetStats* stats) {
  stats->seek_file = nullptr;
  stats->seek_file_level = -1;

 ...
     
// 核心逻辑在这里
  ForEachOverlapping(state.saver.user_key, state.ikey, &state, &State::Match);

  return state.found ? state.s : Status::NotFound(Slice());
}

```

```c++
void Version::ForEachOverlapping(Slice user_key, Slice internal_key, void* arg,
                                 bool (*func)(void*, int, FileMetaData*)) {
  const Comparator* ucmp = vset_->icmp_.user_comparator();

    // 先在level0中search
  // Search level-0 in order from newest to oldest.
  std::vector<FileMetaData*> tmp;
  tmp.reserve(files_[0].size());
  for (uint32_t i = 0; i < files_[0].size(); i++) {
    FileMetaData* f = files_[0][i];
    if (ucmp->Compare(user_key, f->smallest.user_key()) >= 0 &&
        ucmp->Compare(user_key, f->largest.user_key()) <= 0) { // 找到level0的sstable，这些sstable的range 都 包含了 user_key.
      tmp.push_back(f);
    }
  }
  if (!tmp.empty()) {
    std::sort(tmp.begin(), tmp.end(), NewestFirst);	// sstables按照sequence number的由大到小排序（因为seq越大，代表数据越新）
    for (uint32_t i = 0; i < tmp.size(); i++) {
      if (!(*func)(arg, 0, tmp[i])) {	// 调用Match函数
        return;
      }
    }
  }

   // 在其他level中查询
  // Search other levels.
  for (int level = 1; level < config::kNumLevels; level++) {
    size_t num_files = files_[level].size();
    if (num_files == 0) continue;

    // Binary search to find earliest index whose largest key >= internal_key.
    uint32_t index = FindFile(vset_->icmp_, files_[level], internal_key);
    if (index < num_files) {
      FileMetaData* f = files_[level][index];
      if (ucmp->Compare(user_key, f->smallest.user_key()) < 0) {	// 本层中的最小key都比 internal key大，说明本层没有合适的sstable
        // All of "f" is past any data for user_key
      } else {
        if (!(*func)(arg, level, f)) {	// 调用match
          return;
        }
      }
    }
  }
}
```

这里的函数逻辑也比较简单，总结起来就一句话，先看level0是否能找到合适的sstable，如果找不到，再依次逐层往下找。

到这里我们都是用的sstable的元数据 FileMeta 再寻找， 那在哪儿具体落实到某个实际的sstable呢？ 那就是Match函数：

```c++
    static bool Match(void* arg, int level, FileMetaData* f) {
   
	  ...
      // 从cache中获取
      state->s = state->vset->table_cache_->Get(*state->options, f->number,
                                                f->file_size, state->ikey,
                                                &state->saver, SaveValue);
      if (!state->s.ok()) {
        state->found = true;
        return false;
      }
      switch (state->saver.state) {
        case kNotFound:
          return true;  // Keep searching in other files
        case kFound:
          state->found = true;
          return false;
        case kDeleted:
          return false;
        case kCorrupt:
          state->s =
              Status::Corruption("corrupted key for ", state->saver.user_key);
          state->found = true;
          return false;
      }

      // Not reached. Added to avoid false compilation warnings of
      // "control reaches end of non-void function".
      return false;
    }
```

实际的sstable来自cache（前提是开启了cache，否则依然是从文件系统中的文件中获取）。

### 3. 总结：

总结流程图如下：

![](https://pic.downk.cc/item/5f82c4201cd1bbb86b405985.png)

首先去mem中读，读不到，去imm中读，读不到，去底层sstable中读，因为sstable可能被cache到内存，所以可以去cache中读，如果系统没有配置cache，或者cache中没有cache到指定sstable，则到文件系统中读。

关于RangeQuery操作，其实就是对Iterator的操作，而leveldb的Iterator已经在DBIter介绍，所以不再赘述。