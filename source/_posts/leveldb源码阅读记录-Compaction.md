---
title: leveldb源码阅读记录-Compaction
categories: leveldb
abbrlink: 1ba074b9
date: 2020-10-12 18:00:00
tags:
---

截至到上文，我们已经将levedb中几个重要的组件都分析了，包括log、manifest、memtable和sstable。今天将介绍leveldb中最重要的内部操作--Compaction。

<!--more-->

# MemTable 到 SStable

在leveldb中，compaction共有两种，分别叫 minor compaction 和major compaction。

- minor compaction，将immtable dump到SStable
- major compaction，level之间的SSTable compaction。

这里先来分析minor compaction。

我们主要关注以下问题：

1. minor compaction是如何进行的？
2. minor compaction何时会进行？

# 1. minor compaction如何进行？

compaction的入口是 `   DBImpl::MaybeScheduleCompaction() `

```c++
void DBImpl::MaybeScheduleCompaction() {
  mutex_.AssertHeld();
  if (background_compaction_scheduled_) {
    // Already scheduled
  } else if (shutting_down_.load(std::memory_order_acquire)) {
    // DB is being deleted; no more background compactions
  } else if (!bg_error_.ok()) {
    // Already got an error; no more changes
  } else if (imm_ == nullptr && manual_compaction_ == nullptr &&
             !versions_->NeedsCompaction()) {
      // 递归结束点，防止无限递归
    // No work to be done
  } else {
    background_compaction_scheduled_ = true;
    env_->Schedule(&DBImpl::BGWork, this);
  }
}
```

在 env_->Schedule(&DBImpl::BGWork, this);中，将BGWork放入线程池，由子线程来做:

```c++
void DBImpl::BGWork(void* db) {
  reinterpret_cast<DBImpl*>(db)->BackgroundCall();
}
```

```c++
void DBImpl::BackgroundCall() {
  MutexLock l(&mutex_);
  assert(background_compaction_scheduled_);
  if (shutting_down_.load(std::memory_order_acquire)) {
    // No more background work when shutting down.
  } else if (!bg_error_.ok()) {
    // No more background work after a background error.
  } else {
    BackgroundCompaction();
  }

  background_compaction_scheduled_ = false;

  // Previous compaction may have produced too many files in a level,
  // so reschedule another compaction if needed.
    // 递归调用compaction，因为有可能这次compaction产生了过多的sst
  MaybeScheduleCompaction();
  background_work_finished_signal_.SignalAll();
}
```

## 1. 调用流程图

这里的调用链比较清晰：

<img src="https://pic.downk.cc/item/5f84430d1cd1bbb86b056b74.png" style="zoom: 33%;" />

<img src="https://pic.downk.cc/item/5f84432b1cd1bbb86b0582d0.png" style="zoom:33%;" />

需要注意的是，DBImpl::MaybeScheduleCompaction 是一个递归调用，递归结束的地方在：

```c++
} else if (imm_ == nullptr && manual_compaction_ == nullptr &&
             !versions_->NeedsCompaction()) {
    // No work to be done
  }
```

也就说结束的条件是：

1. 当前immemtable为null

2. 非手动compaction

3. versionset判定为不需要compaction

   ```c++
     // Returns true iff some level needs a compaction.
     bool NeedsCompaction() const {
       Version* v = current_;
       return (v->compaction_score_ >= 1) || (v->file_to_compact_ != nullptr);
     }
   ```

   我们曾在分析VersionEdit，VersionSet时，提到过compaction_score是如何计算的，这里提一下它的入口在 VersionSet::Finalize 。后面再做详细分析。

## 2. Minor Compaction执行细节

ok，说完宏观的调用链，现在来详细分一下leveldb是如何左minor compaction的，核心函数在：==BackgroundCompaction();==

```c++
void DBImpl::BackgroundCompaction() {
  mutex_.AssertHeld();

  if (imm_ != nullptr) {	// minor compaction的触发点
    CompactMemTable();
    return;
  }
  xxx
}
```

```c++
void DBImpl::CompactMemTable() {
  mutex_.AssertHeld();
  assert(imm_ != nullptr);

  // Save the contents of the memtable as a new Table
  VersionEdit edit;
  Version* base = versions_->current();
  base->Ref();
   // 将 数据写入到第0层（实际上不一定是第0层)
  Status s = WriteLevel0Table(imm_, &edit, base);
  base->Unref();

  if (s.ok() && shutting_down_.load(std::memory_order_acquire)) {
    s = Status::IOError("Deleting DB during memtable compaction");
  }

  // Replace immutable memtable with the generated Table
  if (s.ok()) {
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);  // Earlier logs no longer needed
      // 应用生成的一个VersionEdit到当前VersionSet
    s = versions_->LogAndApply(&edit, &mutex_);
  }

  if (s.ok()) {
    // Commit to the new state
     // 减少计数，引用计数归0时会delete当前immemtable
    imm_->Unref();
    imm_ = nullptr;
    has_imm_.store(false, std::memory_order_release);
    RemoveObsoleteFiles();
  } else {
    RecordBackgroundError(s);
  }
}
```

我们继续看==WriteLevel0Table==

```c++
Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
  mutex_.AssertHeld();
  const uint64_t start_micros = env_->NowMicros();
  FileMetaData meta;
  meta.number = versions_->NewFileNumber();
  pending_outputs_.insert(meta.number);
  Iterator* iter = mem->NewIterator();
  Log(options_.info_log, "Level-0 table #%llu: started",
      (unsigned long long)meta.number);

  Status s;
  {
    mutex_.Unlock();
      //!! 1. 将memtable dump 到SSTable中
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
    mutex_.Lock();
  }

  Log(options_.info_log, "Level-0 table #%llu: %lld bytes %s",
      (unsigned long long)meta.number, (unsigned long long)meta.file_size,
      s.ToString().c_str());
  delete iter;
  pending_outputs_.erase(meta.number);

  // Note that if file_size is zero, the file has been deleted and
  // should not be added to the manifest.
  int level = 0;
  if (s.ok() && meta.file_size > 0) {
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    if (base != nullptr) {
        // 2.SSTable 应该写入到哪个level？
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
      // 3. 生成VersionEdit，给后序Manifest做记录
    edit->AddFile(level, meta.number, meta.file_size, meta.smallest,
                  meta.largest);
  }

    // 4. 保存本次compaction所在level的 compaction状态
  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros;
  stats.bytes_written = meta.file_size;
  stats_[level].Add(stats);
  return s;
}
```

### 1 WriteLevel0Table 流程图

<img src="https://pic.downk.cc/item/5f84433d1cd1bbb86b058e5b.png" style="zoom:50%;" />

#### 3个函数的分析

现在来分别分析这3个函数：

#### 1. BuildTable (建立ssttable并持久化)

下面这些 .ldb的文件就是sstable，可以看到它们都是一些由数字组成的文件名，这些数字是哪里来的？我们可以从源码中获得答案：

![image-20200925153434477](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200925153434477.png)

```c++
Status BuildTable(const std::string& dbname, Env* env, const Options& options,
                  TableCache* table_cache, Iterator* iter, FileMetaData* meta) {
  Status s;
  meta->file_size = 0;
  iter->SeekToFirst();

   // meta number就是上图的数字来源， meta->number的赋值语句在DBImpl::WriteLevel0Table中的 `meta.number = versions_->NewFileNumber();`
    // 所以该数字由VersionSet分配
  std::string fname = TableFileName(dbname, meta->number);
  if (iter->Valid()) {
    WritableFile* file;
    s = env->NewWritableFile(fname, &file);
    if (!s.ok()) {
      return s;
    }
      // 新建sstable
    TableBuilder* builder = new TableBuilder(options, file);
      // 因为跳表是有序的，所以第一个key肯定是最小的
    meta->smallest.DecodeFrom(iter->key());
    Slice key;
    for (; iter->Valid(); iter->Next()) {
      key = iter->key();
       // 向sstable中添加key value。
      builder->Add(key, iter->value());
    }
    if (!key.empty()) {
        // 同理，最后一个key是最大的
      meta->largest.DecodeFrom(key);
    }

    // Finish and check for builder errors
      // 写入sstable中的其他块，index block, meta block ,meta index block footer等
    s = builder->Finish();
    if (s.ok()) {
      meta->file_size = builder->FileSize();
      assert(meta->file_size > 0);
    }
    delete builder;

    // Finish and check for file errors
    if (s.ok()) {
        // 写入到硬件
      s = file->Sync();
    }
    if (s.ok()) {
      s = file->Close();
    }
    delete file;
    file = nullptr;

   xxx

  if (s.ok() && meta->file_size > 0) {
    // Keep it
  } else {
    env->RemoveFile(fname);
  }
  return s;
}
```

工作：

1. 新建SStable文件
2. 根据memtable提供的Iterator，向SStable中添加所有数据
3. 将SSTable写入到硬件设备中。

