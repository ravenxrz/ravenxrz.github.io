---
title: leveldb源码阅读记录-Put操作
categories: leveldb
abbrlink: d753e372
date: 2020-10-12 18:58:59
tags:
---

前面系列的文章，将leveldb整个系统分成了多个模块讲解，从这篇文章开始，我们讲解leveldb的各个对用户提供的API，将前面的各个组件贯穿起来。这篇文章首先讲解**Put操作**。

<!--more-->

```c++
// Convenience methods
Status DBImpl::Put(const WriteOptions& o, const Slice& key, const Slice& val) {
  return DB::Put(o, key, val);
}

```

现在我们先来看看WriteOptions

```c++
// Options that control write operations
struct LEVELDB_EXPORT WriteOptions {
  WriteOptions() = default;

  // If true, the write will be flushed from the operating system
  // buffer cache (by calling WritableFile::Sync()) before the write
  // is considered complete.  If this flag is true, writes will be
  // slower.
  //
  // If this flag is false, and the machine crashes, some recent
  // writes may be lost.  Note that if it is just the process that
  // crashes (i.e., the machine does not reboot), no writes will be
  // lost even if sync==false.
  //
  // In other words, a DB write with sync==false has similar
  // crash semantics as the "write()" system call.  A DB write
  // with sync==true has similar crash semantics to a "write()"
  // system call followed by "fsync()".
  bool sync = false;
};
```

可以看到，WriteOptions就是保存了一个是否sync的bool变量。和linux下的fsync()语义类似。

ok，正式看看Put函数：

```c++
// Default implementations of convenience methods that subclasses of DB
// can call if they wish
Status DB::Put(const WriteOptions& opt, const Slice& key, const Slice& value) {
  WriteBatch batch;
  batch.Put(key, value);
  return Write(opt, &batch);
}

```

可以看到对于Put单个kv的情况，leveldb默认实现也将它封装成一个WriteBatch。





## 1. Write函数

现在走到`Write(opt, &batch);`

```c++
Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates) {
   // 对于一次写，都将其封装成一个Writer
  Writer w(&mutex_);
  w.batch = updates;
  w.sync = options.sync;
  w.done = false;

    // 加入写队列
  MutexLock l(&mutex_);
  writers_.push_back(&w);
  while (!w.done && &w != writers_.front()) {
    w.cv.Wait();
  }
  if (w.done) {
    return w.status;
  }

  // May temporarily unlock and wait.
    // 首先要制作出空余空间来写入
  Status status = MakeRoomForWrite(updates == nullptr);
  uint64_t last_sequence = versions_->LastSequence();
  Writer* last_writer = &w;
  if (status.ok() && updates != nullptr) {  // nullptr batch is for compactions
     // 为batch添加Sequence（这部分已经在DBIter中讲过）
    WriteBatch* write_batch = BuildBatchGroup(&last_writer);
    WriteBatchInternal::SetSequence(write_batch, last_sequence + 1);
      // 更新last_sequence
    last_sequence += WriteBatchInternal::Count(write_batch);

    // !!! 这里是核心
    // Add to log and apply to memtable.  We can release the lock
    // during this phase since &w is currently responsible for logging
    // and protects against concurrent loggers and concurrent writes
    // into mem_.
    {
      mutex_.Unlock();
        // 首先向log_中添加这条记录，关于log_的分析，请看 log文件章节。
      status = log_->AddRecord(WriteBatchInternal::Contents(write_batch));
      bool sync_error = false;
      if (status.ok() && options.sync) {	// 如果配置了sync=true，则log添加record后就立即落盘。如果sync=false，则可能造成数据loss
        status = logfile_->Sync();
        if (!status.ok()) {
          sync_error = true;
        }
      }
      if (status.ok()) {
          // 插入到memtable中，这部分也在DBIter中讲过
        status = WriteBatchInternal::InsertInto(write_batch, mem_);
      }
      mutex_.Lock();
      if (sync_error) {	// 一旦出错，强制认为log的这条record没有加入
        // The state of the log file is indeterminate: the log record we
        // just added may or may not show up when the DB is re-opened.
        // So we force the DB into a mode where all future writes fail.
        RecordBackgroundError(status);
      }
    }
    if (write_batch == tmp_batch_) tmp_batch_->Clear();
	// 更新versions的sequence number
    versions_->SetLastSequence(last_sequence);
  }

  while (true) {
    Writer* ready = writers_.front();
    writers_.pop_front();
    if (ready != &w) {
      ready->status = status;
      ready->done = true;
      ready->cv.Signal();
    }
    if (ready == last_writer) break;
  }

   // 唤醒新的Writer来写
  // Notify new head of write queue
  if (!writers_.empty()) {
    writers_.front()->cv.Signal();
  }

  return status;
}
```

这部分代码虽然比较长，但是也很直观，每次写操作都会被封装成一个Writer，然后加入到写队列中。**所以leveldb的写操作是单线程的**

当某个线程可执行写时，首先执行 MakeRoomForWrite 让系统能够空出空间来写，接着首先向log写，然后向memtable中写。 一切完成后，唤醒另一个线程来写。（唤醒看着有些奇怪，后面再解释）

## 流程图

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/绘图文件-Put流程.png" style="zoom: 33%;" />

## 2.  MakeRoomForWrite

这个函数主要是给系统”腾出空间“，让后序的写可以正常写入。

