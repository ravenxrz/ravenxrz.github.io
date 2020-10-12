---
title: leveldb源码阅读记录-Version数据结构及MANIFEST文件
categories: leveldb
abbrlink: 90a4812e
date: 2020-10-12 11:19:43
tags:
---

文本要介绍一个在leveldb中相当重要的数据结构 - Version及其相关的VersionEdit和VersionSet。理解了这些内容后，我们会提到leveldb系统中MANIFEST文件。

<!--more-->

# 1. Version

为什么要用version控制？

对于同一笔记录，如果读和写同一时间发生，reader可能读到不一致的数据或者是修改了一半的数据。对于这种情况，有三种常见的解决方法：

- 悲观锁  ，最简单的处理方式，就是加锁保护，写的时候不许读，读的时候不许写。效率低。 
- 乐观锁   它假设多用户并发的事物在处理时不会彼此互相影响，各事务能够在不产生锁的的情况下处理各自影响的那部分数据。在提交数据更新之前，每个事务会先检查在该事务读取数据后，有没有其他事务又修改了该数据。 果其他事务有更新的话，正在提交的事务会进行回滚;这样做不会有锁竞争更不会产生死锁， 但如果数据竞争的概率较高，效率也会受影响 。
- MVCC，MVCC是一个数据库常用的概念。Multiversion concurrency control多版本并发控制。每一个执行操作的用户，看到的都是数据库特定时刻的的快照(snapshot), writer的任何未完成的修改都不会被其他的用户所看到;当对数据进行更新的时候并是不直接覆盖，而是先进行标记, 然后在其他地方添加新的数据，从而形成一个新版本, 此时再来读取的reader看到的就是最新的版本了。所以这种处理策略是维护了多个版本的数据的,但只有一个是最新的。

leveldb中的version就是MVCC思想的体现。

## 1. VersionEdit结构

前面说了整个version相关的数据结构是用来实现MVCC的，这也意味这系统中存在多个版本，那我们是如何从上一个历史版本走到这个版本的？这就是通过VersionEdit来实现的。

VersionEdit是LevelDB两个Version之间的差量，即：

```
Versoin0 + VersoinEdit = Version1
```

差量包括本次操作，新增的文件和删除的文件。

看看VersionEdit的成员：

```c++
 private:
  friend class VersionSet;

  typedef std::set<std::pair<int, uint64_t>> DeletedFileSet;

  std::string comparator_;
  uint64_t log_number_;
  uint64_t prev_log_number_;
  uint64_t next_file_number_;	
  SequenceNumber last_sequence_;
  bool has_comparator_;
  bool has_log_number_;
  bool has_prev_log_number_;
  bool has_next_file_number_;
  bool has_last_sequence_;

  std::vector<std::pair<int, InternalKey>> compact_pointers_;		// 存放这个version的压缩指针，pair.first对应哪一个level， pair.second 对应哪一个key开始compaction
  DeletedFileSet deleted_files_;				// 本次操作要删除的文件
  std::vector<std::pair<int, FileMetaData>> new_files_;	// 本次操作新增的文件
```

关注最后3个， compacton_pointers暂时不管，**delted_files_, new_files_是这次版本修改的差量。**

**关注new_files_中的FileMetaData，因为一次版本修改新增的文件是这个类的集合，**

## 2. FileMetaData

FileMetaData是每个Version内部维持的文件，每层中都有多个FileMetaData， 一个FileMetaData用于描述一个sstable的相关信息，是sstable的元数据。

```c++
struct FileMetaData {
  FileMetaData() : refs(0), allowed_seeks(1 << 30), file_size(0) {}

  int refs;
  int allowed_seeks;  // Seeks allowed until compaction，用于基于seek compaction
  uint64_t number;		// 这个number用来唯一标识一个sstable，还记得文件命名中的编号吗？
  uint64_t file_size;    // File size in bytes 文件大小
  InternalKey smallest;  // Smallest internal key served by table 最小key
  InternalKey largest;   // Largest internal key served by table 最大key
};
```

