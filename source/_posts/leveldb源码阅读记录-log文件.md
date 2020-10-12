---
title: leveldb源码阅读记录-log文件
categories: leveldb
abbrlink: 8ea77a40
date: 2020-10-12 11:02:12
tags:
---

log文件是用来保证写入的持久性的。当用户向系统发出write请求，首先就会将数据写入一份到log文件中，然后再写入到memtable，这是为了防止系统crash而出现数据loss。

<!--more-->

## 1. 结构分析

关于log的结构在levedb源码的 doc/log_format.md中有介绍。

leveldb在内存中的数据结构是memtable，显然memtable是无法保证数据的持久性，因为系统一旦掉电，数据就丢失了，所以leveldb使用了log file来保证数据的持久性。log file具有only append的特点，写入速度快。

我们曾在[整体架构](https://www.ravenxrz.ink/archives/1a545f48.html)中提到过log的结构，这里再说一下：

leveldb存放的是key-value对，因为键值和value值的长度是可变的，因此，每一笔记录都必须有个**length**字段来表明当前记录的长度。 当然了，leveldb为了校验数据的一致性，同时会计算**checksum**，作为记录的一个字段.

还有另外一个字段，即**type**。注意，Log文件是分block存放的，每个block的大小为32KB，这就会存在一个问题，如某个key－value对过大，无法存放在单个block中，可能存放在多个不同的block中,因此引入了另一个字段RecordType, 用于标识当前物理快与逻辑块之间的关系。

RecordType字段主要有以下四种类型：FULL/FIRST/MIDDLE/LAST。

- 如果 一个Record的size在一个block内，则type=FULL

- 否则Record将会跨越多个block，则可能出现上图中的情况，分别对应FIRST/MIDDLE/LAST

下图展示的 log 文件由 3 个 Block 构成，所以从物理布局来讲，一个 log 文件就是由连续的 32K 大小 Block 构成的。

![img](https://img-blog.csdnimg.cn/20190314203727797.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3R1d2VucWkyMDEz,size_16,color_FFFFFF,t_70)

一个Record的逻辑结构如下：

![](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/log-record逻辑结构.jpg)



## 2. 源码分析

与log相关的代码文件存放在：

- db/log_format.h
- db/log_reader.h
- db/log_reader.cc
- db/log_writer.h
- db/log_writer.cc

### 1. leveldb::log命名空间

```c++
namespace leveldb {
namespace log {

enum RecordType {
  // Zero is reserved for preallocated files
  kZeroType = 0,

  kFullType = 1,

  // For fragments
  kFirstType = 2,
  kMiddleType = 3,
  kLastType = 4
};
static const int kMaxRecordType = kLastType;

static const int kBlockSize = 32768;	// 1个block 32kb

// Header is checksum (4 bytes), length (2 bytes), type (1 byte).
static const int kHeaderSize = 4 + 2 + 1;

}  // namespace log
}  // namespace leveldb

```

### 2. 写操作

log wirter类，对log写入的封装：

```c++
namespace leveldb {

class WritableFile;

namespace log {

class Writer {
 public:
  // Create a writer that will append data to "*dest".
  // "*dest" must be initially empty.
  // "*dest" must remain live while this Writer is in use.
  explicit Writer(WritableFile* dest);

  // Create a writer that will append data to "*dest".
  // "*dest" must have initial length "dest_length".
  // "*dest" must remain live while this Writer is in use.
  Writer(WritableFile* dest, uint64_t dest_length);

  Writer(const Writer&) = delete;
  Writer& operator=(const Writer&) = delete;

  ~Writer();

  Status AddRecord(const Slice& slice);

 private:
  Status EmitPhysicalRecord(RecordType type, const char* ptr, size_t length);

  WritableFile* dest_;
  int block_offset_;  // Curr 

    // 预计算的crc
  // crc32c values for all supported record types.  These are
  // pre-computed to reduce the overhead of computing the crc of the
  // record type stored in the header.
  uint32_t type_crc_[kMaxRecordType + 1];
};

}  // namespace log
}  // namespace leveldb
```

#### 核心-AddRecord 函数

```c++
Status Writer::AddRecord(const Slice& slice) {
  const char* ptr = slice.data();
  size_t left = slice.size();	// left表示剩余多少size未写

  // Fragment the record if necessary and emit it.  Note that if slice
  // is empty, we still want to iterate once to emit a single
  // zero-length record
  Status s;
  bool begin = true;
  do {
    const int leftover = kBlockSize - block_offset_;	// 当前32kb的块的剩余量
    assert(leftover >= 0);
    if (leftover < kHeaderSize) {		// 剩余量 < 一个record的header size
      // Switch to a new block			
      if (leftover > 0) {
        // Fill the trailer (literal below relies on kHeaderSize being 7)
        static_assert(kHeaderSize == 7, "");
        dest_->Append(Slice("\x00\x00\x00\x00\x00\x00", leftover));	// 填充本块最后的空间
      }
      block_offset_ = 0;			 // 更换新块
    }

    // Invariant: we never leave < kHeaderSize bytes in a block.
    assert(kBlockSize - block_offset_ - kHeaderSize >= 0);

    const size_t avail = kBlockSize - block_offset_ - kHeaderSize;	// 剩余可用给数据的空间
    const size_t fragment_length = (left < avail) ? left : avail;	

    RecordType type;
    const bool end = (left == fragment_length);
    if (begin && end) {		// 开始和结束都在本块，整条record都可放置
      type = kFullType;
    } else if (begin) {		// 只有开始在，结束不在，说明是第一条
      type = kFirstType;
    } else if (end) {		// 结束在，开始不在，说明是最后一条
      type = kLastType;
    } else {				// 其余就是中间条目
      type = kMiddleType;
    }
	// 将本条Record写入到物理设备上
    s = EmitPhysicalRecord(type, ptr, fragment_length);
    ptr += fragment_length;
    left -= fragment_length;
    begin = false;
  } while (s.ok() && left > 0);
  return s;
}
```

基本流程如下：

![](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/AddRecord函数流程图-1600848090666.jpg)

#### EmitPhysicalRecord函数

```c++
Status Writer::EmitPhysicalRecord(RecordType t, const char* ptr,
                                  size_t length) {
  assert(length <= 0xffff);  // Must fit in two bytes 一个块最多写32kb，即2^15，至少需要15个bit，所以需要2个字节（解释了为什么要用0xffff)
  assert(block_offset_ + kHeaderSize + length <= kBlockSize);

  // Format the header
  char buf[kHeaderSize];
   // 初始化header， 4-5为数据的length， 6 为类型
  buf[4] = static_cast<char>(length & 0xff);
  buf[5] = static_cast<char>(length >> 8);
  buf[6] = static_cast<char>(t);

  // Compute the crc of the record type and the payload.
  uint32_t crc = crc32c::Extend(type_crc_[t], ptr, length);
  crc = crc32c::Mask(crc);  // Adjust for storage
    // crc校验
  EncodeFixed32(buf, crc);	

  // Write the header and the payload
  Status s = dest_->Append(Slice(buf, kHeaderSize));		// 加入heder
  if (s.ok()) {
    s = dest_->Append(Slice(ptr, length));		// 加如实际数据
    if (s.ok()) {
      s = dest_->Flush();
    }
  }
  block_offset_ += kHeaderSize + length;
  return s;
}
```

这个函数相对简单：

1. 检查写入数据的length是否符合条件。满足在2个字节以内。
2. 填充record的元数据， length + type + crc校验码
3. 写入到dest_中。

## 3. 总结

本文介绍了leveldb中用来保证持久性的log文件，重点介绍了它的逻辑布局和物理布局，同时介绍了一次log写入操作是如何执行的。