#### 2. Version::PickLevelForMemTableOutput

PickLevelForMemTableOutput决定新生成的sstable所在的level，原则上，从memtable dump出来sstable应该首先放到level0， 但是如果总是放到level 0，后序的compaction会耗费过多的io吞吐量，所以这个函数的意思是，尽量将新生成的sstable往更深的level放，但是又不能放的太深，因为如果这个sstable的访问频率较高，过深的level意味着读性能的降低。所以往下push得有个读。

leveldb定义了kMaxMemCompactLevel这个参数来限制新生成的sstable能够下推的层次：

```c++
// Maximum level to which a new compacted memtable is pushed if it
// does not create overlap.  We try to push to level 2 to avoid the
// relatively expensive level 0=>1 compactions and to avoid some
// expensive manifest file operations.  We do not push all the way to
// the largest level since that can generate a lot of wasted disk
// space if the same key space is being repeatedly overwritten.
static const int kMaxMemCompactLevel = 2;
```

所以，默认最高只能到level2.

现在来看看源码：

```c++
int Version::PickLevelForMemTableOutput(const Slice& smallest_user_key,
                                        const Slice& largest_user_key) {
  int level = 0;
    // 如果与level 0有重叠，直接return 0
  if (!OverlapInLevel(0, &smallest_user_key, &largest_user_key)) {
    // Push to next level if there is no overlap in next level,
    // and the #bytes overlapping in the level after that are limited.
    InternalKey start(smallest_user_key, kMaxSequenceNumber, kValueTypeForSeek);
    InternalKey limit(largest_user_key, 0, static_cast<ValueType>(0));
    std::vector<FileMetaData*> overlaps;
    while (level < config::kMaxMemCompactLevel) {	// 最高到kMaxMemCompactLevel
       // 与level+1有重叠
      if (OverlapInLevel(level + 1, &smallest_user_key, &largest_user_key)) {
        break;
      }
      if (level + 2 < config::kNumLevels) {
         // 与祖父level的重叠size过大，则直接break
        // Check that file does not overlap too many grandparent bytes.
        GetOverlappingInputs(level + 2, &start, &limit, &overlaps);
        const int64_t sum = TotalFileSize(overlaps);
        if (sum > MaxGrandParentOverlapBytes(vset_->options_)) {
          break;
        }
      }
      level++;
    }
  }
  return level;
}
```

引用一张流程图：

![img](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/PickLevelForMemTableOutput.png)



#### 3. edit->Addfile （记录元sstable所在level等元数据）

最后就是edit->Addfile。将生成的sstable的元数据（filemeta）加入到versionedit中。

```c++
  // Add the specified file at the specified number.
  // REQUIRES: This version has not been saved (see VersionSet::SaveTo)
  // REQUIRES: "smallest" and "largest" are smallest and largest keys in file
  void AddFile(int level, uint64_t file, uint64_t file_size,
               const InternalKey& smallest, const InternalKey& largest) {
    FileMetaData f;
    f.number = file;
    f.file_size = file_size;
    f.smallest = smallest;
    f.largest = largest;
    new_files_.push_back(std::make_pair(level, f));
  }
```

从edit->AddFile可知，一个SSTable对应有一个FileMeta存放在edit中，edit最终会存放在manifest，同时edit最终会演变成version，version又会加入到versioneset中。

### 2. LogAndApply

LogAndApply前面已经分析过了。 之前我们将所有filemeta存放在一个versionedit中，通过这个LogAndApply即可将versionedit应用到当前versionset中，并持久化到manifest。

## 2. 何时Tigger Compaction？

让我们回到MaybeScheduleCompaction

```c++
void DBImpl::MaybeScheduleCompaction() {
  mutex_.AssertHeld();
  if (background_compaction_scheduled_) {
    // Already scheduled
  } else if (shutting_down_.load(std::memory_order_acquire)) {
    // DB is being deleted; no more background compactions
  } else if (!bg_error_.ok()) {
    // Already got an error; no more changes
  } else if (imm_ == nullptr && manual_compaction_ == nullptr &&
             !versions_->NeedsCompaction()) {
      // 递归结束点，防止无限递归
    // No work to be done
  } else {
    background_compaction_scheduled_ = true;
    env_->Schedule(&DBImpl::BGWork, this);
  }
}
```

可以看到，正常情况下只要满足一下3种条件中任何一个都会触发一次compaction：

- imm != null, memtable已经转化为immtable，需要及时dump到外存中。
- manual_compaction， 手动compaction
- versions->NeedCompaction()

前两种都好说，重点看一下第3种：

```c++
  // Returns true iff some level needs a compaction.
  bool NeedsCompaction() const {
    Version* v = current_;
    return (v->compaction_score_ >= 1) || (v->file_to_compact_ != nullptr);
  }
```

这里又分为了两种情况，根据compaction_score和根据 filte_compact。先说第一种：`v->compaction_score_ `

### 1. compaction_score_  & Finalize函数

==Finalize函数==

```c++
void VersionSet::Finalize(Version* v) {
  // Precomputed best level for next compaction
  int best_level = -1;
  double best_score = -1;
						// 最高level
  for (int level = 0; level < config::kNumLevels - 1; level++) {
    double score;
    if (level == 0) {
      // We treat level-0 specially by bounding the number of files
      // instead of number of bytes for two reasons:
      //
      // (1) With larger write-buffer sizes, it is nice not to do too
      // many level-0 compactions.
      //
      // (2) The files in level-0 are merged on every read and
      // therefore we wish to avoid too many files when the individual
      // file size is small (perhaps because of a small write-buffer
      // setting, or very high compression ratios, or lots of
      // overwrites/deletions).
      score = v->files_[level].size() /
              static_cast<double>(config::kL0_CompactionTrigger);
    } else {
      // Compute the ratio of current size to size limit.
      const uint64_t level_bytes = TotalFileSize(v->files_[level]);
      score =
          static_cast<double>(level_bytes) / MaxBytesForLevel(options_, level);
    }

    if (score > best_score) {
      best_level = level;
      best_score = score;
    }
  }

  v->compaction_level_ = best_level;
  v->compaction_score_ = best_score;
}
```

在leveldb的设计中，level0和其余level的compaction设计是不同的，level0基于文件数量，而其余层基于文件的总大小。

#### level 0

level0为什么要要这样设计？根据注释：

1. 如果有更大的写buffer, 应该尽量避免多个level 0的compactions。
2. 每次读，都会level0的file merge，应该如果level0的文件数量过多。（个人理解：因为读会衰减？）

> 引用博文：
>
> 注释说的很明白，level 0的文件之间，key可能是交叉重叠的，因此不希望level 0的文件数特别多。我们考虑write buffer 比较小的时候，如果使用size来限制，那么level 0的文件数可能太多。
>
> 另一个方面，如果write buffer过大，使用固定大小的size 来限制level 0的话，可能算出来的level 0的文件数又太少，触发 level 0 compaction的情况发生的又太频繁。

所以，对于level0，其默认设计的文件数量不超过4:

```c++
// Level-0 compaction is started when we hit this many files.
static const int kL0_CompactionTrigger = 4;
```

#### 其他level

其他level则是根据当前文件大小与size limit的比值。

```c++
 else {
      // Compute the ratio of current size to size limit.
      const uint64_t level_bytes = TotalFileSize(v->files_[level]);
      score =
          static_cast<double>(level_bytes) / MaxBytesForLevel(options_, level);
    }
```



```c++
static double MaxBytesForLevel(const Options* options, int level) {
  // Note: the result for level zero is not really used since we set
  // the level-0 compaction threshold based on number of files.

  // Result for both level-0 and level-1
  double result = 10. * 1048576.0;	// 默认是10M
  while (level > 1) {
    result *= 10;
    level--;
  }
  return result;
}
```

这段代码说明，从level1开始，每相邻两层的level总大小，相差10.  level1=10M, level2=100M，以此类推。

同时这里也指明了leveldb的最高level为多少：

```c++
for (int level = 0; level < config::kNumLevels - 1; level++) {
    double score;
    if (level == 0) {
    xxx
```

```c++
static const int kNumLevels = 7;
```

所以最高到level6, 则各层大小分布为：

```
level 1               10M 
level 2              100M
level 3             1000M
level 4            10000M
level 5           100000M
level 6          1000000M
```

#### 选择得分最高的来做compaction

```c++
   if (score > best_score) {
      best_level = level;
      best_score = score;
    }
```

得分越高，越想compaction。

#### 总结：

所以根据compaction_score_ 来看，当level0的文件过多，或者其余level的总file size过大时，会触发compacton。

### 2. file_to_compact_ & Seek Compaction

除了上述情况外，leveldb还有用了基于seek的compactoin。

除了level 0以外，任何一个level的文件内部是有序的，文件之间也是有序的。但是level（n）和level （n＋1）中的两个文件的key可能存在交叉。正是因为这种交叉，查找某个key值的时候， level（n） 的查找无功而返，而不得不restart to level(n＋1)。