```c++
// REQUIRES: mutex_ is held
// REQUIRES: this thread is currently at the front of the writer queue
Status DBImpl::MakeRoomForWrite(bool force) {
  mutex_.AssertHeld();
  assert(!writers_.empty());
  bool allow_delay = !force;
  Status s;
  while (true) {
    if (!bg_error_.ok()) {
      // Yield previous error
      s = bg_error_;
      break;
    } else if (allow_delay && versions_->NumLevelFiles(0) >=
                                  config::kL0_SlowdownWritesTrigger) {	// 允许当前写延迟，并且level0的个数达到软限制个数
       // 让写延迟1ms， 休眠期间，让出cpu给compaction thread，并且不与compaction thread竞争。
      // We are getting close to hitting a hard limit on the number of
      // L0 files.  Rather than delaying a single write by several
      // seconds when we hit the hard limit, start delaying each
      // individual write by 1ms to reduce latency variance.  Also,
      // this delay hands over some CPU to the compaction thread in
      // case it is sharing the same core as the writer.
      mutex_.Unlock();
      env_->SleepForMicroseconds(1000);
      allow_delay = false;  // Do not delay a single write more than once
      mutex_.Lock();
    } else if (!force &&
               (mem_->ApproximateMemoryUsage() <= options_.write_buffer_size)) {	// 由足够的空间
      // There is room in current memtable
      break;
    } else if (imm_ != nullptr) {			// memtable满，等待compactoin
      // We have filled up the current memtable, but the previous
      // one is still being compacted, so we wait.
      Log(options_.info_log, "Current memtable full; waiting...\n");
      background_work_finished_signal_.Wait();
    } else if (versions_->NumLevelFiles(0) >= config::kL0_StopWritesTrigger) {	// 达到最大L0数（12），卡死后序线程，直到Compaction完成
      // There are too many level-0 files.
      Log(options_.info_log, "Too many L0 files; waiting...\n");
      background_work_finished_signal_.Wait();
    } else {		//  如果force=true, 则进入到这里，强制将mem_转为imm_, 强制生成新的log_
      // Attempt to switch to a new memtable and trigger compaction of old
      assert(versions_->PrevLogNumber() == 0);
      uint64_t new_log_number = versions_->NewFileNumber();
      WritableFile* lfile = nullptr;
      s = env_->NewWritableFile(LogFileName(dbname_, new_log_number), &lfile);
      if (!s.ok()) {
        // Avoid chewing through file number space in a tight loop.
        versions_->ReuseFileNumber(new_log_number);
        break;
      }
      delete log_;
      delete logfile_;
      logfile_ = lfile;
      logfile_number_ = new_log_number;
      log_ = new log::Writer(lfile);
      imm_ = mem_;
      has_imm_.store(true, std::memory_order_release);
      mem_ = new MemTable(internal_comparator_);
      mem_->Ref();
      force = false;  // Do not force another compaction if have room
      MaybeScheduleCompaction();		// 执行压缩
    }
  }
  return s;
}
```

总体来说，这个函数都是判定某个组件是否满，如mem满？level0 的file个数达到上限？如果达到这些条件，则让本线程wait。直到copmpaction做完，重新唤醒本线程的点：

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
  MaybeScheduleCompaction();
    // 在这里重新唤醒
  background_work_finished_signal_.SignalAll();
}
```

## 3. BuildBatchGroup

BuildBatchGroup的作用是聚合多个写请求到一个Batch，当然聚合后的size是有上限的，不能过大。

```c++
// REQUIRES: Writer list must be non-empty
// REQUIRES: First writer must have a non-null batch
WriteBatch* DBImpl::BuildBatchGroup(Writer** last_writer) {
  mutex_.AssertHeld();
  assert(!writers_.empty());
  Writer* first = writers_.front();
  WriteBatch* result = first->batch;
  assert(result != nullptr);

  size_t size = WriteBatchInternal::ByteSize(first->batch);

   // 最大size初始化
  // Allow the group to grow up to a maximum size, but if the
  // original write is small, limit the growth so we do not slow
  // down the small write too much.
  size_t max_size = 1 << 20;
  if (size <= (128 << 10)) {
    max_size = size + (128 << 10);
  }

    // 遍历所有writer
  *last_writer = first;
  std::deque<Writer*>::iterator iter = writers_.begin();
  ++iter;  // Advance past "first"
  for (; iter != writers_.end(); ++iter) {
    Writer* w = *iter;
    if (w->sync && !first->sync) {
      // Do not include a sync write into a batch handled by a non-sync write.
      break;
    }

    if (w->batch != nullptr) {
      size += WriteBatchInternal::ByteSize(w->batch);
      if (size > max_size) {	// 避免聚合后的batch过大
        // Do not make batch too big
        break;
      }

      // Append to *result
      if (result == first->batch) {	// 切换到临时的batch，避免扰乱原writer中的batch
        // Switch to temporary batch instead of disturbing caller's batch
        result = tmp_batch_;
        assert(WriteBatchInternal::Count(result) == 0);
        WriteBatchInternal::Append(result, first->batch);
      }
        //Append
      WriteBatchInternal::Append(result, w->batch);
    }
      // 设置last_writer指针
    *last_writer = w;
  }
  return result;
}

```

工作比较简单：

1. 初始化聚合后的batch最大大小为max_size
2. 遍历writer，逐渐递增batch
3. 更新last_writer,相当于记录哪些writer中的batch已经被聚合了，后面不用再写这些writer。

## 4. 唤醒

有了对BUildBatchGroup的理解，再看唤醒就很简单了。

```c++
 while (true) {	// 把 front <--> last_write 逐渐的所有writer全部剔除，先设置done=true,再唤醒。
    Writer* ready = writers_.front();
    writers_.pop_front();
    if (ready != &w) {
      ready->status = status;
      ready->done = true;
      ready->cv.Signal();
    }
    if (ready == last_writer) break;
  }

   // 唤醒last_writer后面的新的Writer来写
  // Notify new head of write queue
  if (!writers_.empty()) {
    writers_.front()->cv.Signal();
  }
```

