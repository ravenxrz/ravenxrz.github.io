---
title: leveldb源码阅读记录-SSTable
categories: leveldb
abbrlink: b2082466
date: 2020-10-12 17:00:00
tags:
---

上文中，我们介绍了Version和Manifest，这篇文章将介绍leveldb的核心--SStable。

## SSTable

## 1. 引言

> https://bean-li.github.io/leveldb-sstable/

**这部分在doc/table_format.md有介绍。**

SSTable文件是落在磁盘上的真正文件，leveldb存储路径中的.sst 类型的文件都是SSTable文件。 本文介绍该文件的格式，以及leveldb如何一条记录一条记录的增加到SSTable文件。

<!--more-->

首先要注意，SSTable文件里面存放的是大量的key-value，因为leveldb就是一个key-value DB。我们都买过字典，如果把字典的每一个字当成key，对字的解释当成value，字典就是一个key-value DB。

在收录几千上万个字的字典中，如何快速寻找到茴香的茴字？

字典第一个思想是有序，按照一定的顺序收录，如果无序，杂乱无章地收录key-value就会给检索带来麻烦。

字典的第二个思想是目录，本质是索引，茴读作Hui，因此，在字典中有如下的格式

```
A                  ...............................................................页码
B                  ...............................................................页码
C                  ...............................................................页码
D                  ...............................................................页码
E                  ...............................................................页码
F                  ...............................................................页码
H
|____ a             ...............................................................页码
|____ ..            ...............................................................页码
|____ u
      |____ a       ...............................................................页码
      |____ ..      ...............................................................页码
      |____ i       ...............................................................页码
...
```

这两种思想在leveldb中都有体现，但是leveldb的挑战要大于组织一个字典。首先字典的是死的，一旦字典组织好，字典不会发生变动，既不会增加新的内容，也不会删除某一个字，leveldb则不同，leveldb是动态变化的，你无法预测用户会插入多少笔key-value的记录，用户可能修改某条key-value对，也可能删除某条记录。

另外一个难点是字可以穷尽，但是key-value中的key无法穷举。

## 2. SSTable的layout

```
<beginning_of_file>
[data block 1]
[data block 2]
...
[data block N]
[meta block 1]
...
[meta block K]
[metaindex block]
[index block]
[Footer]        (fixed size; starts at file_size - sizeof(Footer))
<end_of_file>
```

即：