各字段都在注释中解释，我们可以得到一个信息，在SSTable中存放的是key是InternalKey， 对应了 [MemTable](https://www.ravenxrz.ink/archives/aba77258.html) 文章中说的Key类型。

## 3. Version结构

来看看Version的成员：

```c++
 private:
 ...

 // 这里看出leveldb在系统中维护的version组成一个链表，且系统中可能存在多个VersionSet。每个Set维护一（多）组Version
  VersionSet* vset_;  // VersionSet to which this Version belongs
  Version* next_;     // Next version in linked list
  Version* prev_;     // Previous version in linked list
  int refs_;          // Number of live refs to this version  // 引用计数，估计和回收Version相关

  // 每层的files, 每个file都是FileMetadata
  // List of files per level
  std::vector<FileMetaData*> files_[config::kNumLevels];

  // Next file to compact based on seek stats.
  FileMetaData* file_to_compact_;		// compaction过程会用到
  int file_to_compact_level_;

  // compaction相关，根据compactoin_score_决定是否需要compaction
  // Level that should be compacted next and its compaction score.
  // Score < 1 means compaction is not strictly needed.  These fields
  // are initialized by Finalize().
  double compaction_score_;	
  int compaction_level_;
```

next_和prev_指针，表明version之间组成一个双链表。

files_表示了这个版本下整个系统的所有sstable的元数据。

下面是和compaction相关的结构。

## 4. VersionSet结构

```c++
private:
class Builder;

friend class Compaction;
friend class Version;

bool ReuseManifest(const std::string& dscname, const std::string& dscbase);

void Finalize(Version* v);

void GetRange(const std::vector<FileMetaData*>& inputs, InternalKey* smallest,
InternalKey* largest);

void GetRange2(const std::vector<FileMetaData*>& inputs1,
const std::vector<FileMetaData*>& inputs2,
InternalKey* smallest, InternalKey* largest);

void SetupOtherInputs(Compaction* c);

// Save current contents to *log
Status WriteSnapshot(log::Writer* log);

void AppendVersion(Version* v);

Env* const env_;
const std::string dbname_;
const Options* const options_;
TableCache* const table_cache_;		// cache相关
const InternalKeyComparator icmp_;	// 比较器
uint64_t next_file_number_;			// !!!前面文件编号文章中重点想说的就是这个变量
uint64_t manifest_file_number_;		// 当前manifest文件
uint64_t last_sequence_;			// 这个序列号是用来表示Internal key中的序列号
uint64_t log_number_;				// log文件序列号
uint64_t prev_log_number_;  // 0 or backing store for memtable being compacted

// Opened lazily
WritableFile* descriptor_file_;
log::Writer* descriptor_log_;
Version dummy_versions_;  // Head of circular doubly-linked list of versions.	链表head
Version* current_;        // == dummy_versions_.prev_							 当前version

// 下一次compaction时，每层compaction的开始key
// Per-level key at which the next compaction at that level should start.
// Either an empty string, or a valid InternalKey.
std::string compact_pointer_[config::kNumLevels];
```

VersionSet维护了所有有效的version，内部采用双链表的结构来维护。

下图展示了这几个数据结构之间的关系，引用自：https://bean-li.github.io/leveldb-version/

![img](https://bean-li.github.io/assets/LevelDB/version_versionset.png)



一个version维护整个系统中的所有sstable文件的元数据，versionset维护多个version，显然我们不可能无限增加version个数。那如何清理version？

LevelDB会触发Compaction，能对一些文件进行清理操作，让数据更加有序，清理后的数据放到新的版本里面，而老的数据作为原始的素材，最终是要清理掉的，但是如果有读事务位于旧的文件，那么暂时就不能删除。因此利用引用计数，只要一个Verison还活着，就不允许删除该Verison管理的所有文件。当一个Version生命周期结束，它管理的所有文件的引用计数减1.

当一个version被销毁时，每个和它想关联的file的引用计数都会-1，当引用计数小于=0时，file被删除：

```c++
Version::~Version() {
  assert(refs_ == 0);

  // Remove from linked list
  prev_->next_ = next_;
  next_->prev_ = prev_;

  // Drop references to files
  for (int level = 0; level < config::kNumLevels; level++) {
    for (size_t i = 0; i < files_[level].size(); i++) {
      FileMetaData* f = files_[level][i];
      assert(f->refs > 0);
      f->refs--;
      if (f->refs <= 0) {
        delete f;
      }
    }
  }
}
```

# 2. 如何应用VersionEdit+Version = New Version

前面说到， Version + VersionEdit = new Version，如何应用这个增量呢？

具体的操作是在VersionSet中的Builder中的。

首先可以看到，Builder是在 LogAndApply和Recover中被调用的：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200917210900575.png" alt="image-20200917210900575" style="zoom:50%;" />

重点看一下LogAndApply

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200917210931871.png" alt="image-20200917210931871" style="zoom:50%;" />

可以看到，一共有4个函数调用了LogAndApply，DB打开时，其余3个都是和Compaction相关。

## 1. VersionEdit记录一次操作中涉及的文件过程

> 因为和Compaction过程相关，所以初次看不懂没关系，只用记住VersionEdit是用来保存一次操作过程涉及的文件的差量的。

说LogAndApply之前，先说一下VersionEdit是如何保存一次操作过程中的涉及的文件的：

```c++
  private:
  friend class VersionSet;

  typedef std::set<std::pair<int, uint64_t>> DeletedFileSet; // level和filenumber的pair
 DeletedFileSet deleted_files_;
  std::vector<std::pair<int, FileMetaData>> new_files_;	// level和filemetadata
```

先看==**deleted_files_**:==

只在RemoveFile函数调用中，增加。

```c++
  // Delete the specified "file" from the specified "level".
  void RemoveFile(int level, uint64_t file) {
    deleted_files_.insert(std::make_pair(level, file));
  }
```

RemoveFile函数调用，有两个函数caller:

![image-20201006194127291](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201006194127291.png)

AddInputDeletions， 在常规compaction中，加入所有选中了的compaction文件。

```c++
void Compaction::AddInputDeletions(VersionEdit* edit) {
  for (int which = 0; which < 2; which++) {
    for (size_t i = 0; i < inputs_[which].size(); i++) {
        // !!!
      edit->RemoveFile(level_ + which, inputs_[which][i]->number);
    }
  }
}

```

BackgroundCompaction，只看上层与下层不重叠的情况。

```c++
 Status status;
  if (c == nullptr) {
    // Nothing to do
  } else if (!is_manual && c->IsTrivialMove()) {
    // Move file to next level
    assert(c->num_input_files(0) == 1);
    FileMetaData* f = c->input(0, 0);
      // !!!!
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
  }
```

再看==**new_files_**==

new_files_在==**AddFile**==中增加：‘

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

AddFile被下图中的3各函数call：

![image-20201006194516231](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201006194516231.png)

WriteLevel0Table中是memtable dump 到level0所生成的新文件。

BackgroundCompaction是上层与下层不重叠的情况，紧接着是InstallCompactionResults：

InstallCompactionResults：

```c++
Status DBImpl::InstallCompactionResults(CompactionState* compact) {
  // Add compaction outputs
  compact->compaction->AddInputDeletions(compact->compaction->edit());
  const int level = compact->compaction->level();
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    const CompactionState::Output& out = compact->outputs[i];
     // 在level+1中生成新文件的结果
    compact->compaction->edit()->AddFile(level + 1, out.number, out.file_size,
                                         out.smallest, out.largest);
  }
  return versions_->LogAndApply(compact->compaction->edit(), &mutex_);
}
```

## 2. LogAndApply

有了VersionEdit，自然就要分析下如何应用这个Edit了。

看一下LogAndApply的工作：

```c++
Status VersionSet::LogAndApply(VersionEdit* edit, port::Mutex* mu) {
    // 1. 为edit绑定log_number
  if (edit->has_log_number_) {
    assert(edit->log_number_ >= log_number_);
    assert(edit->log_number_ < next_file_number_);
  } else {
    edit->SetLogNumber(log_number_);
  }

  if (!edit->has_prev_log_number_) {
    edit->SetPrevLogNumber(prev_log_number_);
  }

  edit->SetNextFile(next_file_number_);
  edit->SetLastSequence(last_sequence_);

    // !!!应用edit到当前版本
  Version* v = new Version(this);
  {
    Builder builder(this, current_);
    builder.Apply(edit);
    builder.SaveTo(v);
  }
    
   // 通过本次version，计算下一次compaction相关变量，（compaction level和compaction score)
  Finalize(v);

    // 下面和Manifest文件相关
  // Initialize new descriptor log file if necessary by creating
  // a temporary file that contains a snapshot of the current version.
  std::string new_manifest_file;
  Status s;
  if (descriptor_log_ == nullptr) {
    // No reason to unlock *mu here since we only hit this path in the
    // first call to LogAndApply (when opening the database).
    assert(descriptor_file_ == nullptr);
     // new_manifest_file为当前manifest文件路径
    new_manifest_file = DescriptorFileName(dbname_, manifest_file_number_);
    edit->SetNextFile(next_file_number_);
     // 创建文件
    s = env_->NewWritableFile(new_manifest_file, &descriptor_file_);
    if (s.ok()) {
        // 创建manifest写者
      descriptor_log_ = new log::Writer(descriptor_file_);
      s = WriteSnapshot(descriptor_log_);
    }
  }

  // Unlock during expensive MANIFEST log write
  {
    mu->Unlock();

    // Write new record to MANIFEST log
    if (s.ok()) {
      std::string record;
        // edit内容编码到record !!, 可以看一下是怎么编码的，这样就知道manifest文件中保存的内容布局了
      edit->EncodeTo(&record);
       // 写入manifest文件
      s = descriptor_log_->AddRecord(record);
      if (s.ok()) {
          // 刷新到设备上
        s = descriptor_file_->Sync();
      }
      if (!s.ok()) {
        Log(options_->info_log, "MANIFEST write: %s\n", s.ToString().c_str());
      }
    }

     // 更新Current指针
    // If we just created a new descriptor file, install it by writing a
    // new CURRENT file that points to it.
    if (s.ok() && !new_manifest_file.empty()) {
      s = SetCurrentFile(env_, dbname_, manifest_file_number_);
    }

    mu->Lock();
  }

   // 插入当前version到VersionSet中
  // Install the new version
  if (s.ok()) {
     // 插入version，更新current
    AppendVersion(v);
    log_number_ = edit->log_number_;
    prev_log_number_ = edit->prev_log_number_;
  } else {
    delete v;
    if (!new_manifest_file.empty()) {
      delete descriptor_log_;
      delete descriptor_file_;
      descriptor_log_ = nullptr;
      descriptor_file_ = nullptr;
      env_->RemoveFile(new_manifest_file);
    }
  }

  return s;
}
```

工作：

1. 执行 old version + version_edit = new version
2. 编码versionedit，并将其写入到manifest中，同时修改 CURRENT 指针
3. 将新生成的version插入到versionset中

current_版本的更替时机一定要注意到，LogAndApply生成新版本之后，同时将VersionEdit记录到MANIFEST文件之后，不然系统一旦crash，虽然数据不会丢失，但是系统后续启动时的性能会降低。（这段话不理解也没关系，等待后面对会专门出文章分析系统是如何启动的）。

### 1.Builder

接下来需要重点分析下面三行代码：

```c++
    // !!!应用edit到当前版本
  Version* v = new Version(this);
  {
    Builder builder(this, current_);
    builder.Apply(edit);
    builder.SaveTo(v);
  }
```

所以builder自然是重点分析对象：

builder的内部数据域：

```c++
  typedef std::set<FileMetaData*, BySmallestKey> FileSet;
  struct LevelState {
    std::set<uint64_t> deleted_files;
    FileSet* added_files;
  };

  VersionSet* vset_;
  Version* base_;
  LevelState levels_[config::kNumLevels];		// 本次操作造成每层的数据变化
```

#### 1.构造函数

==Builder的构造函数：==

```c++

 // 构造函数
  // Initialize a builder with the files from *base and other info from *vset
  Builder(VersionSet* vset, Version* base) : vset_(vset), base_(base) {
    base_->Ref();
    BySmallestKey cmp;
    cmp.internal_comparator = &vset_->icmp_;
    for (int level = 0; level < config::kNumLevels; level++) {
      levels_[level].added_files = new FileSet(cmp);
    }
  }
```

这里只是完成了一些初始化工作。 这里使用了一个BySmallestKey比较器，简单看下实现：

```c++
  // Helper to sort by v->files_[file_number].smallest
  struct BySmallestKey {
    const InternalKeyComparator* internal_comparator;

    bool operator()(FileMetaData* f1, FileMetaData* f2) const {
      int r = internal_comparator->Compare(f1->smallest, f2->smallest);	// 按照smallest key比较
      if (r != 0) {
        return (r < 0);		// 按照最小key升序
      } else {
        // Break ties by file number
        return (f1->number < f2->number);	// 按照文件序列升序，由于文件序列号越小，文件越旧，所以这里是按照文件从旧到新
      }
    }
  };
```

#### 2. Apply

**将edit中的更改保存在builder中。**

```c++
  // Apply all of the edits in *edit to the current state.
  void Apply(VersionEdit* edit) {
    // Update compaction pointers
    for (size_t i = 0; i < edit->compact_pointers_.size(); i++) {
      const int level = edit->compact_pointers_[i].first;	// first为 level
      vset_->compact_pointer_[level] =
          edit->compact_pointers_[i].second.Encode().ToString(); // second 为 这一level开始compaction的key
    }

    // Delete files 删除文件保存在builder中
    for (const auto& deleted_file_set_kvp : edit->deleted_files_) {
      const int level = deleted_file_set_kvp.first;
      const uint64_t number = deleted_file_set_kvp.second;
      levels_[level].deleted_files.insert(number);			// delete 的 file用number表示
    }

    // Add new files
    for (size_t i = 0; i < edit->new_files_.size(); i++) {
      const int level = edit->new_files_[i].first;
      FileMetaData* f = new FileMetaData(edit->new_files_[i].second);
      f->refs = 1;
	
        // 省略compaction相关代码
       ...

       
      levels_[level].deleted_files.erase(f->number);		// 这里非常奇怪，将再SaveTo函数里面谈一点个人的理解
      levels_[level].added_files->insert(f);
    }
  }
```

代码注释说明了逻辑：

1. 更新compaction指针
2. 保存操作中删除了的文件
3. 保存操作中新增的文件

**注意，added_files添加完元素后，内部是按照smallest key排序（如果smallest key相同则按照文件旧->新排序）的，这一点很重要，不然不好理解下面的SaveTo函数**

#### 3. SaveTo

builder中经过Apply已经保存了这次操作的增量，通过SaveTo将这个增量融合到Version中去。

```c++
// Save the current state in *v.
  void SaveTo(Version* v) {
    BySmallestKey cmp;		// 按照smallestkey比较，如果key相同，按照file number比较。
    cmp.internal_comparator = &vset_->icmp_;
    for (int level = 0; level < config::kNumLevels; level++) {		// 一层层的合并
      // Merge the set of added files with the set of pre-existing files.
      // Drop any deleted files.  Store the result in *v.
      const std::vector<FileMetaData*>& base_files = base_->files_[level];
      std::vector<FileMetaData*>::const_iterator base_iter = base_files.begin();
      std::vector<FileMetaData*>::const_iterator base_end = base_files.end();
      const FileSet* added_files = levels_[level].added_files;
      v->files_[level].reserve(base_files.size() + added_files->size());
      // 小于added_files的key 的 当前版本中的文件，全部加入新版本中
      for (const auto& added_file : *added_files) {
        // Add all smaller files listed in base_

        for (std::vector<FileMetaData*>::const_iterator bpos =
                 std::upper_bound(base_iter, base_end, added_file, cmp);	
             base_iter != bpos; ++base_iter) {
		
          MaybeAddFile(v, level, *base_iter);
        }
		 // 加入 added_file
        MaybeAddFile(v, level, added_file);
      }
	
      // 剩余文件整合
      // Add remaining base files
      for (; base_iter != base_end; ++base_iter) {
        MaybeAddFile(v, level, *base_iter);
      }

       // 下面是检查level>0是否有overlap
#ifndef NDEBUG
      // Make sure there is no overlap in levels > 0
      if (level > 0) {
        for (uint32_t i = 1; i < v->files_[level].size(); i++) {
          const InternalKey& prev_end = v->files_[level][i - 1]->largest;
          const InternalKey& this_begin = v->files_[level][i]->smallest;
          if (vset_->icmp_.Compare(prev_end, this_begin) >= 0) {
            std::fprintf(stderr, "overlapping ranges in same level %s vs. %s\n",
                         prev_end.DebugString().c_str(),
                         this_begin.DebugString().c_str());
            std::abort();
          }
        }
      }
#endif
    }
  }
```

base_存放的是当前系统版本，我们的目标是使用当前versionedit+base\_得到一个新的version。这里的工作就是将builder中的文件+base\_中的文件融合后加入到version中。核心逻辑如下：

```c++
 for (std::vector<FileMetaData*>::const_iterator bpos =
                 std::upper_bound(base_iter, base_end, added_file, cmp);	
             base_iter != bpos; ++base_iter) {
		
          MaybeAddFile(v, level, *base_iter);
        }
		 // 加入 added_file
        MaybeAddFile(v, level, added_file);
      }
```

引用Api文档中对std中的upper_bound的作用的解释：

> 函数签名：
>
> template< class ForwardIt, class T, class Compare >
> ForwardIt upper_bound( ForwardIt first, ForwardIt last, const T& value, Compare comp );
>
> 解释：
>
> Returns an iterator pointing to the first element in the range `[first, last)` that is *greater* than `value`, or `last` if no such element is found.

具体例子，假设有`[1,3,7,8]`, 现在value设置为5， 则upper_bound返回的iterator指向 7.

有了对upper_bound的理解+ added_files内部是有序的前提，就不难理解这个循环了。画个图：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb源码阅读-copy-第 29 页.png" style="zoom:33%;" />



第一次循环：added_file=2，bpos指向7，所以base_iter只能添加1，接着添加added_file=2.

第二次循环，added_file=3,bpos指向7，因为base_iter已经指向7，所以不从base\_添加，只加入3

第三次循环, added_file=11, bpos指向15，从base\_中添加7和10，再添加added_file=11

第四次循环，added_file=16，bpos指向`last`，从base\_中添加15，再添加added_file=16

最终可得如图所示结果。

不过这一切都建立在added_files中的所有文件都会添加到最终version，但实际上却不一定，所以用了`MaybeAddFile` 可能添加File：

```c++
 void MaybeAddFile(Version* v, int level, FileMetaData* f) {
    if (levels_[level].deleted_files.count(f->number) > 0) {
      // File is deleted: do nothing  // 在删除列表中，文件将被删除，不用添加
    } else {	// 
      std::vector<FileMetaData*>* files = &v->files_[level];
      if (level > 0 && !files->empty()) {	// 考虑level>0, 要求key不能overlap
        // Must not overlap
        assert(vset_->icmp_.Compare((*files)[files->size() - 1]->largest,
                                    f->smallest) < 0);
      }
      f->refs++;	//  当前新版本v 对file有引用，所以refs++
      files->push_back(f);	 // 实际压入到新版本
    }	
  }

```

通过 deleted_files来判定是否需要添加这个文件,如果deleted_files中没有该file number，则可以添加，否则不能添加。现在联系前面留下的问题：

```c++
      levels_[level].deleted_files.erase(f->number);		// 这里非常奇怪，将再SaveTo函数里面谈一点个人的理解
      levels_[level].added_files->insert(f);
```

这两行代码会觉得非常奇怪，为什么前期加上了要删除的文件，这里却又要删除了？以下只是我个人的看法，**经过我个人的测试，正常情况和系统crash的情况下，` levels_[level].deleted_files.erase(f->number);`这句话是完全不起作用的，永远不会擦除到元素。那是否这行代码就没用？我也不知道，希望后来人能告诉我。**

目前我将这行代码改为了：

```c++
size_t n = levels_[level].deleted_files.erase(f->number);
assert(n==0);
levels_[level].added_files->insert(f);
```

但还没出现过assert失败过。后期测试中一旦出现，再补上。

### 2. Finalize

生成新版本后，需要更新这个新版本的compaction辅助变量，用于下次compaction，这个工作由Finalize函数完成。这里只用知道这个有这个功能即可，具体再Compaction章节再说。

```c++
void VersionSet::Finalize(Version* v) {
  // Precomputed best level for next compaction
  int best_level = -1;
  double best_score = -1;

  for (int level = 0; level < config::kNumLevels - 1; level++) {
    double score;
    if (level == 0) {
      // level0 单独处理，文件数量 超过kL0_CompactionTrigger时，就trigger compaction
        
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
              static_cast<double>(config::kL0_CompactionTrigger);	// static const int kL0_CompactionTrigger = 4;
    } else {
        // 其余level 用文件size来比较
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

level0:  根据文件数量决定。

其余：根据该层的文件大小决定。

## 3. Manifest

### 1. 向manifest中""写入VersionEdit"

在VersionSet::LogAndApply中的后半段就是与Manifest相关的内容。

```c++
    // 下面和Manifest文件相关
  // Initialize new descriptor log file if necessary by creating
  // a temporary file that contains a snapshot of the current version.
  std::string new_manifest_file;
  Status s;
	// 创建manifest writer
  if (descriptor_log_ == nullptr) {	// 首次进入，创建manifest writer
    // No reason to unlock *mu here since we only hit this path in the
    // first call to LogAndApply (when opening the database).
    assert(descriptor_file_ == nullptr);
     // new_manifest_file为当前manifest文件路径
    new_manifest_file = DescriptorFileName(dbname_, manifest_file_number_);
    edit->SetNextFile(next_file_number_);
     // 创建文件
    s = env_->NewWritableFile(new_manifest_file, &descriptor_file_);
    if (s.ok()) {
        // 创建manifest写者，从本质上来看，manifest和log文件的布局完全相同
      descriptor_log_ = new log::Writer(descriptor_file_);
        // 保存一次当前系统的快照内容到manifest中
      s = WriteSnapshot(descriptor_log_);
    }
  }

  // Unlock during expensive MANIFEST log write
  {
    mu->Unlock();
	
     // 将当前edit内容编码成一条recored，添加到manifest中
    // Write new record to MANIFEST log
    if (s.ok()) {
      std::string record;
        // edit内容编码到record !!, 可以看一下是怎么编码的，这样就知道manifest文件中保存的内容布局了
      edit->EncodeTo(&record);
       // 写入manifest文件
      s = descriptor_log_->AddRecord(record);
      if (s.ok()) {
          // 刷新到设备上
        s = descriptor_file_->Sync();
      }
      if (!s.ok()) {
        Log(options_->info_log, "MANIFEST write: %s\n", s.ToString().c_str());
      }
    }

     // 更新Current指针
    // If we just created a new descriptor file, install it by writing a
    // new CURRENT file that points to it.
    if (s.ok() && !new_manifest_file.empty()) {
      s = SetCurrentFile(env_, dbname_, manifest_file_number_);
    }

    mu->Lock();
  }
```

工作:

1. 如果是首次进入，此时还没有manifest的writer，则新建一个writer，并保存一次当前系统版本的快照到manifest中

   ```c++
    descriptor_log_ = new log::Writer(descriptor_file_);
   ```

   根据这行代码来看，manifest和log文件的布局是完全相同的。

2. 将本次edit中的内容转换成一条记录并Add到manifest中。

### 2. manifest中保存的内容

那manifest中到底存放了些什么？这就需要看看`EncodeTo`函数了：

```c++
void VersionEdit::EncodeTo(std::string* dst) const {
  if (has_comparator_) {
    PutVarint32(dst, kComparator);
    PutLengthPrefixedSlice(dst, comparator_);
  }
  if (has_log_number_) {
    PutVarint32(dst, kLogNumber);
    PutVarint64(dst, log_number_);
  }
  if (has_prev_log_number_) {
    PutVarint32(dst, kPrevLogNumber);
    PutVarint64(dst, prev_log_number_);
  }
  if (has_next_file_number_) {
    PutVarint32(dst, kNextFileNumber);
    PutVarint64(dst, next_file_number_);
  }
  if (has_last_sequence_) {
    PutVarint32(dst, kLastSequence);
    PutVarint64(dst, last_sequence_);
  }

  for (size_t i = 0; i < compact_pointers_.size(); i++) {
    PutVarint32(dst, kCompactPointer);
    PutVarint32(dst, compact_pointers_[i].first);  // level
    PutLengthPrefixedSlice(dst, compact_pointers_[i].second.Encode());
  }

  for (const auto& deleted_file_kvp : deleted_files_) {
    PutVarint32(dst, kDeletedFile);
    PutVarint32(dst, deleted_file_kvp.first);   // level
    PutVarint64(dst, deleted_file_kvp.second);  // file number
  }

  for (size_t i = 0; i < new_files_.size(); i++) {
    const FileMetaData& f = new_files_[i].second;
    PutVarint32(dst, kNewFile);
    PutVarint32(dst, new_files_[i].first);  // level
    PutVarint64(dst, f.number);
    PutVarint64(dst, f.file_size);
    PutLengthPrefixedSlice(dst, f.smallest.Encode());
    PutLengthPrefixedSlice(dst, f.largest.Encode());
  }
}

```

也就是保存这些字段：

```c++
// Tag numbers for serialized VersionEdit.  These numbers are written to
// disk and should not be changed.
enum Tag {
  kComparator = 1,
  kLogNumber = 2,
  kNextFileNumber = 3,
  kLastSequence = 4,
  kCompactPointer = 5,
  kDeletedFile = 6,
  kNewFile = 7,
  // 8 was used for large value refs
  kPrevLogNumber = 9
};

// 对应VersionEdit中的成员
std::string comparator_;
uint64_t log_number_;
uint64_t prev_log_number_;
uint64_t next_file_number_;
SequenceNumber last_sequence_;
bool has_comparator_;
bool has_log_number_;
bool has_prev_log_number_;
bool has_next_file_number_;
bool has_last_sequence_;

std::vector<std::pair<int, InternalKey>> compact_pointers_;
DeletedFileSet deleted_files_;
std::vector<std::pair<int, FileMetaData>> new_files_;
```

如下图：

![img](https://bean-li.github.io/assets/LevelDB/write_a_manifest.png)



# 3. 总结

本文中，我们介绍了leveldb中的重要数据结构，Version，VersionEdit和VersionSet。特别是VersionEdit，它表示了一次leveldb操作过程产生的文件增删的差量。 详细剖析了如何从一个旧版本，通过应用VersionEdit，产生一个新版本。最后，我们还讲解了Manifest文件，它是用来表述sstable的元数据的文件，其内容来自VersionEdit。