---
title: leveldb源码阅读记录-Open操作
categories: leveldb
abbrlink: 2f5e74df
date: 2020-10-12 19:00:31
tags:
---

这篇文章我们一起来看看Open操作。

<!--more-->

## 1. Open函数

==DB::Open==

```c++
Status DB::Open(const Options& options, const std::string& dbname, DB** dbptr) {
  *dbptr = nullptr;

  DBImpl* impl = new DBImpl(options, dbname);
  impl->mutex_.Lock();
  VersionEdit edit;
   // 恢复阶段
  // Recover handles create_if_missing, error_if_exists
  bool save_manifest = false;
  Status s = impl->Recover(&edit, &save_manifest);
  if (s.ok() && impl->mem_ == nullptr) {
     // 创建log file和memtable
    // Create new log and a corresponding memtable.
    uint64_t new_log_number = impl->versions_->NewFileNumber();
    WritableFile* lfile;
    s = options.env->NewWritableFile(LogFileName(dbname, new_log_number),
                                     &lfile);
    if (s.ok()) {
      edit.SetLogNumber(new_log_number);
      impl->logfile_ = lfile;
      impl->logfile_number_ = new_log_number;
      impl->log_ = new log::Writer(lfile);
      impl->mem_ = new MemTable(impl->internal_comparator_);
      impl->mem_->Ref();
    }
  }
  if (s.ok() && save_manifest) {
      // 应用从recovery过程中生成version edit
    edit.SetPrevLogNumber(0);  // No older logs needed after recovery.
    edit.SetLogNumber(impl->logfile_number_);
    s = impl->versions_->LogAndApply(&edit, &impl->mutex_);
  }
  if (s.ok()) {
      // remove废旧文件
    impl->RemoveObsoleteFiles();
      // 启动压缩线程
    impl->MaybeScheduleCompaction();	
  }
  impl->mutex_.Unlock();
  if (s.ok()) {
    assert(impl->mem_ != nullptr);
    *dbptr = impl;
  } else {
    delete impl;
  }
  return s;
}

```

工作：

1. 执行recover，恢复系统元数据

2. 生成log文件和memtable

3. 如果有edit，则应用version edit到系统中。

   edit中保存的是上次系统crash后，log中保存了一些”loss”掉的数据，在Recover阶段，这些log中的数据，会重新load出来并插入到memtable中，当数据量达到使memtable dump，则需要生成一个sstable及其元数据，edit用来保存这些信息。

4. 移除废旧文件

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb源码阅读-copy-第 26 页 (2).png" style="zoom:33%;" />

我们分别来看看这几部分工作。

## 2. Recover

Recover函数工作主要包含两大块：

1. 上次系统关闭后，元数据信息持久化到了外存（主要是manifest），现在要重新加载出来。
2. 如果上次系统关闭过程中出现了crash，但是log文件中有用来crash 恢复的数据，则还要冲log文件中恢复数据。

