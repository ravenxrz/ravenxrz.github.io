---
title: leveldb源码阅读记录-文件命名规则
categories: leveldb
abbrlink: f38b067b
date: 2020-10-12 09:50:34
tags:
---

本片文章我们说一说leveldb中的文件命名规则。

> 这篇文章虽然放在前置知识，但是内部原理牵涉到leveldb中的Version相关类，所以看不懂没关系，只用知道一个结论：leveldb的所有带编号的文件共用一套编号系统，也就是说任何带编号的文件不可能有重复的编号。如不会出现 000001.ldb和000001.log文件这种情况。

<!--more-->

运行leveldb程序一段时间后，我们会发现在系统给中，存在很多由数字编号的文件：

![image-20201009111936923](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20201009111936923.png)

这篇文章从整体上谈一谈leveldb的文件命令和文件编号的来源。

## 1. 文件命名

在leveldb中和文件命名相关的代码在：

- leveldb/db/filename.cc
- leveldb/db/filename.h

其核心为MakeFileName函数：

```c++
static std::string MakeFileName(const std::string& dbname, uint64_t number,
                                const char* suffix) {
  char buf[100];
  std::snprintf(buf, sizeof(buf), "/%06llu.%s",
                static_cast<unsigned long long>(number), suffix);
  return dbname + buf;
}

```

可以看到文件名时由 dbname + 6位number + 后缀组成的。 比如dbname = testdb, 则一个sstable的文件名可以为： testdb/000132.ldb. (ldb和sst都是sstable的有效后缀)

```c++
std::string TableFileName(const std::string& dbname, uint64_t number) {
  assert(number > 0);
  return MakeFileName(dbname, number, "ldb");
}

std::string SSTTableFileName(const std::string& dbname, uint64_t number) {
  assert(number > 0);
  return MakeFileName(dbname, number, "sst");
}
```

另外，manifest文件没有调用MakeFileName，而是自己单独的函数:

```c++
std::string DescriptorFileName(const std::string& dbname, uint64_t number) {
  assert(number > 0);
  char buf[100];
  std::snprintf(buf, sizeof(buf), "/MANIFEST-%06llu",
                static_cast<unsigned long long>(number));
  return dbname + buf;
}
```

## 2. 文件编号

### 1. sstable, log和manifest的number

dbname和后缀容易理解。那number是在哪儿来的？

实际上在leveldb中所有和编号有关的文件名的编码都有versonset来维护：

如log file的number:

```c++
    uint64_t new_log_number = impl->versions_->NewFileNumber();
    WritableFile* lfile;
    s = options.env->NewWritableFile(LogFileName(dbname, new_log_number),
                                     &lfile);
```

又如sstable的number:

```c++
  file_number = versions_->NewFileNumber();

  // Make the output file
  std::string fname = TableFileName(dbname_, file_number);
```

看看NewFileNumber：

```c++
 uint64_t NewFileNumber() { return next_file_number_++; }
```

所以log和sst文件的编号都是由versionset中的next_file_number_来维护的。注意在versionset的构造函数中，next_file_number被初始化为2:

```c++
VersionSet::VersionSet(const std::string& dbname, const Options* options,
                       TableCache* table_cache,
                       const InternalKeyComparator* cmp)
    : env_(options->env),
      dbname_(dbname),
      options_(options),
      table_cache_(table_cache),
      icmp_(*cmp),
      next_file_number_(2),		// !! 这里被初始化为2的
      manifest_file_number_(0),  // Filled by Recover()
      last_sequence_(0),
      log_number_(0),
      prev_log_number_(0),
      descriptor_file_(nullptr),
      descriptor_log_(nullptr),
      dummy_versions_(this),
      current_(nullptr) {
  AppendVersion(new Version(this));
}
```

#### 文件编号1在哪儿？

那谁在用1呢？`manifest_file_number_`在用。注意到在`versionset`的构造函数中`manifest_file_number_`初始化为0. 如果系统是首次使用，则会在NewDB中重新对`manifest_file_number_`赋值：

```c++
Status DBImpl::NewDB() {
  VersionEdit new_db;
  new_db.SetComparatorName(user_comparator()->Name());
  new_db.SetLogNumber(0);
  new_db.SetNextFile(2);	// !! 记住这里的next file number = 2
  new_db.SetLastSequence(0);

    // !! 这里设置了manifest的number = 1
  const std::string manifest = DescriptorFileName(dbname_, 1);
	...
    
  if (s.ok()) {
      // 设置CURRENT 指向
    // Make "CURRENT" file that points to the new manifest file.
    s = SetCurrentFile(env_, dbname_, 1);
  } else {
    env_->RemoveFile(manifest);
  }
  return s;
}
```

不过一般来说，除非你修改了options，否则你在运行leveldb时，看到的最小的manifest文件应该是，MANIFEST-000002. 这是为什么？这是因为在DBImpl::Open函数中调用NewDB函数后，紧接着调用了Recover.

```c++
Status VersionSet::Recover(bool* save_manifest) {
    // 省略掉的... 中做工作就是中 CURRENT 所指向的manifest文件读取内容， 先前在NewDB中CURRENT指向 manifest000001
	...

  {
  	  ...
      if (edit.has_next_file_number_) {
        next_file = edit.next_file_number_;		// 此时next_file = 2, 因为在NewDB中 next_file_number_ 被设置为2
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

	...
  if (s.ok()) {
    Version* v = new Version(this);
    builder.SaveTo(v);
    // Install recovered version
    Finalize(v);
    AppendVersion(v);
    manifest_file_number_ = next_file;		// ！！ 这里manifest_file_number_ = 2
 	...
  }

  return s;
}
```

从这里就解释了 manifest_file_number_ = 2是如何来的， 那manifest_file_number_ = 1是如何消失的？在DBImpl::Open的下半部分：

```c++
  if (s.ok()) {
    impl->RemoveObsoleteFiles();		// 其中移除了manifest_file_number = 1时的文件
    impl->MaybeScheduleCompaction();
  }
```

不过要意识到，这只是系统第一次使用启动时manifest_file_number的指定过程。 在后序的使用过程中，manifest_file_number的初始化是从`VersionSet::Recover`中开始，这其中 manifest_file_number仍然由next_file_number_指定，另外还要考虑是否Reuse旧manifest文件，如果复用，manifest_file_number将等于旧manifest的number。

#### 总结

**总结起来一句话，sstable, log, manifest的编号都是由versoinset的next_file_number_来指定。**

### 2. VersoinSet中的LogNumber和PrevLogNumber

PrevLogNumber已不再使用。

LogNumber存放的是当前系统有效的log file 的number。用于写log、recover系统时判定是否有newer log没有转化为sstable等。



