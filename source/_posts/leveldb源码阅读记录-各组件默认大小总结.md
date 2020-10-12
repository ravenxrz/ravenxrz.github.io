---
title: leveldb源码阅读记录-各组件默认大小总结
categories: leveldb
abbrlink: 80e35e78
date: 2020-10-12 19:10:28
tags:
---

至此，我们已经基本分析完整个leveldb，本文给出leveldb中的各组件的默认大小。

- log 文件，由一系列的**32kb**物理块组成。

- manifest 文件，目前来看未限制其大小，但是在系统Open时，有一次机会重新**（文件大小超过2M）**开辟一个新的manifest，并移除旧文件。这部分在ReuseManifest函数中有体现：

<!--more-->

```c++
bool VersionSet::ReuseManifest(const std::string& dscname,
                               const std::string& dscbase) {
  ...
  if (!ParseFileName(dscbase, &manifest_number, &manifest_type) ||
      manifest_type != kDescriptorFile ||
      !env_->GetFileSize(dscname, &manifest_size).ok() ||
      // Make new compacted MANIFEST if old one is too big
      manifest_size >= TargetFileSize(options_)) {
    return false;
  }
  ...
 }

// TargetFileSize函数
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

- memtable: 4MB，源自:`leveldb/options.h`

```c++
 // Amount of data to build up in memory (backed by an unsorted log
  // on disk) before converting to a sorted on-disk file.
  //
  // Larger values increase performance, especially during bulk loads.
  // Up to two write buffers may be held in memory at the same time,
  // so you may wish to adjust this parameter to control memory usage.
  // Also, a larger write buffer will result in a longer recovery time
  // the next time the database is opened.
  size_t write_buffer_size = 4 * 1024 * 1024;
```

- sstable：**2MB**，来源：

```c++
static uint64_t MaxFileSizeForLevel(const Options* options, int level) {
  // We could vary per level to reduce number of files?
  return TargetFileSize(options);
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

具体应在，在DoCompactionWork函数内部，有一段代码：

```c++
  // Close output file if it is big enough
      if (compact->builder->FileSize() >=
          compact->compaction->MaxOutputFileSize()) {	// 此时的MaxOutputFileSize（）返回的就是2MB
        status = FinishCompactionOutputFile(compact, input);
        if (!status.ok()) {
          break;
        }
      }
```

- SStable的datablock， 变长，但**最小为4kb**。

- 各level的大小

  - level0，不按大小算，而是按照文件个数算，level0**默认为4个文件**触发一次compaction。其他还有两个配置，1个8个文件，在MakeRoomForWrite函数中使用，一个12，达到12个文件，系统将暂停写入。

    ```
    // Level-0 compaction is started when we hit this many files.
    static const int kL0_CompactionTrigger = 4;
    
    
    // Soft limit on number of level-0 files.  We slow down writes at this point.
    static const int kL0_SlowdownWritesTrigger = 8;
    
    // Maximum number of level-0 files.  We stop writes at this point.
    static const int kL0_StopWritesTrigger = 12;
    ```

  - 其他level， **level1 10M， level 2 100M， 逐层递乘10.**

  