```c++
Status DBImpl::Recover(VersionEdit* edit, bool* save_manifest) {
  mutex_.AssertHeld();

  // Ignore error from CreateDir since the creation of the DB is
  // committed only when the descriptor is created, and this directory
  // may already exist from a previous failed creation attempt.
  // 创建数据库目录
  env_->CreateDir(dbname_);
  assert(db_lock_ == nullptr);
  Status s = env_->LockFile(LockFileName(dbname_), &db_lock_);
  if (!s.ok()) {
    return s;
  }

  if (!env_->FileExists(CurrentFileName(dbname_))) {	// 首次运行系统
    if (options_.create_if_missing) {
      s = NewDB();			// 创建一个新DB
      if (!s.ok()) {
        return s;
      }
    } else {
      return Status::InvalidArgument(
          dbname_, "does not exist (create_if_missing is false)");
    }
  } else {
    if (options_.error_if_exists) {
      return Status::InvalidArgument(dbname_,
                                     "exists (error_if_exists is true)");
    }
  }

    // 第一部分
   // 执行Recover
  s = versions_->Recover(save_manifest);
  if (!s.ok()) {
    return s;
  }
  SequenceNumber max_sequence(0);

    
   // 第二部分
   // 从那些未注册的log中还原数据（即系统crash后，从log文件中恢复数据）
    
  // Recover from all newer log files than the ones named in the
  // descriptor (new log files may have been added by the previous
  // incarnation without registering them in the descriptor).
  //
  // Note that PrevLogNumber() is no longer used, but we pay
  // attention to it in case we are recovering a database
  // produced by an older version of leveldb.
  const uint64_t min_log = versions_->LogNumber();
  const uint64_t prev_log = versions_->PrevLogNumber();
  std::vector<std::string> filenames;
  s = env_->GetChildren(dbname_, &filenames);
  if (!s.ok()) {
    return s;
  }
  std::set<uint64_t> expected;
  versions_->AddLiveFiles(&expected);
  uint64_t number;
  FileType type;
  std::vector<uint64_t> logs;
  for (size_t i = 0; i < filenames.size(); i++) {
    if (ParseFileName(filenames[i], &number, &type)) {
      expected.erase(number);
      if (type == kLogFile && ((number >= min_log) || (number == prev_log)))
        logs.push_back(number);
    }
  }
  if (!expected.empty()) {
    char buf[50];
    std::snprintf(buf, sizeof(buf), "%d missing files; e.g.",
                  static_cast<int>(expected.size()));
    return Status::Corruption(buf, TableFileName(dbname_, *(expected.begin())));
  }

  // Recover in the order in which the logs were generated
  std::sort(logs.begin(), logs.end());
  for (size_t i = 0; i < logs.size(); i++) {
    s = RecoverLogFile(logs[i], (i == logs.size() - 1), save_manifest, edit,
                       &max_sequence);
    if (!s.ok()) {
      return s;
    }

    // The previous incarnation may not have written any MANIFEST
    // records after allocating this log number.  So we manually
    // update the file number allocation counter in VersionSet.
    versions_->MarkFileNumberUsed(logs[i]);
  }

  if (versions_->LastSequence() < max_sequence) {
    versions_->SetLastSequence(max_sequence);
  }

  return Status::OK();
}
```

### 1. NewDB

如果系统首次运行，创建一个新DB并做简单初始化即。

```c++
Status DBImpl::NewDB() {
  VersionEdit new_db;
  new_db.SetComparatorName(user_comparator()->Name());
  new_db.SetLogNumber(0);
  new_db.SetNextFile(2);
  new_db.SetLastSequence(0);

  const std::string manifest = DescriptorFileName(dbname_, 1);
  WritableFile* file;
  Status s = env_->NewWritableFile(manifest, &file);
  if (!s.ok()) {
    return s;
  }
  {
    log::Writer log(file);	// 在log中记录本次操作
    std::string record;
    new_db.EncodeTo(&record);
    s = log.AddRecord(record);
    if (s.ok()) {
      s = file->Close();
    }
  }
  delete file;
  if (s.ok()) {		// 生成 "CURRENT" 文件
    // Make "CURRENT" file that points to the new manifest file.
    s = SetCurrentFile(env_, dbname_, 1);
  } else {
    env_->RemoveFile(manifest);
  }
  return s;
}
```

### 2. versions_->Recover(save_manifest)