我们考虑寻找某一个key，如果找了曾经查找了level (n) ,但是没找到，然后去level (n+1)查找，结果找到了，那么对level (n)的某个文件而言，该文件就意味着有一次 未命中。

我们可以很容易想到，如果查找了多次，某个文件不得不查找，却总也找不到，总是去高一级的level，才能找到。这说明该层级的文件和上一级的文件，key的范围重叠的很严重，这是不合理的，会导致效率的下降。因此，需要对该level 发起一次Major compaction，减少 level 和level ＋ 1的重叠情况。

这就是所谓的 Seek Compaction。

我个人的理解是，查找一个key，根据manifest判定这个key可能在某个sstable中（manifest中存放了sstable的smallest和largest key）,但是实际上并不在，所以总是在更深层中去找。那查找本层的sstable就是对io的浪费，而且也说明了本层和更深层的key有比较严重的相互重叠。举个例子，如下图：

<img src="https://pic.downk.cc/item/5f844aeb1cd1bbb86b0a9343.png" style="zoom:33%;" />

现在查找6， 对于level1的sstable来说，key的range在[1,9], 所以会查找这个sstable，显然6不在其中，于是向下层中找，level2的这个sstable的key range为[2,8]，在这里找到了。 这样level1的io就是浪费掉的， level1和level2的key overlap也比较严重，长此以往浪费io，所以需要compaction。

seek compaction在filemeta中用 **allowed_seeks** 来控制。

```c++
struct FileMetaData {
  FileMetaData() : refs(0), allowed_seeks(1 << 30), file_size(0) {}

  int refs;
  int allowed_seeks;  // !!!Seeks allowed until compaction
  uint64_t number;
  uint64_t file_size;    // File size in bytes
  InternalKey smallest;  // Smallest internal key served by table
  InternalKey largest;   // Largest internal key served by table
};

```

==VersionSet::Builder::Apply==对其初始化：

```c++
  // Apply all of the edits in *edit to the current state.
  void Apply(VersionEdit* edit) {
    // Update compaction pointers
    for (size_t i = 0; i < edit->compact_pointers_.size(); i++) {
      const int level = edit->compact_pointers_[i].first;
      vset_->compact_pointer_[level] =
          edit->compact_pointers_[i].second.Encode().ToString();
    }

    // Delete files
    for (const auto& deleted_file_set_kvp : edit->deleted_files_) {
      const int level = deleted_file_set_kvp.first;
      const uint64_t number = deleted_file_set_kvp.second;
      levels_[level].deleted_files.insert(number);
    }

    // Add new files
    for (size_t i = 0; i < edit->new_files_.size(); i++) {
      const int level = edit->new_files_[i].first;
      FileMetaData* f = new FileMetaData(edit->new_files_[i].second);
      f->refs = 1;

      // We arrange to automatically compact this file after
      // a certain number of seeks.  Let's assume:
      //   (1) One seek costs 10ms
      //   (2) Writing or reading 1MB costs 10ms (100MB/s)
      //   (3) A compaction of 1MB does 25MB of IO:
      //         1MB read from this level
      //         10-12MB read from next level (boundaries may be misaligned)
      //         10-12MB written to next level
      // This implies that 25 seeks cost the same as the compaction
      // of 1MB of data.  I.e., one seek costs approximately the
      // same as the compaction of 40KB of data.  We are a little
      // conservative and allow approximately one seek for every 16KB
      // of data before triggering a compaction.
      f->allowed_seeks = static_cast<int>((f->file_size / 16384U));
      if (f->allowed_seeks < 100) f->allowed_seeks = 100;

      levels_[level].deleted_files.erase(f->number);
      levels_[level].added_files->insert(f);
    }
  }
```

这里说的是， 根据估算，大概每25次seek的time cost = 1次compaction的。保守估计，1次seek相当于compaction16kb的数据。 所以==当seek的总耗时约等于一次compaction的耗时时，就触发一次compaction==。则允许seek的次数为：

```c++
f->allowed_seeks = static_cast<int>((f->file_size / 16384U)); // file_size/16KB
```

# 2. 如何确定Compaction的输入源

结合前面两种compaction来看，触发compaction的时机：

1. size compaction :文件过多或文件过大
2. seek compaction: seek次数过多。

现在回到==DBImpl::BackgroundCompaction:==

```c++
void DBImpl::BackgroundCompaction() {
  mutex_.AssertHeld();

  if (imm_ != nullptr) {
    CompactMemTable();
    return;
  }

  Compaction* c;
  bool is_manual = (manual_compaction_ != nullptr);
  InternalKey manual_end;
  if (is_manual) {
    ManualCompaction* m = manual_compaction_;
    c = versions_->CompactRange(m->level, m->begin, m->end);
    m->done = (c == nullptr);
    if (c != nullptr) {
      manual_end = c->input(0, c->num_input_files(0) - 1)->largest;
    }
    Log(options_.info_log,
        "Manual compaction at level-%d from %s .. %s; will stop at %s\n",
        m->level, (m->begin ? m->begin->DebugString().c_str() : "(begin)"),
        (m->end ? m->end->DebugString().c_str() : "(end)"),
        (m->done ? "(end)" : manual_end.DebugString().c_str()));
  } else {		// 现在只看非manual的情况
    c = versions_->PickCompaction();
  }
   ...
}

```

## 1. PickCompaction

### 流程图

PickCompaction流程图：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb源码阅读-copy-PickCompaction.png" style="zoom: 33%;" />

转到==PickCompaction==函数：

```c++
// Pick level and inputs for a new compaction.
// Returns nullptr if there is no compaction to be done.
// Otherwise returns a pointer to a heap-allocated object that
// describes the compaction.  Caller should delete the result.
Compaction* PickCompaction();

Compaction* VersionSet::PickCompaction() {
  Compaction* c;
  int level;

   // 优先考虑 size_compaction, 再考虑seek_compaction.
  // We prefer compactions triggered by too much data in a level over
  // the compactions triggered by seeks.
  const bool size_compaction = (current_->compaction_score_ >= 1);
  const bool seek_compaction = (current_->file_to_compact_ != nullptr);
  if (size_compaction) {
    level = current_->compaction_level_;
    assert(level >= 0);
    assert(level + 1 < config::kNumLevels);
    c = new Compaction(options_, level);

    // Pick the first file that comes after compact_pointer_[level]
    for (size_t i = 0; i < current_->files_[level].size(); i++) {
      FileMetaData* f = current_->files_[level][i];
      if (compact_pointer_[level].empty() ||
          icmp_.Compare(f->largest.Encode(), compact_pointer_[level]) > 0) {
        c->inputs_[0].push_back(f);
        break;
      }
    }
    if (c->inputs_[0].empty()) {
      // Wrap-around to the beginning of the key space
      c->inputs_[0].push_back(current_->files_[level][0]);
    }
  } else if (seek_compaction) {
    level = current_->file_to_compact_level_;
    c = new Compaction(options_, level);
    c->inputs_[0].push_back(current_->file_to_compact_);
  } else {
    return nullptr;
  }

  c->input_version_ = current_;
  c->input_version_->Ref();

  // Files in level 0 may overlap each other, so pick up all overlapping ones
  if (level == 0) {
    InternalKey smallest, largest;
    GetRange(c->inputs_[0], &smallest, &largest);
    // Note that the next call will discard the file we placed in
    // c->inputs_[0] earlier and replace it with an overlapping set
    // which will include the picked file.
    current_->GetOverlappingInputs(0, &smallest, &largest, &c->inputs_[0]);
    assert(!c->inputs_[0].empty());
  }

  SetupOtherInputs(c);

  return c;
}
```

根据函数签名注释，PickCompaction的作用是，找到level以及level对应的需要compaction的文件。现在来拆开代码看：

优先考虑 size_compaction, 再考虑seek_compaction.

```c++
// We prefer compactions triggered by too much data in a level over
// the compactions triggered by seeks.
const bool size_compaction = (current_->compaction_score_ >= 1);
const bool seek_compaction = (current_->file_to_compact_ != nullptr);
```

### 1. Size Compaction

#### 1. leve n的sstable确定

1. 得到  level[n]的输入源

```c++
 if (size_compaction) {	
    level = current_->compaction_level_;
    assert(level >= 0);
    assert(level + 1 < config::kNumLevels);
    c = new Compaction(options_, level);

    // Pick the first file that comes after compact_pointer_[level]
    for (size_t i = 0; i < current_->files_[level].size(); i++) {
      FileMetaData* f = current_->files_[level][i];
      if (compact_pointer_[level].empty() ||
          icmp_.Compare(f->largest.Encode(), compact_pointer_[level]) > 0) {
        c->inputs_[0].push_back(f);
        break;
      }
    }
    if (c->inputs_[0].empty()) {
      // Wrap-around to the beginning of the key space
      c->inputs_[0].push_back(current_->files_[level][0]);
    }
  }

```

