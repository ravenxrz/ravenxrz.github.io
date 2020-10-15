---
title: leveldb源码阅读记录-删除数据库对象操作
categories: leveldb
abbrlink: f6637d5f
date: 2020-10-12 19:02:59
tags:
---

当我们不再使用leveldb时，会将 db 这个对象删除。 那在db的析构函数中会做哪些工作？

```c++
leveldb::DB *db;
 ...
delete db;
```

<!--more-->

## 1. db的析构函数

```c++
DBImpl::~DBImpl() {
  // Wait for background work to finish.
  // 等待后台压缩工作结束
  mutex_.Lock();
  shutting_down_.store(true, std::memory_order_release);
  while (background_compaction_scheduled_) {
    background_work_finished_signal_.Wait();
  }
  mutex_.Unlock();

    // db_lock_是为了防止多个进程使用同一个leveldb而设力的lock， 在数据库目录下的 "LOCK"文件就是干这活的
    // 现在本进程已经不使用leveldb了，需要释放lock，让其它进程可以使用leveldb。
  if (db_lock_ != nullptr) {
    env_->UnlockFile(db_lock_);
  }

    // 删除版本
  delete versions_;
    // 解引用memtable
  if (mem_ != nullptr) mem_->Unref();
  if (imm_ != nullptr) imm_->Unref();
    
    // 释放其它资源
  delete tmp_batch_;
  delete log_;
  delete logfile_;
  delete table_cache_;

  if (owns_info_log_) {
    delete options_.info_log;
  }
  if (owns_cache_) {
    delete options_.block_cache;
  }
}
```

基本上这个函数都是在做资源释放，包括：

1. 等待compaction结束，强制开启`shutting_down_`， 后序compaction会监测这个变量，一旦它为true，就不会compaction了。也就是说当前的memtable不一定能完全dump到下层中。 但是后序重启Open leveldb时，可以从log文件中恢复。
2. 解文件锁，这个文件锁是用来保证同一时刻只有一个进程可以使用一个leveldb数据库。
3. 释放version_资源
4. 释放memtable
5. 释放其它资源

## 2. versions_析构

```c++
VersionSet::~VersionSet() {
  current_->Unref();
  assert(dummy_versions_.next_ == &dummy_versions_);  // List must be empty 必须保证所有保本都已经释放
    // 释放manifest相关资源
  delete descriptor_log_;
  delete descriptor_file_;
}

```

首先看 current_->Unref():

```c++
void Version::Unref() {
  assert(this != &vset_->dummy_versions_);
  assert(refs_ >= 1);
  --refs_;
  if (refs_ == 0) {
    delete this;
  }
}
```

### 1. version析构

接着是当前版本的析够：

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

第一步从versionset中移除本版本，第二步释放所有 FileMeta 。

## 3. mem析构

mem的析够比较简单了，当引用计数降低到0时，自动delete：

```c++
  // Drop reference count.  Delete if no more references exist.
  void Unref() {
    --refs_;
    assert(refs_ >= 0);
    if (refs_ <= 0) {
      delete this;
    }
  }
```

## 4. 总结

db对象的删除，基本做的工作就是释放各个环节中设涉及到的资源，如version，memtable，cache。