```c++
Status VersionSet::Recover(bool* save_manifest) {
  struct LogReporter : public log::Reader::Reporter {
    Status* status;
    void Corruption(size_t bytes, const Status& s) override {
      if (this->status->ok()) *this->status = s;
    }
  };

  // Read "CURRENT" file, which contains a pointer to the current manifest file
  std::string current;
   // current保存的是当前的manifest文件名
  Status s = ReadFileToString(env_, CurrentFileName(dbname_), &current);
  if (!s.ok()) {
    return s;
  }
  if (current.empty() || current[current.size() - 1] != '\n') {
    return Status::Corruption("CURRENT file does not end with newline");
  }
  current.resize(current.size() - 1);

    // 获取manifest文件路径
  std::string dscname = dbname_ + "/" + current;
  SequentialFile* file;
  s = env_->NewSequentialFile(dscname, &file);
  if (!s.ok()) {
    if (s.IsNotFound()) {
      return Status::Corruption("CURRENT points to a non-existent file",
                                s.ToString());
    }
    return s;
  }

  bool have_log_number = false;
  bool have_prev_log_number = false;
  bool have_next_file = false;
  bool have_last_sequence = false;
  uint64_t next_file = 0;
  uint64_t last_sequence = 0;
  uint64_t log_number = 0;
  uint64_t prev_log_number = 0;
  Builder builder(this, current_);

  {
    LogReporter reporter;
    reporter.status = &s;
    log::Reader reader(file, &reporter, true /*checksum*/,
                       0 /*initial_offset*/);
    Slice record;
    std::string scratch;
    while (reader.ReadRecord(&record, &scratch) && s.ok()) {	// 读取一条manifest recored
      VersionEdit edit;
      s = edit.DecodeFrom(record);
      if (s.ok()) {
        if (edit.has_comparator_ &&
            edit.comparator_ != icmp_.user_comparator()->Name()) {	// 验证comparator是否被修改了
          s = Status::InvalidArgument(
              edit.comparator_ + " does not match existing comparator ",
              icmp_.user_comparator()->Name());
        }
      }

      if (s.ok()) {		// 将这条record添加到builder中
        builder.Apply(&edit);
      }

       // 其他参数赋值
      if (edit.has_log_number_) {
        log_number = edit.log_number_;
        have_log_number = true;
      }

      if (edit.has_prev_log_number_) {
        prev_log_number = edit.prev_log_number_;
        have_prev_log_number = true;
      }

      if (edit.has_next_file_number_) {
        next_file = edit.next_file_number_;
        have_next_file = true;
      }

      if (edit.has_last_sequence_) {
        last_sequence = edit.last_sequence_;
        have_last_sequence = true;
      }
    }
  }
  delete file;
  file = nullptr;

  if (s.ok()) {
    if (!have_next_file) {
      s = Status::Corruption("no meta-nextfile entry in descriptor");
    } else if (!have_log_number) {
      s = Status::Corruption("no meta-lognumber entry in descriptor");
    } else if (!have_last_sequence) {
      s = Status::Corruption("no last-sequence-number entry in descriptor");
    }

    if (!have_prev_log_number) {
      prev_log_number = 0;
    }

    MarkFileNumberUsed(prev_log_number);
    MarkFileNumberUsed(log_number);
  }

  if (s.ok()) {
     // 生成新version
    Version* v = new Version(this);
      // 将多个version edit增量，应用到这个新version中
    builder.SaveTo(v);
    // Install recovered version
     // Finalize是用于计算一些compaction操作会用到的标志
    Finalize(v);
      // 添加这个version到versionsets中
    AppendVersion(v);
      // 其他参数初始化
    manifest_file_number_ = next_file;
    next_file_number_ = next_file + 1;
    last_sequence_ = last_sequence;
    log_number_ = log_number;
    prev_log_number_ = prev_log_number;

      // 是否需要复用manifest
    // See if we can reuse the existing MANIFEST file.
    if (ReuseManifest(dscname, current)) {	// 看manifest文件的大小是否过大，不算大，则可复用
      // No need to save new manifest
    } else {
      *save_manifest = true;
    }
  }

  return s;
}
```

上面的操作也不算复杂，但需要对version，versionedit，versionset有清晰的概念。流程图如下：

<img src="https://pic.downk.cc/item/5f8438b51cd1bbb86bfce29d.png" style="zoom:33%;" />