inputs_数组中存放的是输入源。 

```c++
  // Each compaction reads inputs from "level_" and "level_+1"
  std::vector<FileMetaData*> inputs_[2];  // The two sets of inputs
```

接着看`size_compaction`

```c++
level = current_->compaction_level_;
assert(level >= 0);
assert(level + 1 < config::kNumLevels);
c = new Compaction(options_, level);

// 找到第一个文件，其最大key比 compact_pointer_[level]的key大
// Pick the first file that comes after compact_pointer_[level]
for (size_t i = 0; i < current_->files_[level].size(); i++) {
    FileMetaData* f = current_->files_[level][i];
    if (compact_pointer_[level].empty() ||
        icmp_.Compare(f->largest.Encode(), compact_pointer_[level]) > 0) {
        c->inputs_[0].push_back(f);
        break;
    }
}
// 如果找不到这样的文件，从level头开始（round-robin)
if (c->inputs_[0].empty()) {
    // Wrap-around to the beginning of the key space
    c->inputs_[0].push_back(current_->files_[level][0]);
}
```

size_compation中**只用确定一个要输入的 sstable文件。** 接着看：

```c++
// Files in level 0 may overlap each other, so pick up all overlapping ones
if (level == 0) {
    InternalKey smallest, largest;
    GetRange(c->inputs_[0], &smallest, &largest);
    // Note that the next call will discard the file we placed in
    // c->inputs_[0] earlier and replace it with an overlapping set
    // which will include the picked file.
    current_->GetOverlappingInputs(0, &smallest, &largest, &c->inputs_[0]);
    assert(!c->inputs_[0].empty());
}
```

level0由于存在overlap key，所以需要这些key加入。看注释有一点值得注意：

> 在GetOverlappingInputs函数中，会discard我们之前加入的sstable filemeta, 但是后会replace一个overlap set，这个overlap set将会包含之前picked file。
>
> 是否还包含，这点还有待验证。

#### GetOverlappingInputs函数

继续深追到GetOverlappingInputs:

```c++
// Store in "*inputs" all files in "level" that overlap [begin,end]
void Version::GetOverlappingInputs(int level, const InternalKey* begin,
                                   const InternalKey* end,
                                   std::vector<FileMetaData*>* inputs) {
  assert(level >= 0);
  assert(level < config::kNumLevels);
  inputs->clear();		// 清空
  Slice user_begin, user_end;
  if (begin != nullptr) {
    user_begin = begin->user_key();		// 确定begin
  }
  if (end != nullptr) {
    user_end = end->user_key();			// 确定end
  }
  const Comparator* user_cmp = vset_->icmp_.user_comparator();
  for (size_t i = 0; i < files_[level].size();) {
    FileMetaData* f = files_[level][i++];
    const Slice file_start = f->smallest.user_key();
    const Slice file_limit = f->largest.user_key();
    if (begin != nullptr && user_cmp->Compare(file_limit, user_begin) < 0) {
      // "f" is completely before specified range; skip it
    } else if (end != nullptr && user_cmp->Compare(file_start, user_end) > 0) {
      // "f" is completely after specified range; skip it
    } else {
      inputs->push_back(f);
      if (level == 0) {		// 对于level0，由于存在overlap key，所以可能会扩大begin和end的范围，一旦扩大，重新搜索整个files.
        // Level-0 files may overlap each other.  So check if the newly
        // added file has expanded the range.  If so, restart search.
        if (begin != nullptr && user_cmp->Compare(file_start, user_begin) < 0) {
          user_begin = file_start;
          inputs->clear();
          i = 0;
        } else if (end != nullptr &&
                   user_cmp->Compare(file_limit, user_end) > 0) {
          user_end = file_limit;
          inputs->clear();
          i = 0;
        }
      }
    }
  }
}
```

这部分代码的工作是，根据在PickCompaction() 中选定的文件，确定初步的key range[begin,end], 然后在level0中选择所有与该key range有重叠的sstable，**同时每选择一个还会扩大key range，然后重新add。**

####  举个例子（level0 compaction的sstable选择）