![img](https://bean-li.github.io/assets/LevelDB/sstable_format.png)

首先SSTtable文件不是固定长度的，从上图中也可以看出，文件的内容要能够自解释，就需要有在固定位置有一定数据结构，顺藤摸瓜，理顺文件的内容。

### 1.Footer

对于leveldb而言，Footer是线头，从Footer开始就可以找到该文件的其他组成部分如index block和metaindex block，进而解释整个文件的内容。

Footer的长度是固定的，因此对于SSTable文件的最后 sizeof(Footer)字节就是存放的Footer信息。 Footer固定48B，如下图所示：

![img](https://bean-li.github.io/assets/LevelDB/footer-format.png)

```c++
metaindex_handle: char[p];      // Block handle for metaindex
index_handle:     char[q];      // Block handle for index
padding:          char[40-p-q]; // zeroed bytes to make fixed length
// (40==2*BlockHandle::kMaxEncodedLength)
magic:            fixed64;      // == 0xdb4775248b80fb57 (little-endian)
```

其中最后的magic number是固定的长度的8字节：

```
static const uint64_t kTableMagicNumber = 0xdb4775248b80fb57ull;
```

为了文件的自解释，内部必须要有指针指向文件的其他位置来表示某个section的开始和结束位置。负责记录这个的变量叫做BlockHandle，他有两个成员变量offset_ 和 size_，分别记录的某个数据块的起始位置和长度。

```c++
// BlockHandle is a pointer to the extent of a file that stores a data
// block or a meta block.
class BlockHandle {
	...

 private:
  uint64_t offset_;
  uint64_t size_;
};
```

一个uint64整数经过varint64编码后最大占用10个字节，一个BlockHandle包含两个uint64类型(size和offset)，则一个BlockHandle最多占用20个字节，即BLockHandle::kMaxEncodedLength=20。metaindex_handle和index_handle最大占用字节为40个字节。magic number占用8个字节，是个固定数值，用于读取时校验是否跟填充时相同，不相同的话就表示此文件不是一个SSTable文件(bad magic number)。padding用于补齐为40字节。

sstable文件中footer中可以解码出在文件的结尾处距离footer最近的index block的BlockHandle，以及metaindex block的BlockHandle，从而确定这两个组成部分在文件中的位置。

事实上，在table/table_build.cc中的Status TableBuilder::Finish()函数，我们可以看出，当生成sstable文件的时候，各个组成部分的写入顺序：

### 2.sst layout的源码

```c++
Status TableBuilder::Finish() {
  Rep* r = rep_;
   // 写入data block
  Flush();	// 写入还在buffer中的data block到文件中
  assert(!r->closed);
  r->closed = true;

  BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle;

    // 写入metablock
  // Write filter block
  if (ok() && r->filter_block != nullptr) {
    WriteRawBlock(r->filter_block->Finish(), kNoCompression,
                  &filter_block_handle);
  }

  // Write metaindex block
  if (ok()) {
    BlockBuilder meta_index_block(&r->options);
    if (r->filter_block != nullptr) {
      // Add mapping from "filter.Name" to location of filter data
      std::string key = "filter.";
      key.append(r->options.filter_policy->Name());
      std::string handle_encoding;
      filter_block_handle.EncodeTo(&handle_encoding);
      meta_index_block.Add(key, handle_encoding);
    }

    // TODO(postrelease): Add stats and other meta blocks
    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }

  // Write index block
  if (ok()) {
    if (r->pending_index_entry) {
      r->options.comparator->FindShortSuccessor(&r->last_key);
      std::string handle_encoding;
      r->pending_handle.EncodeTo(&handle_encoding);
      r->index_block.Add(r->last_key, Slice(handle_encoding));
      r->pending_index_entry = false;
    }
    WriteBlock(&r->index_block, &index_block_handle);
  }

  // Write footer
  if (ok()) {
    Footer footer;
    footer.set_metaindex_handle(metaindex_block_handle);
    footer.set_index_handle(index_block_handle);
    std::string footer_encoding;
    footer.EncodeTo(&footer_encoding);
    r->status = r->file->Append(footer_encoding);
    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }
  return r->status;
}
```

这里按顺序写入了几种block:

1. data block
2. meta block
   1. filter block （目前已经实现）
   2. 后面可选实现
3. meta index block
4. inde block
5. footer

## 3. Data Block

data block里面存放的东西很简单，就是一个个的key/value 数据对。

### 1. 向SST中写入一个datablock的过程

#### 处理流程图

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/写入一个datablock的流程-1600853106778.png" style="zoom:25%;" />

通过TableBuilder::Add函数，可以将一个pair加入到TableBuilder中，至于是否写入到sst中，则需要满足一定条件：

==TableBuilder::Add==

```c++
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
	...
  r->data_block.Add(key, value);

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}
```

当当前data_block的size超过options中的block_size阈值时，将当前的block_size写入到sst中（通过Flush函数）。这也说明，**data_block size不是定长的，但一定是超过4k的（应该是会除了最后一个）。**

options.block_size = 4k

```c++
  // Approximate size of user data packed per block.  Note that the
  // block size specified here corresponds to uncompressed data.  The
  // actual size of the unit read from disk may be smaller if
  // compression is enabled.  This parameter can be changed dynamically.
  size_t block_size = 4 * 1024;

```

==TableBuilder::Flush==

```c++
void TableBuilder::Flush() {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->data_block.empty()) return;
    //  r->pending_index_entry is true only if data_block is empty.
  assert(!r->pending_index_entry);
    // 实际写入
  WriteBlock(&r->data_block, &r->pending_handle);
  if (ok()) {
    r->pending_index_entry = true;
      // 写入成功与否状态
    r->status = r->file->Flush();
  }
  if (r->filter_block != nullptr) {
      // 应用 filter（根据源码来看，目前应该只支持bloom filter)
    r->filter_block->StartBlock(r->offset);
  }
}
```

工作:

1. 写入数据块
2. 应用filter

再稍微的深追一下：

在Flush中又会调用==TableBuilder::WriteBlock==，

```c++
void TableBuilder::WriteBlock(BlockBuilder* block, BlockHandle* handle) {
  // File format contains a sequence of blocks where each block has:
  //    block_data: uint8[n]
  //    type: uint8
  //    crc: uint32
  assert(ok());
  Rep* r = rep_;
  Slice raw = block->Finish();

  Slice block_contents;
  CompressionType type = r->options.compression;
  // TODO(postrelease): Support more compression options: zlib?
   // 压缩
  switch (type) {
    case kNoCompression:
      block_contents = raw;
      break;

    case kSnappyCompression: {
      std::string* compressed = &r->compressed_output;
      if (port::Snappy_Compress(raw.data(), raw.size(), compressed) &&
          compressed->size() < raw.size() - (raw.size() / 8u)) {
        block_contents = *compressed;
      } else {
        // Snappy not supported, or compressed less than 12.5%, so just
        // store uncompressed form
        block_contents = raw;
        type = kNoCompression;
      }
      break;
    }
  }

  WriteRawBlock(block_contents, type, handle);
  r->compressed_output.clear();
  block->Reset();
}
```

这里主要是决定了是否对数据进行压缩，然后将处理后的数据交给==WriteRawBlock==处理，同时从这里我们也可以知道，写入一个key value pair，还会附带两个元数据，type和crc。下图给出实际写入一个datablock的数据格式。

#### Record的layout

data N bytes <= 4k(图少写个=)

<img src="../../../图库/datablock格式.png" style="zoom: 33%;" />

最后，调用==TableBuilder::WriteRawBlock==

```c++
void TableBuilder::WriteRawBlock(const Slice& block_contents,
                                 CompressionType type, BlockHandle* handle) {
  Rep* r = rep_;
  handle->set_offset(r->offset);
  handle->set_size(block_contents.size());
    // 写入key value pairs
  r->status = r->file->Append(block_contents);
  if (r->status.ok()) {
    char trailer[kBlockTrailerSize];
    trailer[0] = type;
    uint32_t crc = crc32c::Value(block_contents.data(), block_contents.size());
    crc = crc32c::Extend(crc, trailer, 1);  // Extend crc to cover block type
    EncodeFixed32(trailer + 1, crc32c::Mask(crc));
      // 写入元数据 type + crc
    r->status = r->file->Append(Slice(trailer, kBlockTrailerSize));
    if (r->status.ok()) {
      r->offset += block_contents.size() + kBlockTrailerSize;
    }
  }
}
```

### 2. DataBlock是如何达到阈值，然后才写入到sst的？

回到==TableBuilder::Add==函数中：

```c++
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  ...

  r->data_block.Add(key, value);

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}
```

其中`r->data_block.Add(key, value);`为核心：

#### 前缀压缩机制

由于在一个data block中存在多个key value paris，且它们相互之间是相互有序的，leveldb为了能够减少冗余存储，使用了key的 前缀压缩机制。假设现在要存储两个key， key1 ="the dog", key2 = "the world", 会发现key1和key的前4个字符"the "都是相同的，采用前缀压缩机制，key1继续存储"the dog",key2只用存储"world"即可。

除此之外，leveldb在每16次共享后，会取消当前的前缀共享机制，重新存储当前完整的key。

16次来自：options::block_restart_interval

```c++
  // Number of keys between restart points for delta encoding of keys.
  // This parameter can be changed dynamically.  Most clients should
  // leave this parameter alone.
  int block_restart_interval = 16;
```

#### datablock的layout

所以一个datablock内部是长这样的：

![img](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/data_block_of_sstable.png)

现在来看看`data_block_Add(key,value)`里面到底是怎么做的：

==BlockBuilder::Add==

```c++
void BlockBuilder::Add(const Slice& key, const Slice& value) {
  Slice last_key_piece(last_key_);
  assert(!finished_);
  assert(counter_ <= options_->block_restart_interval);
  assert(buffer_.empty()  // No values yet?
         || options_->comparator->Compare(key, last_key_piece) > 0);
    
  size_t shared = 0;
  if (counter_ < options_->block_restart_interval) {	// 是否达到16次？
      // 找到与上一个key (last_key_piece)之间共享了多少个字节
    // See how much sharing to do with previous string
    const size_t min_length = std::min(last_key_piece.size(), key.size());
    while ((shared < min_length) && (last_key_piece[shared] == key[shared])) {
      shared++;
    }
  } else {
     // 达到block_restart_interval，设置restart_point
    // Restart compression
    restarts_.push_back(buffer_.size());
    counter_ = 0;
  }
    // 剩下的都是非共享的
  const size_t non_shared = key.size() - shared;

  // Add "<shared><non_shared><value_size>" to buffer_
    // 写入元数据
  PutVarint32(&buffer_, shared);
  PutVarint32(&buffer_, non_shared);
  PutVarint32(&buffer_, value.size());

    // 写入数据
  // Add string delta to buffer_ followed by value
  buffer_.append(key.data() + shared, non_shared);
  buffer_.append(value.data(), value.size());

  // Update state
    // 把当前key作为下一个用于比较的key
  last_key_.resize(shared);
  last_key_.append(key.data() + shared, non_shared);
  assert(Slice(last_key_) == key);
  counter_++;
}

```

这部分代码相对简单，首先确定当前要add的key与之前最后一个加入的key的共享长度，非共享长度，然后写入这些元数据(<shared><non_shared><value_size>)， 接着再写入数据（数据只用写非共享长度即可）。**最后将当前key作为下一次add key的比较对象。**

==前缀压缩的时候当前key主要参照的是他/它前面的一个key，而不是起始点的key。==

#### key value pair的layout

从上面的代码中，我们可以知道datablock里面一个key value pair的数据格式为：

![](https://pic.downk.cc/item/5f8444041cd1bbb86b061575.png)

## 4. Index Block

我们在前面说明了如何存放data block，既然可以存放，那自然可以取出来，问题是应该如何存储，注意到sstable内的datablock是有序的，自然会想到采用二分查找的方法来做搜索。leveldb是如何实现data block的搜索的？答案就在这个index block中。

index block中存放的是data block的索引。看看index block的类型：

![image-20200923192147959](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200923192147959.png)

可以看到index block和data_block都是BlockBuilder类型，意味着index_block和data_block的存储格式是相同的。那index_block中一个entry中的key value分别是什么？

这里先给出答案，假设index block索引到的块为data_block1, data_block1的key="key1", 与data_block1相邻的是data_block2, data_block2的key='key3', 则 index block entry的key是处于["key1","key3"]之间的key，这里可以取"key2", 也即:
$$
index\  entry 索引的data block 的key <= index\  entry的key <= index\ entry索引的data\ block的下一个data\ block的key
$$
那index entry的value是什么？既然index entry要能索引一个data block， 这个value就是用来存放这个data block的位置信息的， 也即该data block的(offset,size).

### 1. Index Block中一个Index entry的数据格式

ok，画个图来说明一下:

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb源码阅读 (2).png" style="zoom:50%;" />

### 2. 什么时候会生成一个Index entry?

下面结合源码看一下

==TableBuilder::Add==

```c++
void TableBuilder::Add(const Slice& key, const Slice& value) {
   ...		
     // 添加index entry
  if (r->pending_index_entry) {
    assert(r->data_block.empty());
      // 计算index entry key
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    std::string handle_encoding;
      // 编码data block的位置信息
    r->pending_handle.EncodeTo(&handle_encoding);
      // 添加一个index entry
    r->index_block.Add(r->last_key, Slice(handle_encoding));
      // 生成添加一个index entry后，设置为false,等待下一个data block被Flush
    r->pending_index_entry = false;
  }

  if (r->filter_block != nullptr) {
    r->filter_block->AddKey(key);
  }
   ...	
}

```

从上面的代码可以看出，是否添加一个index entry, 由pending_index_entry控制。看下它的源码定义：

```c++
  // We do not emit the index entry for a block until we have seen the
  // first key for the next data block.  This allows us to use shorter
  // keys in the index block.  For example, consider a block boundary
  // between the keys "the quick brown fox" and "the who".  We can use
  // "the r" as the key for the index block entry since it is >= all
  // entries in the first block and < all entries in subsequent
  // blocks.
  //
  // Invariant: r->pending_index_entry is true only if data_block is empty.
  bool pending_index_entry;
```

上面的意思是说，只有当下一个block的第一个key Add进来时，pending_index_entry会被设置为true.  那什么时候添加一个index entry就很明朗了，==当当前data block被Flush到SST, 且下一个block的第一个key被添加时，会写入一个index entry，用于索引刚才被Flush的block==。

我们来看看pending_index_entry在源码中是什么时候被设置为true的。

```c++
void TableBuilder::Flush() {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->data_block.empty()) return;
  assert(!r->pending_index_entry);
   // 注意这里的pending_handle, 保存了这个data block的位置信息。
  WriteBlock(&r->data_block, &r->pending_handle);
  if (ok()) {
      // 这里被设置为true
    r->pending_index_entry = true;
    r->status = r->file->Flush();
  }
  if (r->filter_block != nullptr) {
    r->filter_block->StartBlock(r->offset);
  }
}
```

这个函数说明了两个点：

1. 解释了pending_index_entry在一个block被flush时才被设置为true.

2. 解释了index entry的value，即指向的data block的位置信息是什么时候生成的。 这里在深追一点

   ```c++
   void TableBuilder::WriteRawBlock(const Slice& block_contents,
                                    CompressionType type, BlockHandle* handle) {
     Rep* r = rep_;
       // 在这里设置
     handle->set_offset(r->offset);
     handle->set_size(block_contents.size());
     r->status = r->file->Append(block_contents);
     	...
     }
   }
   ```

### 3. index entry的key是如何计算的？

前面说了 index entry key 是介于两个block的key之间的。那它是如何计算的？

```c++
void TableBuilder::Add(const Slice& key, const Slice& value) {
	xxx

  if (r->pending_index_entry) {
    assert(r->data_block.empty());
      // 将计算后的key放在r->last_key中。
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    std::string handle_encoding;
    r->pending_handle.EncodeTo(&handle_encoding);
    r->index_block.Add(r->last_key, Slice(handle_encoding));
    r->pending_index_entry = false;
  }
  xxx
}
```

重点在==FindShortestSeparator==函数:

```c++
  void FindShortestSeparator(std::string* start,
                             const Slice& limit) const override {
 	// diff_index指向 start串和limit串的第一个不同点。 或者其中一个是另一个的子串
      // Find length of common prefix
    size_t min_length = std::min(start->size(), limit.size());
    size_t diff_index = 0;
    while ((diff_index < min_length) &&
           ((*start)[diff_index] == limit[diff_index])) {
      diff_index++;
    }

    if (diff_index >= min_length) {	// 如果其中一个是另一个的子串， 直接用start串作为entry的key
      // Do not shorten if one string is a prefix of the other
    } else {	// 否则
      uint8_t diff_byte = static_cast<uint8_t>((*start)[diff_index]);
      if (diff_byte < static_cast<uint8_t>(0xff) &&
          diff_byte + 1 < static_cast<uint8_t>(limit[diff_index])) {
        (*start)[diff_index]++;	// 不同的地方+1，使得生成的key 处于 [上一个block key， 下一个block key]
        start->resize(diff_index + 1);	// 生成最终key
        assert(Compare(*start, limit) < 0);
      }
    }
  }
```

这里解释一下上面的代码：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb源码阅读-index entry key计算方式.png" style="zoom:50%;" />

这里分别说两点：

1. 为什么start可以直接作为entry的key，不是要求entry key要 <=limit吗？ 万一start大于limit了呢？

   这里limit是新block的新key，一定是比旧block的key大的。

2. 后半段代码分析

```c++
 uint8_t diff_byte = static_cast<uint8_t>((*start)[diff_index]);	
      if (diff_byte < static_cast<uint8_t>(0xff) &&
          diff_byte + 1 < static_cast<uint8_t>(limit[diff_index])) {
        (*start)[diff_index]++;	// 不同的地方+1，使得生成的key 处于 [上一个block key， 下一个block key]
        start->resize(diff_index + 1);	// 生成最终key
        assert(Compare(*start, limit) < 0);
      }
```

1. 取出第一个不同的字节，要求这个字节所对应的 uint8 value <255 并且 该值+1 后的值也小于 Limit对应位置的value。

2. 首个不同字节的位置的值+1

这里我描述的不好，所以举个例子：

==情况1: start不是limit的子串==

```
 *start:    helloleveldb        上一个data block的最后一个key
 limit:     helloworld          下一个data block的第一个key
```

那么调用FindShortestSeparator后：

```
start变成： hellom (保留前缀，第一个不相同的字符+1)
即l+1 = m
```

==情况2: start是limit的子串==

```
 *start:    hello               上一个data block的最后一个key
 limit:     helloworld          下一个data block的第一个key
```

那么调用FindShortestSeparator后：

```
start变成：
 hello
```

### 4. index block如何持久化到硬件上？

上面所做的Add操作，只是将生成的index entry放在内存的index block，如何将index block持久化？

==TableBuilder::Finish==

```c++
Status TableBuilder::Finish() {
  xxx

  // Write index block
  if (ok()) {
    if (r->pending_index_entry) {
      r->options.comparator->FindShortSuccessor(&r->last_key);
      std::string handle_encoding;
      r->pending_handle.EncodeTo(&handle_encoding);
      r->index_block.Add(r->last_key, Slice(handle_encoding));
      r->pending_index_entry = false;
    }
      // 持久化index block,并记录index block的位置信息到index_block_handle
    WriteBlock(&r->index_block, &index_block_handle);
  }

  // Write footer
  if (ok()) {
    Footer footer;
    footer.set_metaindex_handle(metaindex_block_handle);
     // 记录index block handle的位置信息到footer中
    footer.set_index_handle(index_block_handle);
    std::string footer_encoding;
    footer.EncodeTo(&footer_encoding);
    r->status = r->file->Append(footer_encoding);
    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }
  return r->status;
}
```

这里的重点都用注释表示了。

现在再想想如何索引一个key？ 首先找到固定的footer，通过footer找到index block,通过index block找到data block, 在通过data block就可以找到key了。

### 5. 什么时候持久化index block?

目前有3个地方都调用了 ==TableBuilder::Finish==

![image-20200923203705354](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200923203705354.png)

只有持久化SSTable时，才会持久化index block. 从SSTable的layout也可以看出来，因为一个SSTable只有一个index block。

## 5. Meta Block & Meta Index Block

前面介绍了 footer, data block 和 index block, 似乎已经可以完成数据的存与取，但是这样会带来一个问题，search性能过低的问题。如果每次都从footer->index block->data  block->key / value pair, search性能明显不行。 所以leveldb使用了bloom filter来做优化。

bloom filter是什么，参数的设定，可参考：

Onenote中笔记，后序添加。

这里先给出bloom filter的几个参数：

1. m: bloom filter的比特位数
2. n: 预计的key的个数
3. k: 使用的hash function的个数

### 1. BloomFilterPolicy：

```c++
class BloomFilterPolicy : public FilterPolicy {
 public:
  explicit BloomFilterPolicy(int bits_per_key) : bits_per_key_(bits_per_key) {
    // We intentionally round down to reduce probing cost a little bit
    k_ = static_cast<size_t>(bits_per_key * 0.69);  // 0.69 =~ ln(2)
    if (k_ < 1) k_ = 1;
    if (k_ > 30) k_ = 30;
  }

  const char* Name() const override { return "leveldb.BuiltinBloomFilter2"; }

  void CreateFilter(const Slice* keys, int n, std::string* dst) const override {
    // Compute bloom filter size (in both bits and bytes)
    size_t bits = n * bits_per_key_;

    // For small n, we can see a very high false positive rate.  Fix it
    // by enforcing a minimum bloom filter length.
    if (bits < 64) bits = 64;

    size_t bytes = (bits + 7) / 8;
    bits = bytes * 8;

    const size_t init_size = dst->size();
    dst->resize(init_size + bytes, 0);
    dst->push_back(static_cast<char>(k_));  // Remember # of probes in filter
    char* array = &(*dst)[init_size];
    for (int i = 0; i < n; i++) {
      // Use double-hashing to generate a sequence of hash values.
      // See analysis in [Kirsch,Mitzenmacher 2006].
      uint32_t h = BloomHash(keys[i]);
      const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits
      for (size_t j = 0; j < k_; j++) {
        const uint32_t bitpos = h % bits;
        array[bitpos / 8] |= (1 << (bitpos % 8));
        h += delta;
      }
    }
  }

  bool KeyMayMatch(const Slice& key, const Slice& bloom_filter) const override {
    const size_t len = bloom_filter.size();
    if (len < 2) return false;

    const char* array = bloom_filter.data();
    const size_t bits = (len - 1) * 8;

    // Use the encoded k so that we can read filters generated by
    // bloom filters created using different parameters.
    const size_t k = array[len - 1];
    if (k > 30) {
      // Reserved for potentially new encodings for short bloom filters.
      // Consider it a match.
      return true;
    }

    uint32_t h = BloomHash(key);
    const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits
    for (size_t j = 0; j < k; j++) {
      const uint32_t bitpos = h % bits;
      if ((array[bitpos / 8] & (1 << (bitpos % 8))) == 0) return false;
      h += delta;
    }
    return true;
  }

 private:
  size_t bits_per_key_;		// m/n
  size_t k_;		// hash function 个数
};
```

#### 1. 初始化

```c++
  explicit BloomFilterPolicy(int bits_per_key) : bits_per_key_(bits_per_key) {
    // We intentionally round down to reduce probing cost a little bit
    k_ = static_cast<size_t>(bits_per_key * 0.69);  // 0.69 =~ ln(2)
    if (k_ < 1) k_ = 1;
    if (k_ > 30) k_ = 30;
  }
```

bloom filter的初始化只用到了bits_per_key, hash function个数可以通过计算得到:
$$
k = bits\_per\_key \times log2
$$


#### 2. CreateFilter

```c++
  void CreateFilter(const Slice* keys, int n, std::string* dst) const override {
      // 1.得到bloom filter用的bit数
    // Compute bloom filter size (in both bits and bytes)
    size_t bits = n * bits_per_key_;

    // For small n, we can see a very high false positive rate.  Fix it
    // by enforcing a minimum bloom filter length.
    if (bits < 64) bits = 64;

    size_t bytes = (bits + 7) / 8;
    bits = bytes * 8;

    const size_t init_size = dst->size();
    dst->resize(init_size + bytes, 0);
    dst->push_back(static_cast<char>(k_));  // Remember # of probes in filter
    char* array = &(*dst)[init_size];
    for (int i = 0; i < n; i++) {		// 外循环计算每个key，n代表key的个数
      // Use double-hashing to generate a sequence of hash values.
      // See analysis in [Kirsch,Mitzenmacher 2006].
      uint32_t h = BloomHash(keys[i]);
      const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits
      for (size_t j = 0; j < k_; j++) {	 // 内循环计算多个hash
        const uint32_t bitpos = h % bits;
        array[bitpos / 8] |= (1 << (bitpos % 8));
        h += delta;
      }
    }
  }
```

值得一说的是，在代码末尾计算多个hash时，并不是真的用了多个hash函数（多hash计算浪费计算资源），而是采用了`[Kirsch,Mitzenmacher 2006]`中的方法，一个hash函数，然后移位的方法来替代多次hash。

> Leveldb使用了double hashing来模拟多个hash函数，当然这里不是用来解决冲突的。
>
> 和线性再探测（linearprobing）一样，Double hashing从一个hash值开始，重复向前迭代，直到解决冲突或者搜索完hash表。不同的是，double hashing使用的是另外一个hash函数，而不是固定的步长。
>
> 给定两个独立的hash函数h1和h2，对于hash表T和值k，第i次迭代计算出的位置就是：h(i, k) = (h1(k) + i*h2(k)) mod |T|。
>
> 对此，Leveldb选择的hash函数是：
>
> Gi(x)=H1(x)+iH2(x)
>
> H2(x)=(H1(x)>>17) | (H1(x)<<15)
>
> H1是一个基本的hash函数，H2是由H1循环右移得到的，Gi(x)就是第i次循环得到的hash值。【理论分析可参考论文Kirsch,Mitzenmacher2006】





> 注意，因为sstable中key的个数可能很多，当攒了足够多个key值，就会计算一批位图，再攒一批key，又计算一批位图，那么这么多bloom filter的位图，必需分隔开，否则就混了。
>
> 也就说，位图与位图的边界必需清晰，否则就乱了。

==FilterBlockBuilder::GenerateFilter==

```c++
void FilterBlockBuilder::GenerateFilter() {
  const size_t num_keys = start_.size();
  if (num_keys == 0) {
    // Fast path if there are no keys for this filter
    filter_offsets_.push_back(result_.size());
    return;
  }
	
   // tmp_keys中存放本轮要进行计算的keys
  // Make list of keys from flattened key structure
  start_.push_back(keys_.size());  // Simplify length computation
  tmp_keys_.resize(num_keys);
  for (size_t i = 0; i < num_keys; i++) {
      // keys_中存放了所有key，通过偏移量start[i]和length (start[i+1] - start[i]) 得到一个key
    const char* base = keys_.data() + start_[i];
    size_t length = start_[i + 1] - start_[i];
    tmp_keys_[i] = Slice(base, length);
  }

    // 记录上一轮计算的结果
  // Generate filter for current set of keys and append to result_.
  filter_offsets_.push_back(result_.size());
    // 计算本轮keys的filter 位图，并将结果放入到result_中
  policy_->CreateFilter(&tmp_keys_[0], static_cast<int>(num_keys), &result_);

  // 清空本轮涉及的所有keys
  tmp_keys_.clear();
  keys_.clear();
  start_.clear();
}

```

start_和keys是如何生成的？

```c++
void FilterBlockBuilder::AddKey(const Slice& key) {
  Slice k = key;
  start_.push_back(keys_.size());	// 记录当前已经存入的keys的size，可以在后面用这些size来分割出每个key的长度
  keys_.append(k.data(), k.size());
}
```

这个函数在`TableBuilder::Add`中被调用：

```c++

void TableBuilder::Add(const Slice& key, const Slice& value) {
   xxx
  if (r->filter_block != nullptr) {
      // !!
    r->filter_block->AddKey(key);	
  }

  xxx

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}

```

另外

```c++
 policy_->CreateFilter(&tmp_keys_[0], static_cast<int>(num_keys), &result_);
```

result_是一个长字节数组（string表示）。里面存放了多论filter位图的计算结果。每次计算都在result\_后面apped.

#### 3. FilterBlockBuilder结构

```c++
// A FilterBlockBuilder is used to construct all of the filters for a
// particular Table.  It generates a single string which is stored as
// a special block in the Table.
//
// The sequence of calls to FilterBlockBuilder must match the regexp:
//      (StartBlock AddKey*)* Finish
class FilterBlockBuilder {
 public:
  explicit FilterBlockBuilder(const FilterPolicy*);

  void StartBlock(uint64_t block_offset);
  void AddKey(const Slice& key);
  Slice Finish();

 private:
  void GenerateFilter();

  const FilterPolicy* policy_;
  
  /*注意本轮keys产生的位图计算完毕后，会将keys_, start_ ,还有tmp_keys_ 清空*/
  std::string keys_;              // 暂时存放本轮所有keys，追加往后写入
  std::vector<size_t> start_;     // 记录本轮key与key之间的边界的位置，便于分割成多个key
  
  std::string result_;            // 计算出来的位图，多轮计算则往后追加写入
  std::vector<Slice> tmp_keys_;   // 将本轮的所有key，存入该vector，其实并无存在的必要，用临时变量即可
  std::vector<uint32_t> filter_offsets_; //计算出来的多个位图的边界位置，用于分隔多轮keys产生的位图

  // No copying allowed
  FilterBlockBuilder(const FilterBlockBuilder&);
  void operator=(const FilterBlockBuilder&);
};

```

#### 4. when & how to 得到filter 和 filter offset？

==FilterBlockBuilder::StartBlock==

```c++
// See doc/table_format.md for an explanation of the filter block format.

// Generate new filter every 2KB of data
static const size_t kFilterBaseLg = 11;
static const size_t kFilterBase = 1 << kFilterBaseLg;

FilterBlockBuilder::FilterBlockBuilder(const FilterPolicy* policy)
    : policy_(policy) {}

void FilterBlockBuilder::StartBlock(uint64_t block_offset) {
  uint64_t filter_index = (block_offset / kFilterBase);
  assert(filter_index >= filter_offsets_.size());
  while (filter_index > filter_offsets_.size()) {
    GenerateFilter();
  }
}
```

从代码注释上来看，每2kb的data就会生成一个filter。但真的是这样吗？

```c++
 uint64_t filter_index = (block_offset / kFilterBase);	// 计算当前offset所在位置的数据的filter index，入 block_offset=3k,则filter_indx=1
```

```
　filter_offsets_.size() 返回的是当前整个sstable的filter 个数
```

所以：

```c++
while (filter_index > filter_offsets_.size()) {
    GenerateFilter();
  }
```

##### 1. how

假设（注意我说的是假设）GenerateFilter一次处理2kb数据，filter_offsets\_.size() 会+1，则上面的代码即将data按照2kb分一个filter来处理。但是实际上==GenerateFilter==的处理方式并不是按2kbf分块处理为，但可确定是，GenerateFilter函数内部，每进行一轮计算，filter_offsets_.size()会+1.

下面看看GenerateFilter的源码：

```c++
void FilterBlockBuilder::GenerateFilter() {
    // 根据前面说所，可推得start_.size()为当前Add进来的key的个数
  const size_t num_keys = start_.size();
  if (num_keys == 0) {
    // Fast path if there are no keys for this filter
    filter_offsets_.push_back(result_.size());
    return;
  }

    // 通过start_和key_将 所有添加的key加入到tmp_keys_中。
  // Make list of keys from flattened key structure
  start_.push_back(keys_.size());  // Simplify length computation
  tmp_keys_.resize(num_keys);
  for (size_t i = 0; i < num_keys; i++) {
    const char* base = keys_.data() + start_[i];
    size_t length = start_[i + 1] - start_[i];
    tmp_keys_[i] = Slice(base, length);
  }

   // 为所有添加进来的 keys 生成一个 Filter，并将结果 append到 result_中
  // Generate filter for current set of keys and append to result_.
  filter_offsets_.push_back(result_.size());	// ！filter_offsets_.size() +1
  policy_->CreateFilter(&tmp_keys_[0], static_cast<int>(num_keys), &result_);

  tmp_keys_.clear();
  keys_.clear();
  start_.clear();
}

```

读取上面的代码，我们可以知道，在一次==GenerateFilter==中，就处理完了之前所有Add进来的keys，此时 filter_offsets_的size只+1，所以

```c++
while (filter_index > filter_offsets_.size()) {
    GenerateFilter();
  }
```

这个循环，仍在继续。而后序的调用会进入到这个分支：

```c++
void FilterBlockBuilder::GenerateFilter() {
    // 根据前面说所，可推得start_.size()为当前Add进来的key的个数
  const size_t num_keys = start_.size();
  if (num_keys == 0) {
     // 只增加filter_offsets_的size，没有生成新的filter
    // Fast path if there are no keys for this filter
    filter_offsets_.push_back(result_.size());
    return;
  }

```

另外还有没有说的是，filter_offsets_存放的就是各个filter的位置信息（偏移量）。所以除了第一次调用GenerateFilter会生成一个filter，后序的都是空filter。

##### 例子

假设当前已经插入的data block大小达到6kb，则GenerateFilter会调用三次。下面给出filter block的图解：

第一次：

```c++
   // 为所有添加进来的 keys 生成一个 Filter，并将结果 append到 result_中
  // Generate filter for current set of keys and append to result_.
  filter_offsets_.push_back(result_.size());	// ！filter_offsets_.size() +1
  policy_->CreateFilter(&tmp_keys_[0], static_cast<int>(num_keys), &result_);
```

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb源码阅读-copy-filter block插入流程1 (1).png" style="zoom: 33%;" />

第二次：

```c++
  if (num_keys == 0) {
    // Fast path if there are no keys for this filter
    filter_offsets_.push_back(result_.size());		// 此时第一次调用已经使得result_ size增加, 假设这里增加了9
    return;
  }
```

<img src="https://pic.downk.cc/item/5f7184db160a154a670fe620.png" style="zoom: 33%;" />



第三次：

```c++
  if (num_keys == 0) {
    // Fast path if there are no keys for this filter
    filter_offsets_.push_back(result_.size());		// 此时result未发生变化，size依然等于9
    return;
  }
```

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb源码阅读-copy-filter block插入流程3.png" style="zoom:33%;" />



这看起来似乎有点奇怪， 注释中说的是一个filter对应2kb的data block, 一个filter offset应该对应一个filter才对，但是这里多个filter offset对应到了一个filter。留个悬念，下面的 `KeyMayMatch`中会进行讲解。

##### 2. when

现在，回到问题，什么时候发起一轮位图计算（即生成一个filter）。

答案：Flush函数。

==TableBuilder::Flush==调用==FilterBlockBuilder::StartBlock==:

```c++
void TableBuilder::Flush() {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->data_block.empty()) return;
  assert(!r->pending_index_entry);
  WriteBlock(&r->data_block, &r->pending_handle);
  if (ok()) {
    r->pending_index_entry = true;
    r->status = r->file->Flush();
  }
  if (r->filter_block != nullptr) {
      // !! 
    r->filter_block->StartBlock(r->offset);
  }
}
```

什么时候Flush?

当前 data_block超过阈值4kb时。

```c++
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
	xxx

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}
```

#### 5. 生成filter的调用流程图

![leveldb源码阅读-copy-第 3 页](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb源码阅读-copy-第 3 页.png)

#### 6. KeyMayMatch

前面说了何时以及如何生成filter和filter offset，有了这两样，我们应该如何使用filter？

**==传入 data_block 的 block_offset==**, 计算给定key是否在这个data block中。

传入 data_block 的 block_offset

传入 data_block 的 block_offset。

重要的事情说3遍。**传入的的参数是一个block的起始offset，而不是一个key所在data block的offset。**

先看代码：

```c++
bool FilterBlockReader::KeyMayMatch(uint64_t block_offset, const Slice& key) {
  uint64_t index = block_offset >> base_lg_;
  if (index < num_) {
      // 启始偏移
    uint32_t start = DecodeFixed32(offset_ + index * 4);
     // 结尾偏移量
    uint32_t limit = DecodeFixed32(offset_ + index * 4 + 4);
    if (start <= limit && limit <= static_cast<size_t>(offset_ - data_)) {	// 考虑一下，什么时候start会=limit
        // 得到filter
      Slice filter = Slice(data_ + start, limit - start);
        // 正式匹配
      return policy_->KeyMayMatch(key, filter);
    } else if (start == limit) {
      // Empty filters do not match any keys
      return false;
    }
  }
  return true;  // Errors are treated as potential matches
}
```

再看 return policy_->KeyMayMatch(key, filter);

```c++
  bool KeyMayMatch(const Slice& key, const Slice& bloom_filter) const override {
    const size_t len = bloom_filter.size();
    if (len < 2) return false;

    const char* array = bloom_filter.data();
    const size_t bits = (len - 1) * 8;

   	xxx
        
    // bloom filter 的hash index判断，只要有一个0，则返回false，都为1，返回true
    uint32_t h = BloomHash(key);
    const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits
    for (size_t j = 0; j < k; j++) {
      const uint32_t bitpos = h % bits;
      if ((array[bitpos / 8] & (1 << (bitpos % 8))) == 0) return false;
      h += delta;
    }
    return true;
  }
```

假设之前写入了两个data block，一个6kb，一个4kb，则对应的data block和 filter block如下:

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb源码阅读-copy-第 12 页 (5).png" style="zoom:33%;" />

现在判定一个key是否在data block1, 只用传入data block1起始地址0， 计算得到 index = 0, 则start= 0，limit=9， 所以得到filter的地址未0-9.

判定一个key是否在data block2， 传入data block2起始地址6k，计算得到index=3，则start=9,limit=15。

这回答了一个问题，中间 filter offset 1, filter offset 2虽然指向了同一个filter，但是实际上，filter offset2从不会被访问，所以并不影响。==**那为什么要这样设计呢？**==

想想如何没有这 ”2kb分块“的设计，我们是无法快速的 通过一个block_offset就定位到一个filter offset，进而定位到filter。所以这样的设计还是很巧妙的。

==**另外，还有个问题**==

什么时候:

```c++
start == limit
```

filter 为空时。 

### 2. Meta Block 在SSTable中的布局

#### 1. 什么时候写入Meta Block（持久化 filter block)

==TableBuilder::Finish==中:

```c++
Status TableBuilder::Finish() {
  xxx

  BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle;

  // Write filter block
  if (ok() && r->filter_block != nullptr) {
     // ！！这里写入filter_block！！
    WriteRawBlock(r->filter_block->Finish(), kNoCompression,
                  &filter_block_handle);
  }

  // Write metaindex block
  if (ok()) {
    BlockBuilder meta_index_block(&r->options);
    if (r->filter_block != nullptr) {
      // Add mapping from "filter.Name" to location of filter data
      std::string key = "filter.";
      key.append(r->options.filter_policy->Name());
      std::string handle_encoding;
      filter_block_handle.EncodeTo(&handle_encoding);
      meta_index_block.Add(key, handle_encoding);
    }

    // TODO(postrelease): Add stats and other meta blocks
    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }

  xxx
  return r->status;
}
```

```c++
Slice FilterBlockBuilder::Finish() {
  if (!start_.empty()) {
    GenerateFilter();
  }

  // Append array of per-filter offsets
    // result_.size()是总filter的长度，也是 filter_offsets的起始偏移量
  const uint32_t array_offset = result_.size();
  for (size_t i = 0; i < filter_offsets_.size(); i++) {
    PutFixed32(&result_, filter_offsets_[i]);
  }
	
  PutFixed32(&result_, array_offset);
    // filter编码参数
  result_.push_back(kFilterBaseLg);  // Save encoding parameter in result
  return Slice(result_);
}
```

#### 2. Meta Block（filter block）& Meta Index Block的布局

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb源码阅读-copy-filter block的layout (1).png" style="zoom:50%;" />

#### 3. meta index block的写入

```c++
  // Write metaindex block
  if (ok()) {
    BlockBuilder meta_index_block(&r->options);
    if (r->filter_block != nullptr) {
      // Add mapping from "filter.Name" to location of filter data
      std::string key = "filter.";
      key.append(r->options.filter_policy->Name());
      std::string handle_encoding;
      filter_block_handle.EncodeTo(&handle_encoding);
      meta_index_block.Add(key, handle_encoding);
    }

    // TODO(postrelease): Add stats and other meta blocks
    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }

```

meta index block和 data block以及index block一样，都是BlockBuilder类型。

key: meta index block的key是 **filter.Name**, 其中，Name来自 filter_policy->Name()调用。

value:  value是filter_block_handle,也就是 filter_block的位置信息。 还记得位置信息是在哪里生成的吗？

```c++
 // Write filter block
  if (ok() && r->filter_block != nullptr) {
     // !! 这里生成的 filter_block_handle信息s
    WriteRawBlock(r->filter_block->Finish(), kNoCompression,
                  &filter_block_handle);
  }

  // Write metaindex block
...
```