额外说一下 == edit.DecodeFrom(record); ==了解这个函数，就能知道MANIFEST中存放的是些什么数据。

```c++
Status VersionEdit::DecodeFrom(const Slice& src) {
  Clear();
  Slice input = src;
  const char* msg = nullptr;
  uint32_t tag;

  // Temporary storage for parsing
  int level;
  uint64_t number;
  FileMetaData f;
  Slice str;
  InternalKey key;

  while (msg == nullptr && GetVarint32(&input, &tag)) {
    switch (tag) {
      case kComparator:
       ...
        break;

      case kLogNumber:
        ...
        break;

      case kPrevLogNumber:
        ...
        break;

      case kNextFileNumber:
        ...
        break;

      case kLastSequence:
       ...
        break;

      case kCompactPointer:
        ...
        break;

      case kDeletedFile:
       ...
        break;

      case kNewFile:
       ...
        break;

      default:
        msg = "unknown tag";
        break;
    }
  }

  if (msg == nullptr && !input.empty()) {
    msg = "invalid tag";
  }

  Status result;
  if (msg != nullptr) {
    result = Status::Corruption("VersionEdit", msg);
  }
  return result;
}
```

中间的swtich是核心，**每个case都说明了MANIFEST中会保存些什么东西。**

执行完 versions_->Recover ，就完成了整个Recover的第一大部分，如果系统未曾出过异常，那到这里，系统已经初始化完成。 但是也可能需要第二部分，如果系统上次异常关机，可能存在数据loss，我们可以从log文件中恢复这些数据。下面看第二部分。

### 3. 从log文件中恢复数据

先给出这部分的流程图：

 <img src="https://pic.downk.cc/item/5f8438c91cd1bbb86bfcf476.png" style="zoom:33%;" />



```c++
// 第二部分
   // 从那些未注册的log中还原数据（即系统crash后，从log文件中恢复数据）
    
  // Recover from all newer log files than the ones named in the
  // descriptor (new log files may have been added by the previous
  // incarnation without registering them in the descriptor).
  //
  // Note that PrevLogNumber() is no longer used, but we pay
  // attention to it in case we are recovering a database
  // produced by an older version of leveldb.
  const uint64_t min_log = versions_->LogNumber();		// 正常系统环境下的最小log号，所有异常环境下（crash）的log号都比这个大
  const uint64_t prev_log = versions_->PrevLogNumber();	
  std::vector<std::string> filenames;
  s = env_->GetChildren(dbname_, &filenames);		// 获取所有数据库文件名
  if (!s.ok()) {
    return s;
  }
  std::set<uint64_t> expected;
  versions_->AddLiveFiles(&expected);		// 加入所有版本中的sstable文件number（到这里还只有一个版本）
  uint64_t number;
  FileType type;
  std::vector<uint64_t> logs;				// logs保存需要执行恢复的log文件名
  for (size_t i = 0; i < filenames.size(); i++) {
    if (ParseFileName(filenames[i], &number, &type)) {
      expected.erase(number);
      if (type == kLogFile && ((number >= min_log) || (number == prev_log)))	
        logs.push_back(number);				// 得到需要执行恢复的log文件
    }
  }
  if (!expected.empty()) {
    char buf[50];
    std::snprintf(buf, sizeof(buf), "%d missing files; e.g.",
                  static_cast<int>(expected.size()));
    return Status::Corruption(buf, TableFileName(dbname_, *(expected.begin())));
  }

  // Recover in the order in which the logs were generated
  std::sort(logs.begin(), logs.end());
  for (size_t i = 0; i < logs.size(); i++) {
    s = RecoverLogFile(logs[i], (i == logs.size() - 1), save_manifest, edit,
                       &max_sequence);		// 正式从log中恢复数据
    if (!s.ok()) {
      return s;
    }

    // The previous incarnation may not have written any MANIFEST
    // records after allocating this log number.  So we manually
    // update the file number allocation counter in VersionSet.
    versions_->MarkFileNumberUsed(logs[i]);
  }

  if (versions_->LastSequence() < max_sequence) {
    versions_->SetLastSequence(max_sequence);
  }
```