![](https://pic.downk.cc/item/5f84436e1cd1bbb86b05b42f.png)

假设在level 0中选择了**8-14** 这个sstable。现在从头开始遍历，查看是否有需要加入的其他sstable。

第一次，搜索3-6，因为3-6与8-14无重叠，所以无需加入。

第二次，搜索5-12，因为5-12与8-14有重叠，所以加入5-12。 同时由于现在是level0，5-12扩大了搜索域的下界，现在的搜索域改为 **5-14**, 清空所有已经加入的sstable，重头开始。

第三次，搜索3-6，因为3-6与5-14有重叠，所以加入3-6.

第四次，搜索5-12，因为5-12与5-14有重叠，所以5-12.

第五次，所有8-14，因为8-14与5-14有重叠，所以加入8-14.

最后加入的sstable，包括3-6，5-12，8-14.

==上面说的，都是如何level n的输入源，总结起来就是，除了level0，其余level只加入一个sstable，level0可能加入多个sstable==

#### 2. level n+1的sstable确定

level n+1是在==SetupOtherInputs== 函数中确定的

```c++
void VersionSet::SetupOtherInputs(Compaction* c) {
  const int level = c->level();
  InternalKey smallest, largest;

   // 扩展上边界 
  AddBoundaryInputs(icmp_, current_->files_[level], &c->inputs_[0]);
   // 获取当前level n的range
  GetRange(c->inputs_[0], &smallest, &largest);
   // 获取level n+1中与 level n的range重叠的sstable，将这些sstable存放在c->inputs_[1]中
  current_->GetOverlappingInputs(level + 1, &smallest, &largest,
                                 &c->inputs_[1]);

  // Get entire range covered by compaction
  InternalKey all_start, all_limit;
   // 计算leveln level n+1的range
  GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);

  // See if we can grow the number of inputs in "level" without
  // changing the number of "level+1" files we pick up.
  if (!c->inputs_[1].empty()) {
    std::vector<FileMetaData*> expanded0;
    current_->GetOverlappingInputs(level, &all_start, &all_limit, &expanded0);
    AddBoundaryInputs(icmp_, current_->files_[level], &expanded0);
    const int64_t inputs0_size = TotalFileSize(c->inputs_[0]);
    const int64_t inputs1_size = TotalFileSize(c->inputs_[1]);
    const int64_t expanded0_size = TotalFileSize(expanded0);
    if (expanded0.size() > c->inputs_[0].size() &&
        inputs1_size + expanded0_size <
            ExpandedCompactionByteSizeLimit(options_)) {
      InternalKey new_start, new_limit;
      GetRange(expanded0, &new_start, &new_limit);
      std::vector<FileMetaData*> expanded1;
      current_->GetOverlappingInputs(level + 1, &new_start, &new_limit,
                                     &expanded1);
      if (expanded1.size() == c->inputs_[1].size()) {
        Log(options_->info_log,
            "Expanding@%d %d+%d (%ld+%ld bytes) to %d+%d (%ld+%ld bytes)\n",
            level, int(c->inputs_[0].size()), int(c->inputs_[1].size()),
            long(inputs0_size), long(inputs1_size), int(expanded0.size()),
            int(expanded1.size()), long(expanded0_size), long(inputs1_size));
        smallest = new_start;
        largest = new_limit;
        c->inputs_[0] = expanded0;
        c->inputs_[1] = expanded1;
        GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);
      }
    }
  }

  // Compute the set of grandparent files that overlap this compaction
  // (parent == level+1; grandparent == level+2)
  if (level + 2 < config::kNumLevels) {
    current_->GetOverlappingInputs(level + 2, &all_start, &all_limit,
                                   &c->grandparents_);
  }

  // Update the place where we will do the next compaction for this level.
  // We update this immediately instead of waiting for the VersionEdit
  // to be applied so that if the compaction fails, we will try a different
  // key range next time.
  compact_pointer_[level] = largest.Encode().ToString();
  c->edit_.SetCompactPointer(level, largest);
}
```

#### 核心代码

核心在着几行：

```c++
   // 获取当前level n的range
  GetRange(c->inputs_[0], &smallest, &largest);
   // 获取level n+1中与 level n的range重叠的sstable，将这些sstable存放在c->inputs_[1]中
  current_->GetOverlappingInputs(level + 1, &smallest, &largest,
                                 &c->inputs_[1]);

  // Get entire range covered by compaction
  InternalKey all_start, all_limit;
   // 计算leveln level n+1的range
  GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);
```

#### level n新SStable的加入

那下面还有一大段是做什么用的？

```c++
// See if we can grow the number of inputs in "level" without
// changing the number of "level+1" files we pick up.
if (!c->inputs_[1].empty()) {
    std::vector<FileMetaData*> expanded0;
    current_->GetOverlappingInputs(level, &all_start, &all_limit, &expanded0);
    AddBoundaryInputs(icmp_, current_->files_[level], &expanded0);
    const int64_t inputs0_size = TotalFileSize(c->inputs_[0]);
    const int64_t inputs1_size = TotalFileSize(c->inputs_[1]);
    const int64_t expanded0_size = TotalFileSize(expanded0);
    if (expanded0.size() > c->inputs_[0].size() &&
        inputs1_size + expanded0_size <
        ExpandedCompactionByteSizeLimit(options_)) {
        InternalKey new_start, new_limit;
        GetRange(expanded0, &new_start, &new_limit);
        std::vector<FileMetaData*> expanded1;
        current_->GetOverlappingInputs(level + 1, &new_start, &new_limit,
                                       &expanded1);
        if (expanded1.size() == c->inputs_[1].size()) {
            Log(options_->info_log,
                "Expanding@%d %d+%d (%ld+%ld bytes) to %d+%d (%ld+%ld bytes)\n",
                level, int(c->inputs_[0].size()), int(c->inputs_[1].size()),
                long(inputs0_size), long(inputs1_size), int(expanded0.size()),
                int(expanded1.size()), long(expanded0_size), long(inputs1_size));
            smallest = new_start;
            largest = new_limit;
            c->inputs_[0] = expanded0;
            c->inputs_[1] = expanded1;
            GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);
        }
    }
}
```

这段代码的意思是，在选中level n+1层的sstable后，可能还可以加入一些level n的sstable。举个例子：

![](https://pic.downk.cc/item/5f84437a1cd1bbb86b05bbf2.png)

假设现在在level n选择的是8-12这个sstable，则在level n+1 可以选择8-9，10-16着连个sstable。 这一切做完后，发现在level n中存在13-15这个sstable，加入13-15并不影响level n+1 sstable的选择。

上面那段代码就是做这个工作。举个不会加入新sstable的例子：

![](https://pic.downk.cc/item/5f8443831cd1bbb86b05c1ea.png)

在这样的情况下，13-18的重新加入，level n+1 需要重新加入17-20，所以不应该加入13-18.

但是再看下一个例子：

![](https://pic.downk.cc/item/5f84438c1cd1bbb86b05c90f.png)

这个例子中，虽然新加入level 13-18，扩大了inputs_的范围，但是由于并没有造成level n+1的sstable重新选择，所以依然可以加入13-18.

ok，例子说了好几个，正式看代码。

1. 计算如果要在level n要重新加入sstable，那加入后的第level n层的compaction sstable的总大小为多少（即expanded0_size)

```c++
 current_->GetOverlappingInputs(level, &all_start, &all_limit, &expanded0);
 AddBoundaryInputs(icmp_, current_->files_[level], &expanded0);
 const int64_t inputs0_size = TotalFileSize(c->inputs_[0]);
 const int64_t inputs1_size = TotalFileSize(c->inputs_[1]);
 const int64_t expanded0_size = TotalFileSize(expanded0);
```

2. 如果新加入后，level n和level n+1的总size小于一次compaction的总size，则考虑加入。

```c++
  if (expanded0.size() > c->inputs_[0].size() &&
        inputs1_size + expanded0_size <
            ExpandedCompactionByteSizeLimit(options_)) {
      InternalKey new_start, new_limit;
      GetRange(expanded0, &new_start, &new_limit);
      std::vector<FileMetaData*> expanded1;
      current_->GetOverlappingInputs(level + 1, &new_start, &new_limit,
                                     &expanded1);
```

一次compaction的总size：

```c++
// Maximum number of bytes in all compacted files.  We avoid expanding
// the lower level file set of a compaction if it would make the
// total compaction cover more than this many bytes.
static int64_t ExpandedCompactionByteSizeLimit(const Options* options) {
  return 25 * TargetFileSize(options);			// 默认是50M
}

static size_t TargetFileSize(const Options* options) {
  return options->max_file_size;
}

// Leveldb will write up to this amount of bytes to a file before
// switching to a new one.
// Most clients should leave this parameter alone.  However if your
// filesystem is more efficient with larger files, you could
// consider increasing the value.  The downside will be longer
// compactions and hence longer latency/performance hiccups.
// Another reason to increase this parameter might be when you are
// initially populating a large database.
size_t max_file_size = 2 * 1024 * 1024;
```

3. 在level n中加入新sstable，但没有引起level n+1的sstable选择，则加入这个新sstable。

```c++
if (expanded1.size() == c->inputs_[1].size()) {
    Log(options_->info_log,
        "Expanding@%d %d+%d (%ld+%ld bytes) to %d+%d (%ld+%ld bytes)\n",
        level, int(c->inputs_[0].size()), int(c->inputs_[1].size()),
        long(inputs0_size), long(inputs1_size), int(expanded0.size()),
        int(expanded1.size()), long(expanded0_size), long(inputs1_size));
    smallest = new_start;
    largest = new_limit;
    c->inputs_[0] = expanded0;
    c->inputs_[1] = expanded1;
    GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);
}
    
```

#### 3. 记录下一轮的压缩起始文件

```c++
  // Update the place where we will do the next compaction for this level.
  // We update this immediately instead of waiting for the VersionEdit
  // to be applied so that if the compaction fails, we will try a different
  // key range next time.
  compact_pointer_[level] = largest.Encode().ToString();
  c->edit_.SetCompactPointer(level, largest);
```

#### 4. AddBoundaryInputs

在前面的介绍中，我曾说过，除了level0， 其余level中 inputs数组的inputs[0]总是先只选择一个文件，然后通过`VersionSet::SetupOtherInputs`，确定inputs_[1],  当inputs\_[1]确定后，会回退到input\_[0]，尝试增加input\_[0]的文件。其实除了这里会增加input[0]的选择外，还有一处可能增加Inputs[0]的选择。那就是AddBoundaryInputs函数：

```c++
void VersionSet::SetupOtherInputs(Compaction* c) {
  const int level = c->level();
  InternalKey smallest, largest;

   // !! 这里
  AddBoundaryInputs(icmp_, current_->files_[level], &c->inputs_[0]);
  GetRange(c->inputs_[0], &smallest, &largest);

  current_->GetOverlappingInputs(level + 1, &smallest, &largest,
                                 &c->inputs_[1]);
 ...
}
```

==AddBoundaryInputs==

```c++

// Extracts the largest file b1 from |compaction_files| and then searches for a
// b2 in |level_files| for which user_key(u1) = user_key(l2). If it finds such a
// file b2 (known as a boundary file) it adds it to |compaction_files| and then
// searches again using this new upper bound.
//
// If there are two blocks, b1=(l1, u1) and b2=(l2, u2) and
// user_key(u1) = user_key(l2), and if we compact b1 but not b2 then a
// subsequent get operation will yield an incorrect result because it will
// return the record from b2 in level i rather than from b1 because it searches
// level by level for records matching the supplied user key.
//
// parameters:
//   in     level_files:      List of files to search for boundary files.
//   in/out compaction_files: List of files to extend by adding boundary files.
void AddBoundaryInputs(const InternalKeyComparator& icmp,
                       const std::vector<FileMetaData*>& level_files,
                       std::vector<FileMetaData*>* compaction_files) {
  InternalKey largest_key;

  // Quick return if compaction_files is empty.
  if (!FindLargestKey(icmp, *compaction_files, &largest_key)) {
    return;
  }

  bool continue_searching = true;
  while (continue_searching) {
    FileMetaData* smallest_boundary_file =
        FindSmallestBoundaryFile(icmp, level_files, largest_key);

    // If a boundary file was found advance largest_key, otherwise we're done.
    if (smallest_boundary_file != NULL) {
      compaction_files->push_back(smallest_boundary_file);
      largest_key = smallest_boundary_file->largest;
    } else {
      continue_searching = false;
    }
  }
}
```

在AddBoundaryInputs调用前，我们已经确定了inputs[0], 如果不考虑level0的话，对于其他level，inputs[0]为一个sstable。这个sstable存在一个key range[low,high], 我们都知道 sstable内部存的key是InternalKey, InternalKey内部封装了user key。如下图：

![](https://pic.downk.cc/item/5f8442441cd1bbb86b04da1b.png)

因为只要internel key不相同，那么就可认为key是不重叠的。也就是说只要(key, sequence number,type)三个任一个不同，就可以认为key是不同。 那现在可能就存在这样一个情况， 相邻两个sstable的user key相同。如下图表示：

![](https://pic.downk.cc/item/5f84439f1cd1bbb86b05d684.png)

sstable 1中的上届(upper)的user key 和 sstable 2的下届(lower)的user key相同，都为key2. 假设我们当前inputs[0]就是sstable 1。 那么AddBoundaryInputs函数的作用就是将sstable 2添加到inputs[0]中。 当然这是一个递归的过程，加完sstable 2，可能sstable 3的下届user key又和sstable 2的上届相同，所以继续添加。

为什么要这样加？因为若是不添加，sstable 1经过compaction陷入下层，而sstable 2还在上层。那么下一次Get操作时，leveldb将由上往下search，这样找到的key在sstable 2，而sstable 2中的key不是最新的，这显然是不对的。

### 2. Seek Compaction

```c++
// PickCompaction函数中
level = current_->file_to_compact_level_;
c = new Compaction(options_, level);
c->inputs_[0].push_back(current_->file_to_compact_);
```

根据`file_to_compact_`指针和`file_to_compact_level_`即可得到 ==一个输入源文件==

```c++
// Next file to compact based on seek stats.
FileMetaData* file_to_compact_;
int file_to_compact_level_;
```

#### 1. 什么时候触发一次seek compaction？

我们从`file_to_compact_`反追踪，可以发现 UpdateStats 函数中对file_to_compact_进行了赋值。

```c++
bool Version::UpdateStats(const GetStats& stats) {
  FileMetaData* f = stats.seek_file;
  if (f != nullptr) {
    f->allowed_seeks--;
    if (f->allowed_seeks <= 0 && file_to_compact_ == nullptr) {
      file_to_compact_ = f;
      file_to_compact_level_ = stats.seek_file_level;
      return true;
    }
  }
  return false;
}
```

从这里可以看出，当一个文件的allowed_seeks<=0时，就可以对这个file进行compaction。

#### 2. allowed_seeks--的时机 (1) & DBImpl::Get的简单分析

继续反追踪 ，什么时候调用UpdateStats? **在DBImpl::Get函数中**

==DBImpl::Get==

```c++
Status DBImpl::Get(const ReadOptions& options, const Slice& key,
                   std::string* value) {
  Status s;
  MutexLock l(&mutex_);
  SequenceNumber snapshot;
  if (options.snapshot != nullptr) {
    snapshot =
        static_cast<const SnapshotImpl*>(options.snapshot)->sequence_number();
  } else {
    snapshot = versions_->LastSequence();
  }

  MemTable* mem = mem_;
  MemTable* imm = imm_;
  Version* current = versions_->current();
  mem->Ref();
  if (imm != nullptr) imm->Ref();
  current->Ref();

  bool have_stat_update = false;
  Version::GetStats stats;

  // Unlock while reading from files and memtables
  {
    mutex_.Unlock();
    // First look in the memtable, then in the immutable memtable (if any).
    LookupKey lkey(key, snapshot);
    if (mem->Get(lkey, value, &s)) { // 1.首先在mem中
      // Done
    } else if (imm != nullptr && imm->Get(lkey, value, &s)) {	// 2.然后在imm中
      // Done
    } else {
      s = current->Get(options, lkey, value, &stats);		// 3.再去sstable， ！！注意这里的status
      have_stat_update = true;
    }
    mutex_.Lock();
  }

  if (have_stat_update && current->UpdateStats(stats)) { // 4.这里调用了UpdateStats
    MaybeScheduleCompaction();
  }
  mem->Unref();
  if (imm != nullptr) imm->Unref();
  current->Unref();
  return s;
}

```

==Version::Get==

```c++
Status Version::Get(const ReadOptions& options, const LookupKey& k,
                    std::string* value, GetStats* stats) {
  stats->seek_file = nullptr;
  stats->seek_file_level = -1;

  struct State {
     ...

    static bool Match(void* arg, int level, FileMetaData* f) {
      State* state = reinterpret_cast<State*>(arg);
		
       ...

      // Not reached. Added to avoid false compilation warnings of
      // "control reaches end of non-void function".
      return false;
    }
  };

  State state;
  state.found = false;
   // ！！stats控制转移到State类
  state.stats = stats;
  state.last_file_read = nullptr;
  state.last_file_read_level = -1;
	
   ...

  ForEachOverlapping(state.saver.user_key, state.ikey, &state, &State::Match);

  return state.found ? state.s : Status::NotFound(Slice());
}
```

最后走到了ForEachOverlapping，参数中传入了一个函数指针, State::Match, 后序在分析。先看ForEachOverlapping:

==ForEachOverlapping==

```c++

void Version::ForEachOverlapping(Slice user_key, Slice internal_key, void* arg,
                                 bool (*func)(void*, int, FileMetaData*)) {
  const Comparator* ucmp = vset_->icmp_.user_comparator();

  // Search level-0 in order from newest to oldest.
   // 加入第0层文件
  std::vector<FileMetaData*> tmp;
  tmp.reserve(files_[0].size());
  for (uint32_t i = 0; i < files_[0].size(); i++) {
    FileMetaData* f = files_[0][i];
    if (ucmp->Compare(user_key, f->smallest.user_key()) >= 0 &&
        ucmp->Compare(user_key, f->largest.user_key()) <= 0) {
      tmp.push_back(f);
    }
  }
  if (!tmp.empty()) {
      // 按照新旧排序，由新到旧
    std::sort(tmp.begin(), tmp.end(), NewestFirst);
    for (uint32_t i = 0; i < tmp.size(); i++) {
      if (!(*func)(arg, 0, tmp[i])) {	// 调用State::Match, 由于第0层无序，所以可能需要多次调用
        return;
      }
    }
  }

  // Search other levels.
  for (int level = 1; level < config::kNumLevels; level++) {
    size_t num_files = files_[level].size();
    if (num_files == 0) continue;

    // Binary search to find earliest index whose largest key >= internal_key.
    uint32_t index = FindFile(vset_->icmp_, files_[level], internal_key);
    if (index < num_files) {
      FileMetaData* f = files_[level][index];
      if (ucmp->Compare(user_key, f->smallest.user_key()) < 0) {
        // All of "f" is past any data for user_key
      } else {										// 到这里，确定了 internal_key 一定是在 本file的key range中，即overlap
        if (!(*func)(arg, level, f)) {				// 一个level，只会调用
          return;
        }
      }
    }
  }
}
```

在ForEachOverlapping函数中，首先搜索level0，在level 0中找到一些sstable，这些sstable的key range包含了user_key. 然后不断调用Match函数，直到找到相应file。 **由于level0的无序性，所以Match函数可能被调用了多次。**

如果level0中找不到能够匹配的SStable，就逐层往下，因为level1--level6都是有序的，**所以每层最多有一个sstable**，满足其key range包含指定user_key， **所以每层只用调用一次Match函数。**

最后，看看：

==Match==

```c++

static bool Match(void* arg, int level, FileMetaData* f) {
    State* state = reinterpret_cast<State*>(arg);

    if (state->stats->seek_file == nullptr &&
        state->last_file_read != nullptr) {			// 走到这个分支，说明Match函数至少已经被调用过一次，也就是说至少浪费了一次io
        // We have had more than one seek for this read.  Charge the 1st file.
        state->stats->seek_file = state->last_file_read;
        state->stats->seek_file_level = state->last_file_read_level;
    }
	
    state->last_file_read = f;
    state->last_file_read_level = level;

    state->s = state->vset->table_cache_->Get(*state->options, f->number,
                                              f->file_size, state->ikey,
                                              &state->saver, SaveValue);
    if (!state->s.ok()) {
        state->found = true;
        return false;   // 不再search
    }
    switch (state->saver.state) {
        case kNotFound:
            return true;  // Keep searching in other files
        case kFound:
            state->found = true;
            return false;  // 不再search
        case kDeleted:
            return false;  // 不再search
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

从这个函数中也终于可以找到`seek_file`被赋值的地方：

```c++
    if (state->stats->seek_file == nullptr &&
        state->last_file_read != nullptr) {			// 走到这个分支，说明Match函数至少已经被调用过一次，也就是说至少浪费了一次io
        // We have had more than one seek for this read.  Charge the 1st file.
        state->stats->seek_file = state->last_file_read;
        state->stats->seek_file_level = state->last_file_read_level;
    }
```

不过要进入这个分析，Match已经至少被执行了一次，所以现在记录的是上一次调用Match函数时所用到的file。这个file被访问了，但是却没有找到正确的key。所以它浪费了io，进而在后序的`UpdateStats`函数中，它的**allowed_seeks会被--**。

**不过感觉这里记录到第一个浪费了io的sstable，后序可能还会存在浪费io的sstable，可能是因为层别越低，访问频率越高，所以越需要快点将其allowed_seeks减小。**

ok，总结一下，花个流程图。

#### seek_compaction的流程图（何时触发，如何触发）

<img src="https://pic.downk.cc/item/5f8443b11cd1bbb86b05e0f2.png" style="zoom:50%;" />

#### 3.  allowed_seeks--的时机 (2) & DBIter

这部分请参考DBIter。

## 3. Compaction的执行流程

选择了要执行的Compaction文件后，剩下的就是执行Compaction：

==DBImpl::BackgroundCompaction==

```c++
 Status status;
  if (c == nullptr) {
    // Nothing to do
  } else if (!is_manual && c->IsTrivialMove()) {	// trivial move: 下层没有本层的重叠key，修改元数据，移动到下层
    // Move file to next level
    assert(c->num_input_files(0) == 1);
    FileMetaData* f = c->input(0, 0);
    c->edit()->RemoveFile(c->level(), f->number);
    c->edit()->AddFile(c->level() + 1, f->number, f->file_size, f->smallest,
                       f->largest);
    status = versions_->LogAndApply(c->edit(), &mutex_);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    VersionSet::LevelSummaryStorage tmp;
    Log(options_.info_log, "Moved #%lld to level-%d %lld bytes %s: %s\n",
        static_cast<unsigned long long>(f->number), c->level() + 1,
        static_cast<unsigned long long>(f->file_size),
        status.ToString().c_str(), versions_->LevelSummary(&tmp));
  } else {					// ！！实际compaction的代码
    CompactionState* compact = new CompactionState(c);
    status = DoCompactionWork(compact);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    CleanupCompaction(compact);
    c->ReleaseInputs();
    RemoveObsoleteFiles();
  }
  delete c;
```

上面的代码说明了：

1. 若没有选择的compaction file，则忽略。

2. IsTrivialMove，若选择的compaction文件，level n没有和level n+1重叠，并且level n的文件没有与过多的祖父level文件重叠，则直接修改元数据（verisonedit）即可，不必compaction。

   ```c++
   bool Compaction::IsTrivialMove() const {
     const VersionSet* vset = input_version_->vset_;
     // Avoid a move if there is lots of overlapping grandparent data.
     // Otherwise, the move could create a parent file that will require
     // a very expensive merge later on.
     return (num_input_files(0) == 1 && num_input_files(1) == 0 &&
             TotalFileSize(grandparents_) <=
                 MaxGrandParentOverlapBytes(vset->options_));
   }
   
   ```

3. 否则，执行真正的**compaction**。

### DoCompactionWork

```c++

Status DBImpl::DoCompactionWork(CompactionState* compact) {
	
   ...
  if (snapshots_.empty()) {
    compact->smallest_snapshot = versions_->LastSequence();
  } else {
    compact->smallest_snapshot = snapshots_.oldest()->sequence_number();
  }
   // 创建迭代器, 内部同过mergeiterator对本次要compaction的文件做“排序”（没有排序，只不过通过iter依次访问数据，得到的结果就是排序后的结果）
  Iterator* input = versions_->MakeInputIterator(compact->compaction);

  // Release mutex while we're actually doing the compaction work
  mutex_.Unlock();

  input->SeekToFirst();
  Status status;
  ParsedInternalKey ikey;
  std::string current_user_key;
  bool has_current_user_key = false;
  SequenceNumber last_sequence_for_key = kMaxSequenceNumber;
  while (input->Valid() && !shutting_down_.load(std::memory_order_acquire)) {
    // Prioritize immutable compaction work
     // 首先做immtable的dump
    if (has_imm_.load(std::memory_order_relaxed)) {
      const uint64_t imm_start = env_->NowMicros();
      mutex_.Lock();
      if (imm_ != nullptr) {
        CompactMemTable();
        // Wake up MakeRoomForWrite() if necessary.
        background_work_finished_signal_.SignalAll();
      }
      mutex_.Unlock();
      imm_micros += (env_->NowMicros() - imm_start);
    }

    Slice key = input->key();
    if (compact->compaction->ShouldStopBefore(key) &&
        compact->builder != nullptr) {
        //检查当前输出文件是否与level+2层文件有过多冲突，如果是就要完成当前输出文件,并产生新的输出文件
      status = FinishCompactionOutputFile(compact, input);
      if (!status.ok()) {
        break;
      }
    }
	
    // 下面这里是关键！！
    // Handle key/value, add to state, etc.
    bool drop = false;
    if (!ParseInternalKey(key, &ikey)) {
      // Do not hide error keys
      current_user_key.clear();
      has_current_user_key = false;
      last_sequence_for_key = kMaxSequenceNumber;
    } else {		// 正常情况下走这里
      if (!has_current_user_key ||		// 已经有了user_key?
          user_comparator()->Compare(ikey.user_key, Slice(current_user_key)) !=	
              0) {		// 要分析的user_key是否和之前的user_key相同？
        // First occurrence of this user key
        current_user_key.assign(ikey.user_key.data(), ikey.user_key.size());
        has_current_user_key = true;
          // 第一次出现的user_key不允许删除
        last_sequence_for_key = kMaxSequenceNumber;
      }

      if (last_sequence_for_key <= compact->smallest_snapshot) {	//	进入这个判断，一定时是出现了重复key. 如果前一个序列号都已经比当前smallest_snapshot小了， 现在key的序列号肯定更小，也肯定小于smallest_snapshot，所以直接drop
        // Hidden by an newer entry for same user key
        drop = true;  // (A)
      } else if (ikey.type == kTypeDeletion &&			
                 ikey.sequence <= compact->smallest_snapshot &&
                 compact->compaction->IsBaseLevelForKey(ikey.user_key)) {
      // 前一个key的序列号时大于smallest_snapshot，而当前key的序列号小于smallest_snapshot,说名当前key是距离smallest_snapshot最近的key，
          // !!!!!! 待完善
          // For this user key:
        // (1) there is no data in higher levels
        // (2) data in lower levels will have larger sequence numbers
        // (3) data in layers that are being compacted here and have
        //     smaller sequence numbers will be dropped in the next
        //     few iterations of this loop (by rule (A) above).
        // Therefore this deletion marker is obsolete and can be dropped.
        drop = true;
      }

      last_sequence_for_key = ikey.sequence;
    }
      
    if (!drop) {		// 不需要删除，则写入到文件
      // Open output file if necessary
      if (compact->builder == nullptr) {
        status = OpenCompactionOutputFile(compact);
        if (!status.ok()) {
          break;
        }
      }
      if (compact->builder->NumEntries() == 0) {
        compact->current_output()->smallest.DecodeFrom(key);
      }
      compact->current_output()->largest.DecodeFrom(key);
      compact->builder->Add(key, input->value());

      // Close output file if it is big enough
      if (compact->builder->FileSize() >=
          compact->compaction->MaxOutputFileSize()) {
        status = FinishCompactionOutputFile(compact, input);
        if (!status.ok()) {
          break;
        }
      }
    }

    input->Next();
  }

  
  if (status.ok() && shutting_down_.load(std::memory_order_acquire)) {
    status = Status::IOError("Deleting DB during compaction");
  }
  if (status.ok() && compact->builder != nullptr) {
    status = FinishCompactionOutputFile(compact, input);
  }
  if (status.ok()) {
    status = input->status();
  }
  delete input;
  input = nullptr;

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros - imm_micros;
  for (int which = 0; which < 2; which++) {
    for (int i = 0; i < compact->compaction->num_input_files(which); i++) {
      stats.bytes_read += compact->compaction->input(which, i)->file_size;
    }
  }
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    stats.bytes_written += compact->outputs[i].file_size;
  }

  mutex_.Lock();
  stats_[compact->compaction->level() + 1].Add(stats);

  if (status.ok()) {	// 保存此次压缩结果的元数据 
    status = InstallCompactionResults(compact);
  }
  if (!status.ok()) {
    RecordBackgroundError(status);
  }
  VersionSet::LevelSummaryStorage tmp;
  Log(options_.info_log, "compacted to: %s", versions_->LevelSummary(&tmp));
  return status;
  return status;
}
```

这里的整体流程分3步：

1. 创建迭代器，内部对本次要compact的文件进行排序。
2. 丢弃那些被删除的或者旧的文件。
3. 写入新文件。

#### 1. 排序

```c++
  Iterator* input = versions_->MakeInputIterator(compact->compaction);
```

```c++
Iterator* VersionSet::MakeInputIterator(Compaction* c) {
  ReadOptions options;
  options.verify_checksums = options_->paranoid_checks;
  options.fill_cache = false;

  // Level-0 files have to be merged together.  For other levels,
  // we will make a concatenating iterator per level.
  // TODO(opt): use concatenating iterator for level-0 if there is no overlap
  const int space = (c->level() == 0 ? c->inputs_[0].size() + 1 : 2);
   // !!! list中的每个iter，都指向了一个即将被compaction的sstable
  Iterator** list = new Iterator*[space];
  int num = 0;
  for (int which = 0; which < 2; which++) {
    if (!c->inputs_[which].empty()) {
      if (c->level() + which == 0) {	// 对第0层的files，通过table_cache创建iter
        const std::vector<FileMetaData*>& files = c->inputs_[which];
        for (size_t i = 0; i < files.size(); i++) {
          list[num++] = table_cache_->NewIterator(options, files[i]->number,
                                                  files[i]->file_size);
        }
      } else {	// 非第0层的fiels，使用TwoLevelIterator来迭代（index iter 和 data iter)
        // Create concatenating iterator for the files from this level
        list[num++] = NewTwoLevelIterator(
            new Version::LevelFileNumIterator(icmp_, &c->inputs_[which]),
            &GetFileIterator, table_cache_, options);
      }
    }
  }
  assert(num <= space);
   // 所有需要的compaction file都有一个iter，现在需要归并排序，这通过mergeiteraotr实现
  Iterator* result = NewMergingIterator(&icmp_, list, num);
  delete[] list;
  return result;
}
```

关于mergeiteraotr的具体实现，可参考 mergeitator。

#### 2. 丢弃不需要的kv pairs（待完善）

```c++
 if (!has_current_user_key ||			// 已经有了user_key?
          user_comparator()->Compare(ikey.user_key, Slice(current_user_key)) !=	
              0) {					// 要分析的user_key是否和之前的user_key相同？
        // First occurrence of this user key
        current_user_key.assign(ikey.user_key.data(), ikey.user_key.size());
        has_current_user_key = true;
          // 第一次出现的user_key不允许删除
        last_sequence_for_key = kMaxSequenceNumber;
      }

      if (last_sequence_for_key <= compact->smallest_snapshot) {	//	进入这个判断，一定时是出现了重复key. 如果前一个序列号都已经比当前smallest_snapshot小了， 现在key的序列号肯定更小，也肯定小于smallest_snapshot，所以直接drop
        // Hidden by an newer entry for same user key
        drop = true;  // (A)
      } else if (ikey.type == kTypeDeletion &&			
                 ikey.sequence <= compact->smallest_snapshot &&
                 compact->compaction->IsBaseLevelForKey(ikey.user_key)) {
      // 前一个key的序列号大于smallest_snapshot，而当前key的序列号小于smallest_snapshot,说名当前key是距离smallest_snapshot最近的key，
          // !!!!!! 待完善
          // For this user key:
        // (1) there is no data in higher levels
        // (2) data in lower levels will have larger sequence numbers
        // (3) data in layers that are being compacted here and have
        //     smaller sequence numbers will be dropped in the next
        //     few iterations of this loop (by rule (A) above).
        // Therefore this deletion marker is obsolete and can be dropped.
        drop = true;
      }

      last_sequence_for_key = ikey.sequence;
    }
      
```

这部分的第3个分支每看懂。后序补充。

#### 3. 写入不需要drop的kv

```c++
  if (!drop) {
      // Open output file if necessary
      if (compact->builder == nullptr) {	// builder为空，则打开
        status = OpenCompactionOutputFile(compact);
        if (!status.ok()) {
          break;
        }
      }
      if (compact->builder->NumEntries() == 0) {
        compact->current_output()->smallest.DecodeFrom(key);
      }
      compact->current_output()->largest.DecodeFrom(key);
      compact->builder->Add(key, input->value());

      // Close output file if it is big enough
      if (compact->builder->FileSize() >=
          compact->compaction->MaxOutputFileSize()) {
        status = FinishCompactionOutputFile(compact, input);
        if (!status.ok()) {
          break;
        }
      }
    }
```

第一步：OpenCompactionOutputFile

```c++
Status DBImpl::OpenCompactionOutputFile(CompactionState* compact) {
  assert(compact != nullptr);
  assert(compact->builder == nullptr);
  uint64_t file_number;
  {
    mutex_.Lock();
    file_number = versions_->NewFileNumber();
      // 注意这里的pending_outputs
    pending_outputs_.insert(file_number);
    CompactionState::Output out;
    out.number = file_number;
    out.smallest.Clear();
    out.largest.Clear();
     // 加入到outputs
    compact->outputs.push_back(out);
    mutex_.Unlock();
  }

  // Make the output file
  std::string fname = TableFileName(dbname_, file_number);
  Status s = env_->NewWritableFile(fname, &compact->outfile);
  if (s.ok()) {
    compact->builder = new TableBuilder(options_, compact->outfile);
  }
  return s;
}
```

这里有两个点要注意：

1. pending_outputs_， 我们来看的定义：

   ```c++
     // Set of table files to protect from deletion because they are
     // part of ongoing compactions.
     std::set<uint64_t> pending_outputs_ GUARDED_BY(mutex_);
   ```

   它是为了避免tables files被误删除的而设计的。那在哪里会被误删除？在CompactMemTable中：

   ```c++
     if (s.ok()) {
       // Commit to the new state
       imm_->Unref();
       imm_ = nullptr;
       has_imm_.store(false, std::memory_order_release);
         // 这里会删除
       RemoveObsoleteFiles();
     } else {
       RecordBackgroundError(s);
     }
   ```

   RemoveObsoleteFiles中使用到了pending_outputs_，因为在合并过程中，刚生成的sstable还不是“live”的，通过`pending_outputs_`将它们当成 live 的就不会被删除了。

2. 将需要保存的kv，放在compact->outputs中。

   ```c++
   // 加入到outputs
   compact->outputs.push_back(out);
   ```

第二步，记录最小key和最大key，同时并将要保存的kv加入到builder中。

```c++
    if (compact->builder->NumEntries() == 0) {
        compact->current_output()->smallest.DecodeFrom(key);
      }
      compact->current_output()->largest.DecodeFrom(key);
      compact->builder->Add(key, input->value());
```

第三步，如果当前table已经过大，则输出：

```c++
 // Close output file if it is big enough
 if (compact->builder->FileSize() >=
     compact->compaction->MaxOutputFileSize()) {
     status = FinishCompactionOutputFile(compact, input);
     if (!status.ok()) {
     break;
     }
 }
```

==FinishCompactionOutputFile==

```c++
Status DBImpl::FinishCompactionOutputFile(CompactionState* compact,
                                          Iterator* input) {
  assert(compact != nullptr);
  assert(compact->outfile != nullptr);
  assert(compact->builder != nullptr);

  const uint64_t output_number = compact->current_output()->number;
  assert(output_number != 0);

  // Check for iterator errors
  Status s = input->status();
  const uint64_t current_entries = compact->builder->NumEntries();
  if (s.ok()) {/
      // 构建sstable
    s = compact->builder->Finish();
  } else {
    compact->builder->Abandon();
  }
   // 统计sstable的大小
  const uint64_t current_bytes = compact->builder->FileSize();
  compact->current_output()->file_size = current_bytes;
  compact->total_bytes += current_bytes;
  delete compact->builder;
  compact->builder = nullptr;

  // Finish and check for file errors
  if (s.ok()) {
      // 写入sstable
    s = compact->outfile->Sync();
  }
  if (s.ok()) {
    s = compact->outfile->Close();
  }
  delete compact->outfile;
  compact->outfile = nullptr;

  if (s.ok() && current_entries > 0) {	// 验证table是否有效
    // Verify that the table is usable
    Iterator* iter =
        table_cache_->NewIterator(ReadOptions(), output_number, current_bytes);
    s = iter->status();
    delete iter;
    if (s.ok()) {
      Log(options_.info_log, "Generated table #%llu@%d: %lld keys, %lld bytes",
          (unsigned long long)output_number, compact->compaction->level(),
          (unsigned long long)current_entries,
          (unsigned long long)current_bytes);
    }
  }
  return s;
}
```

FinishCompactionOutputFile主完成的工作就是将之前加入的有效kv落盘成一个新的sstable。

1. `compact->builder->Finish();` 构建sstable的所有块，data block, meta block meta index block, index block,footer.
2. 记录本次形成的新sstable的大小
3. 落盘，compact->outfile->Sync();
4. 校验是否正确

#### 4. 元数据修改

经过前面几步，compaction过后的sstable已经持久化到设备上了，现在要做的是修改本次压缩过程中涉及到的sstable的元数据，删除用来compaction的数据，生成compaction后的sstable的元数据。这些都通过一次versionedit来表示，然后通过LogAndApply应用这个edit，生成新edit。

```c++

Status DBImpl::InstallCompactionResults(CompactionState* compact) {
  mutex_.AssertHeld();
  Log(options_.info_log, "Compacted %d@%d + %d@%d files => %lld bytes",
      compact->compaction->num_input_files(0), compact->compaction->level(),
      compact->compaction->num_input_files(1), compact->compaction->level() + 1,
      static_cast<long long>(compact->total_bytes));

  // Add compaction outputs
  compact->compaction->AddInputDeletions(compact->compaction->edit());	// 删除
  const int level = compact->compaction->level();
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    const CompactionState::Output& out = compact->outputs[i];
    compact->compaction->edit()->AddFile(level + 1, out.number, out.file_size,
                                         out.smallest, out.largest);		// 结果
  }
  return versions_->LogAndApply(compact->compaction->edit(), &mutex_);		// 应用edit
}
```

代码逻辑比较简单，将本次操作过程中涉及的文件都加入一个edit中，然后通过LogAndApply应用即可。