上面的代码首先得到需要执行恢复的log文件，然后通过RecoverLogFile函数，从这些文件中而建中恢复数据。

==RecoverLogFile==

```c++
Status DBImpl::RecoverLogFile(uint64_t log_number, bool last_log,
                              bool* save_manifest, VersionEdit* edit,
                              SequenceNumber* max_sequence) {
  ...

  mutex_.AssertHeld();

  // Open the log file
  ...

  // Read all the records and add to a memtable
  std::string scratch;
  Slice record;
  WriteBatch batch;
  int compactions = 0;
  MemTable* mem = nullptr;
  while (reader.ReadRecord(&record, &scratch) && status.ok()) {	// 读取log文件中的每条记录
    if (record.size() < 12) {
      reporter.Corruption(record.size(),
                          Status::Corruption("log record too small"));
      continue;
    }
    WriteBatchInternal::SetContents(&batch, record);

    if (mem == nullptr) {
      mem = new MemTable(internal_comparator_);
      mem->Ref();
    }
      // 插入到memtable中
    status = WriteBatchInternal::InsertInto(&batch, mem);
    MaybeIgnoreError(&status);
    if (!status.ok()) {
      break;
    }
    const SequenceNumber last_seq = WriteBatchInternal::Sequence(&batch) +
                                    WriteBatchInternal::Count(&batch) - 1;
    if (last_seq > *max_sequence) {
      *max_sequence = last_seq;
    }

      // memtable满， 需要执行compaction
    if (mem->ApproximateMemoryUsage() > options_.write_buffer_size) {
      compactions++;
      *save_manifest = true;
      status = WriteLevel0Table(mem, edit, nullptr);
      mem->Unref();
      mem = nullptr;
      if (!status.ok()) {
        // Reflect errors immediately so that conditions like full
        // file-systems cause the DB::Open() to fail.
        break;
      }
    }
  }

  delete file;
  
  ...
  return status;
}
```

和前面从manifest中恢复类似，这里是从log文件中恢复，循环读取log中的数据，将得到的数据直接插入到memtable中。过程中可能伴随compaction。

到这里整个Open函数的Recover部分就分析完了。

## 2.x. 额外-MANIFEST丢失或者损坏，leveldb如何恢复

这部分工作可以看RepairDB类。

> 摘自：https://bean-li.github.io/leveldb-manifest/

如果只有MANIFEST文件损坏，或者干脆误删除，leveldb是可以恢复的。这是结论，事实上这两种实验我都已经做过了。

使用python-leveldb，通过如下手段可以修复Leveldb

```python
import leveldb
ret ＝ leveldb.RepairDB('/data/mon.iecvq/store.db')
```

为什么MANIFEST损坏或者丢失之后，依然可以恢复出来？LevelDB如何做到。

对于LevelDB而言，修复过程如下：

- 首先处理log，这些还未来得及写入的记录，写入新的.sst文件
- 扫描所有的sst文件，生成元数据信息：包括number filesize， 最小key，最大key
- 根据这些元数据信息，将生成新的MANIFEST文件。

第三步如何生成新的MANIFEST？ 因为sstable文件是分level的，但是很不幸，我们无法从名字上判断出来文件属于哪个level。第三步处理的原则是，既然我分不出来，我就认为所有的sstale文件都属于level 0，因为level 0是允许重叠的，因此并没有违法基本的准则。

当修复之后，第一次Open LevelDB的时候，很明显level 0 的文件可能远远超过4个文件，因此会Compaction。 又因为所有的文件都在Level 0 这次Compaction无疑是非常沉重的。它会扫描所有的文件，归并排序，产生出level 1文件，进而产生出其他level的文件。

从上面的处理流程看，如果只有MANIFEST文件丢失，其他文件没有损坏，LevelDB是不会丢失数据的，原因是，LevelDB既然已经无法将所有的数据分到不同的Level，但是数据毕竟没有丢，根据文件的number，完全可以判断出文件的新旧，从而确定不同sstable文件中的重复数据，拿个是最新的。经过一次比较耗时的归并排序，就可以生成最新的levelDB。

上述的方法，从功能的角度看，是正确的，但是效率上不敢恭维。Riak曾经测试过78000个sstable 文件，490G的数据，大家都位于Level 0，归并排序需要花费6 weeks，6周啊，这个耗时让人发疯的。

Riak 1.3 版本做了优化，改变了目录结构，对于google 最初版本的LevelDB，所有的文件都在一个目录下，但是Riak 1.3版本引入了子目录， 将不同level的sst 文件放入不同的子目录：

```c++
sst_0
sst_1
...
sst_6
```

有了这个，重新生成MANIFEST自然就很简单了，同样的78000 sstable文件，Repair过程耗时是分钟级别的。

## 3. 应用edit

经过recover后，edit中保存了从log中恢复的数据（插入到memtable并落盘）的元数据，现在通过LogAndApply函数应用它。

## 4. 移除废旧文件&启动压缩线程

```c++
  if (s.ok()) {
    impl->RemoveObsoleteFiles();
    impl->MaybeScheduleCompaction();
  }
```

经过前面的Recover操作，可能会产生一些不再需要的manifest文件、log文件，经过RemoveObsoleteFiles函数，可将其移除。

```c++
void DBImpl::RemoveObsoleteFiles() {
 ...
  for (std::string& filename : filenames) {
    if (ParseFileName(filename, &number, &type)) {
      bool keep = true;
      switch (type) {
        case kLogFile:	// log file删除
          keep = ((number >= versions_->LogNumber()) ||
                  (number == versions_->PrevLogNumber()));
          break;
        case kDescriptorFile:	// manifest file删除
          // Keep my manifest file, and any newer incarnations'
          // (in case there is a race that allows other incarnations)
          keep = (number >= versions_->ManifestFileNumber());
          break;
        case kTableFile:
          keep = (live.find(number) != live.end());
          break;
        case kTempFile:
          // Any temp files that are currently being written to must
          // be recorded in pending_outputs_, which is inserted into "live"
          keep = (live.find(number) != live.end());
          break;
        case kCurrentFile:
        case kDBLockFile:
        case kInfoLogFile:
          keep = true;
          break;
      }

      if (!keep) {
        files_to_delete.push_back(std::move(filename));	// 添加要删除的文件
        if (type == kTableFile) {	// 从cache中剔除
          table_cache_->Evict(number);
        }
        Log(options_.info_log, "Delete type=%d #%lld\n", static_cast<int>(type),
            static_cast<unsigned long long>(number));
      }
    }
  }

  // While deleting all files unblock other threads. All files being deleted
  // have unique names which will not collide with newly created files and
  // are therefore safe to delete while allowing other threads to proceed.
  mutex_.Unlock();
  for (const std::string& filename : files_to_delete) {
    env_->RemoveFile(dbname_ + "/" + filename);		// 正式删除文件
  }
  mutex_.Lock();
}
```

可以看到，删除的条件基本都是通过file number来判断的。

启动压缩线程：

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
    // No work to be done
  } else {
    background_compaction_scheduled_ = true;
    env_->Schedule(&DBImpl::BGWork, this);
  }
}

void DBImpl::BGWork(void* db) {
  reinterpret_cast<DBImpl*>(db)->BackgroundCall();
}
```

这部分在cmopaction章节有说，这里就不再赘述。

至此，我们已经完成了DB::Open的分析。

## 