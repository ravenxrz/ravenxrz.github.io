---
title: leveldb源码阅读记录-Cache
categories: leveldb
abbrlink: ce22f5e9
date: 2020-10-12 18:22:00
tags:
---

leveldb是一种对写优化的kv存储系统，读性能有所下降，为了充分利用局部性原理，提高读性能，leveldb自己也设计了一个Cache结构。内部采用LRU替换策略。

<!--more-->

leveldb中的cache主要包含以下类：

- LRUHandle -- 数据节点
- HandleTable -- HashTable
- LRUCache
- ShardedLRUCache

事实上到了第三个数据结构LRUCache，LRU的缓存管理数据结构已经实现了，之所以引入第四个数据结构，就是因为减少竞争。因为多线程访问需要加锁，为了减少竞争，提升效率，ShardedLRUCache内部有**16个LRUCach**e，查找key的时候，先计算属于哪一个LRUCache，然后在相应的LRUCache中上锁查找。

```c++
class ShardedLRUCache : public Cache {  
 private:  
  LRUCache shard_[kNumShards];  
  ...
}
```

这不是什么高深的思路，这种减少竞争的策略非常常见。因此，读懂缓存管理策略的关键在前三个数据结构。

LevelDB的Cache管理，维护有2个双向链表和一个哈希表。哈希表是非常容易理解的。如何确定一个key值到底存不存在，如果存在如何快速获取key值对应的value值。我们都学过数据结构，这活，哈希表是比较适合的。

注意，我们都知道，hash表存在一个重要的问题，就是碰撞，有可能多个不同的键值hash之后值相同，解决碰撞的一个重要思路是链表，将hash之后计算的key相同的元素链入同一个表头对应的链表。

可是我们并不满意这种速度，LevelDB做了进一步的优化，即及时扩大hash桶的个数，尽可能地不会发生碰撞。因此LevelDB自己实现了一个hash表，即HandleTable数据结构。

说句题外话，我不太喜欢数据结构的命名方式，比如HandleTable，命名就是个HashTable，如果出现Hash会好理解很多。这个名字还自罢了，LRUHandle这个名字更是让人摸不到头脑，明明就是一个数据节点，如果名字中出现Node，整个代码都会好理解很多。好了吐槽结束，看下HandleTable的数据结构：

## 1. HandleTable

HandleTable在本质上就是一个HashTable，只是leveldb做了优化，通过尽早的拓展桶，来达到减少碰撞的目的。

成员变量：

```c++
private:
  // The table consists of an array of buckets where each bucket is
  // a linked list of cache entries that hash into the bucket.
  uint32_t length_;	// buckets个数
  uint32_t elems_;  // 当前插入的elem个数
  LRUHandle** list_;	// 二级指针，每个一级指针指向一个桶
```

### Insert

```c++
  LRUHandle* Insert(LRUHandle* h) {
    LRUHandle** ptr = FindPointer(h->key(), h->hash);
    LRUHandle* old = *ptr;
    h->next_hash = (old == nullptr ? nullptr : old->next_hash);
    *ptr = h;	// 替换操作
    if (old == nullptr) {
      ++elems_;
      if (elems_ > length_) {	// 尽可能保证一个bucket只有一个elem
        // Since each cache entry is fairly large, we aim for a small
        // average linked list length (<= 1).
        Resize();
      }
    }
    return old;
  }
```

insert函数首先在当前hashtable中尝试找到与插入key具有相同key的entry，即旧entry。 如果存在旧entry，则将旧的覆盖为新的节点。如果不存在旧的entry，则添加一个新的节点。同时可能需要resize hash， resize的条件为“当前插入的节点数比hashtable 的 bucket数大”，这样做的目的是尽量保证每个bucket下只有一个entry，这样search时间能够保证在O(1)。

### Resize

```c++
 void Resize() {
    uint32_t new_length = 4;	// 保证新容量是4的整数倍
    while (new_length < elems_) {	// 2倍扩容
      new_length *= 2;
    }
     // 新hashtable
    LRUHandle** new_list = new LRUHandle*[new_length];
    memset(new_list, 0, sizeof(new_list[0]) * new_length);
    uint32_t count = 0;
    // 移动旧hashtable中的元素到新hashtable
    for (uint32_t i = 0; i < length_; i++) {
      LRUHandle* h = list_[i];
      while (h != nullptr) {
        LRUHandle* next = h->next_hash;
        uint32_t hash = h->hash;
		// 下面3行代码的结果是，如果旧hashtable的一个bucket的多个node，都重新链接到了这个新hashtable的同一个bucket，则这些node将会反序连接
        LRUHandle** ptr = &new_list[hash & (new_length - 1)];
        h->next_hash = *ptr;
        *ptr = h;
        h = next;
        count++;
      }
    }
    assert(elems_ == count);
     // 删除旧表
    delete[] list_;
    list_ = new_list;
    length_ = new_length;
  }
```

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/leveldb源码阅读-copy-第 23 页.png" style="zoom:33%;" />

### Remove

```c++
  LRUHandle* Remove(const Slice& key, uint32_t hash) {
    LRUHandle** ptr = FindPointer(key, hash);
    LRUHandle* result = *ptr;
    if (result != nullptr) {
      *ptr = result->next_hash;	// remove掉当前节点，并指向下一个节点
      --elems_;
    }
    return result;
  }

  // Return a pointer to slot that points to a cache entry that
  // matches key/hash.  If there is no such cache entry, return a
  // pointer to the trailing slot in the corresponding linked list.
  LRUHandle** FindPointer(const Slice& key, uint32_t hash) {
    LRUHandle** ptr = &list_[hash & (length_ - 1)];		// 二级指针
    while (*ptr != nullptr && ((*ptr)->hash != hash || key != (*ptr)->key())) {
      ptr = &(*ptr)->next_hash;
    }
    return ptr;
  }
```

注意，这里remove的entry并没有free掉。entry的free在后文的LRUCache中。

<img src="https://pic.downk.cc/item/5f8421811cd1bbb86be7cd0d.png" style="zoom:33%;" />



### 总结

leveldb的hashtable其实就是一个数组+链表的hashtable，只不过rehash操作做了优化，从而加快search的效率。

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/绘图文件-第 3 页.png" style="zoom:50%;" />





## 2. LRUHandle

```c++
struct LRUHandle {
  void* value;
  void (*deleter)(const Slice&, void* value);
  LRUHandle* next_hash;
  LRUHandle* next;		// 双链表的next
  LRUHandle* prev;		// 双链表的prev
  size_t charge;  // TODO(opt): Only allow uint32_t?
  size_t key_length;
  bool in_cache;     // Whether entry is in the cache.
  uint32_t refs;     // References, including cache reference, if present. 引用计数
  uint32_t hash;     // Hash of key(); used for fast sharding and comparisons
  char key_data[1];  // Beginning of key, ！！ 占位符，这里放在结构体的最后且只有一个字节是有目的的，后面说到

  Slice key() const {
    // next_ is only equal to this if the LRU handle is the list head of an
    // empty list. List heads never have meaningful keys.
    assert(next != this);

    return Slice(key_data, key_length);
  }
};
```

LRUHandle表示一个cache节点。其中next_hash字段用在hashtable中，表明相同bucekt下的下一个节点。

## 3. LRUCache

### 总layout：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/绘图文件-第 4 页.png" style="zoom:50%;" />

HandleTable是用来找到某个cache entry的。但是无法实现LRU算法，现在来说一下实际的LRUCache。在LRUCache的实现里维护了两个链表：

```c++
  // Dummy head of LRU list.
  // lru.prev is newest entry, lru.next is oldest entry.
  // Entries have refs==1 and in_cache==true.
  LRUHandle lru_ GUARDED_BY(mutex_);	// 冷数据

  // Dummy head of in-use list.
  // Entries are in use by clients, and have refs >= 2 and in_cache==true.
  LRUHandle in_use_ GUARDED_BY(mutex_);		// 热数据

  HandleTable table_ GUARDED_BY(mutex_); // 之前讲过
```

注意在lru_链表中， lru.prev代表最新entry，lru.next代表最旧entry。

一个cache entry在上述两条链中的其中一个，通过 Ref() and Unref()调用，一个cache entry在两条链表之间move。

### 1. Ref

Ref将一个cache 节点，从lru_链表插入到in\_user\_链表

```c++
void LRUCache::Ref(LRUHandle* e) {
  if (e->refs == 1 && e->in_cache) {  // If on lru_ list, move to in_use_ list.
    LRU_Remove(e);
    LRU_Append(&in_use_, e);
  }
  e->refs++;
}

void LRUCache::LRU_Remove(LRUHandle* e) {
  e->next->prev = e->prev;
  e->prev->next = e->next;
}

void LRUCache::LRU_Append(LRUHandle* list, LRUHandle* e) {
  // Make "e" newest entry by inserting just before *list
  // 插入到最新位置,list head处
  e->next = list;
  e->prev = list->prev;
  e->prev->next = e;
  e->next->prev = e;
}
```

### 2. UnRef

如果client不再引用一条cache entry， 则会进行UnRef，当一个cache ref=0时，则删除这个entry，当cache ref = 1时，则从热链(in\_user_)迁移到冷链(lru\_)。

```c++
void LRUCache::Unref(LRUHandle* e) {
  assert(e->refs > 0);
  e->refs--;
  if (e->refs == 0) {  // Deallocate.
    assert(!e->in_cache);
    (*e->deleter)(e->key(), e->value);
    free(e);	// 这里free
  } else if (e->in_cache && e->refs == 1) {
    // No longer in use; move to lru_ list.
    LRU_Remove(e);
    LRU_Append(&lru_, e);
  }
}
```

### 3. Insert（evict）

cache是有容量大小限制的，当插入的cache entry达到一定数量时，需要根据LRU算法剔除旧cache。这部分实现的入口在Insert函数：

```c++

Cache::Handle* LRUCache::Insert(const Slice& key, uint32_t hash, void* value,
                                size_t charge,
                                void (*deleter)(const Slice& key,
                                                void* value)) {
  MutexLock l(&mutex_);

  LRUHandle* e =		// !!! 这里解释了上面为什么申请了一个字节的 key_data并且放在最后
      reinterpret_cast<LRUHandle*>(malloc(sizeof(LRUHandle) - 1 + key.size()));
  e->value = value;
  e->deleter = deleter;
  e->charge = charge;
  e->key_length = key.size();
  e->hash = hash;
  e->in_cache = false;
  e->refs = 1;  // for the returned handle.
  std::memcpy(e->key_data, key.data(), key.size());

  if (capacity_ > 0) {
    e->refs++;  // for the cache's reference.
    e->in_cache = true;
    LRU_Append(&in_use_, e);
    usage_ += charge;
    FinishErase(table_.Insert(e));
  } else {  // don't cache. (capacity_==0 is supported and turns off caching.)
    // next is read by key() in an assert, so it must be initialized
    e->next = nullptr;
  }
    
   // ！！！超过容量，需要剔除旧cache lru_.next存放的是最旧的cache entry
  while (usage_ > capacity_ && lru_.next != &lru_) {	// 移除旧cache entry，直到当前usage 小于等于 capacity。 或者最旧的entry已经是lru_
    LRUHandle* old = lru_.next;
    assert(old->refs == 1);
     // 从hashtable中移除，从cache中删除。
    bool erased = FinishErase(table_.Remove(old->key(), old->hash));
    if (!erased) {  // to avoid unused variable when compiled NDEBUG
      assert(erased);
    }
  }

  return reinterpret_cast<Cache::Handle*>(e);
}
```

这里有3个地方需要说明：

1. LRUHandler中为什么申请了一个字节的key_data:

   ```c++
   LRUHandle* e =		// !!! 这里解释了上面为什么申请了一个字节的 key_data并且放在最后
         reinterpret_cast<LRUHandle*>(malloc(sizeof(LRUHandle) - 1 + key.size()));
   ```

   <img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/绘图文件.png" style="zoom:33%;" />

2. 新增一个cache entry到cache中：

   ```c++
   if (capacity_ > 0) {
       e->refs++;  // for the cache's reference.
       e->in_cache = true;
       LRU_Append(&in_use_, e);	// 插入到热链中 in_user_
       usage_ += charge;
       FinishErase(table_.Insert(e));	// 如果是更新，需要删除旧entry
     } 
   ```

   ```c++
     LRUHandle* Insert(LRUHandle* h) {
       LRUHandle** ptr = FindPointer(h->key(), h->hash);
       LRUHandle* old = *ptr;
       h->next_hash = (old == nullptr ? nullptr : old->next_hash);
       *ptr = h;
       if (old == nullptr) {
         ++elems_;
         if (elems_ > length_) {
           // Since each cache entry is fairly large, we aim for a small
           // average linked list length (<= 1).
           Resize();
         }
       }
       return old;
     }
   ```

   Insert在Update的情况下，会返回旧的cache entry. 在FinishErase函数调用中决定该cache entry是删除还是移动到lru_链中：

   ```c++
   // If e != nullptr, finish removing *e from the cache; it has already been
   // removed from the hash table.  Return whether e != nullptr.
   bool LRUCache::FinishErase(LRUHandle* e) {
     if (e != nullptr) {
       assert(e->in_cache);
       LRU_Remove(e);
       e->in_cache = false;
       usage_ -= e->charge;
       Unref(e);
     }
     return e != nullptr;
   }
   ```

3. LRU evict：

   ```c++
      // ！！！超过容量，需要剔除旧cache lru_.next存放的是最旧的cache entry
     while (usage_ > capacity_ && lru_.next != &lru_) {	// 移除旧cache entry，直到当前usage 小于等于 capacity。 或者最旧的entry已经是lru_头，即cache为空
       LRUHandle* old = lru_.next;
       assert(old->refs == 1);
        // 从hashtable中移除，从cache中删除。
       bool erased = FinishErase(table_.Remove(old->key(), old->hash));
       if (!erased) {  // to avoid unused variable when compiled NDEBUG
         assert(erased);
       }
     }
   ```

LRU是如何体现的？

回到插入一个entry到lru_中：

```c++
LRU_Append(&lru_, e);

void LRUCache::LRU_Append(LRUHandle* list, LRUHandle* e) {
  // Make "e" newest entry by inserting just before *list
  e->next = list;
  e->prev = list->prev;
  e->prev->next = e;
  e->next->prev = e;
}
```

每次插入都是向list head的前面插入一个新节点，作为最新节点。所以有：

![](https://pic.downk.cc/item/5f84428f1cd1bbb86b050fa0.png)

## 4. ShardedLRUCache

前面说了ShardedLRUCache是为了减少多线程的竞争延迟而设计的。在SharedLRUCache中有16个LRUCache。

```c++
static const int kNumShardBits = 4;
static const int kNumShards = 1 << kNumShardBits;

class ShardedLRUCache : public Cache {
 private:
  LRUCache shard_[kNumShards];
  port::Mutex id_mutex_;
  uint64_t last_id_;

  static inline uint32_t HashSlice(const Slice& s) {
    return Hash(s.data(), s.size(), 0);
  }

  static uint32_t Shard(uint32_t hash) { return hash >> (32 - kNumShardBits); }

 public:
  explicit ShardedLRUCache(size_t capacity) : last_id_(0) {
    const size_t per_shard = (capacity + (kNumShards - 1)) / kNumShards;
    for (int s = 0; s < kNumShards; s++) {
      shard_[s].SetCapacity(per_shard);
    }
  }
  ~ShardedLRUCache() override {}
  Handle* Insert(const Slice& key, void* value, size_t charge,
                 void (*deleter)(const Slice& key, void* value)) override {
    const uint32_t hash = HashSlice(key);
    return shard_[Shard(hash)].Insert(key, hash, value, charge, deleter);
  }
  Handle* Lookup(const Slice& key) override {
    const uint32_t hash = HashSlice(key);
    return shard_[Shard(hash)].Lookup(key, hash);
  }
  void Release(Handle* handle) override {
    LRUHandle* h = reinterpret_cast<LRUHandle*>(handle);
    shard_[Shard(h->hash)].Release(handle);
  }
  void Erase(const Slice& key) override {
    const uint32_t hash = HashSlice(key);
    shard_[Shard(hash)].Erase(key, hash);
  }
  void* Value(Handle* handle) override {
    return reinterpret_cast<LRUHandle*>(handle)->value;
  }
  uint64_t NewId() override {
    MutexLock l(&id_mutex_);
    return ++(last_id_);
  }
  void Prune() override {
    for (int s = 0; s < kNumShards; s++) {
      shard_[s].Prune();
    }
  }
  size_t TotalCharge() const override {
    size_t total = 0;
    for (int s = 0; s < kNumShards; s++) {
      total += shard_[s].TotalCharge();
    }
    return total;
  }
};
```

如何做到减少竞争带来的延迟的？现在有多个cache，对于每个插入的key，做一次hash，然后取hash结果的高4位作为cache id，用于选择此次用哪个cache来缓存数据。这样就避免了只使用一个cache时，每次插入都要加锁。

## 5. Cache应用1-TableCache

tablecache cache的是**多个sstable（不包含data block)**。其key和value对应关系如下：

![](https://pic.downk.cc/item/5f82b2b81cd1bbb86b3919c9.png)

TableCache中的cache容量为990. 

```c++

// Number of open files that can be used by the DB.  You may need to
// increase this if your database has a large working set (budget
// one open file per 2MB of working set).
int max_open_files = 1000;

const int kNumNonTableCacheFiles = 10;

static int TableCacheSize(const Options& sanitized_options) {
  // Reserve ten files or so for other uses and give the rest to TableCache.
  return sanitized_options.max_open_files - kNumNonTableCacheFiles;
}
```

下面是整个TableCache:

```c++
class TableCache {
 public:
  TableCache(const std::string& dbname, const Options& options, int entries);
  ~TableCache();

  // Return an iterator for the specified file number (the corresponding
  // file length must be exactly "file_size" bytes).  If "tableptr" is
  // non-null, also sets "*tableptr" to point to the Table object
  // underlying the returned iterator, or to nullptr if no Table object
  // underlies the returned iterator.  The returned "*tableptr" object is owned
  // by the cache and should not be deleted, and is valid for as long as the
  // returned iterator is live.
  Iterator* NewIterator(const ReadOptions& options, uint64_t file_number,
                        uint64_t file_size, Table** tableptr = nullptr);

  // If a seek to internal key "k" in specified file finds an entry,
  // call (*handle_result)(arg, found_key, found_value).
  Status Get(const ReadOptions& options, uint64_t file_number,
             uint64_t file_size, const Slice& k, void* arg,
             void (*handle_result)(void*, const Slice&, const Slice&));

  // Evict any entry for the specified file number
  void Evict(uint64_t file_number);

 private:
  Status FindTable(uint64_t file_number, uint64_t file_size, Cache::Handle**);

  Env* const env_;
  const std::string dbname_;
  const Options& options_;
  Cache* cache_;
};

```

### 1. FindTable

FindTable是在cache中根据指定filenumnber,lookup到相关cache的handle。如果找到了，直接返回该handle，如果没找到，则插入到cache中。

**此时的key是编码后的file_number. value是file_number对应的file指针以及打开后的sstable指针**

```c++
Status TableCache::FindTable(uint64_t file_number, uint64_t file_size,
                             Cache::Handle** handle) {
  Status s;
  char buf[sizeof(file_number)];
  EncodeFixed64(buf, file_number);
  Slice key(buf, sizeof(buf));
   // 在cache寻找key对应的handle
  *handle = cache_->Lookup(key);
  if (*handle == nullptr) {	// 找不到
    std::string fname = TableFileName(dbname_, file_number);
    RandomAccessFile* file = nullptr;
    Table* table = nullptr;
      // 根据fname打开file
    s = env_->NewRandomAccessFile(fname, &file);
    if (!s.ok()) {
      std::string old_fname = SSTTableFileName(dbname_, file_number);
      if (env_->NewRandomAccessFile(old_fname, &file).ok()) {
        s = Status::OK();
      }
    }
    if (s.ok()) {
        // 根据file打开table
      s = Table::Open(options_, file, file_size, &table);
    }

    if (!s.ok()) {
      assert(table == nullptr);
      delete file;
      // We do not cache error results so that if the error is transient,
      // or somebody repairs the file, we recover automatically.
    } else {
        // 插入到cache中
      TableAndFile* tf = new TableAndFile;
      tf->file = file;
      tf->table = table;
      *handle = cache_->Insert(key, tf, 1, &DeleteEntry);
    }
  }
  return s;
}
```

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/绘图文件-第 7 页.png" style="zoom:67%;" />

### 2. Get

```c++

  // If a seek to internal key "k" in specified file finds an entry,
  // call (*handle_result)(arg, found_key, found_value).
Status TableCache::Get(const ReadOptions& options, uint64_t file_number,
                       uint64_t file_size, const Slice& k, void* arg,
                       void (*handle_result)(void*, const Slice&,
                                             const Slice&)) {
  Cache::Handle* handle = nullptr;
  Status s = FindTable(file_number, file_size, &handle);
  if (s.ok()) {
      // 找到cache中的table
    Table* t = reinterpret_cast<TableAndFile*>(cache_->Value(handle))->table;
      // 在table内部寻找k，找到了对应data pair调用handle_result
    s = t->InternalGet(options, k, arg, handle_result);
    cache_->Release(handle);
  }
  return s;
}
```

```c++
Status Table::InternalGet(const ReadOptions& options, const Slice& k, void* arg,
                          void (*handle_result)(void*, const Slice&,
                                                const Slice&)) {
  Status s;
   // 生成index迭代器
  Iterator* iiter = rep_->index_block->NewIterator(rep_->options.comparator);
  iiter->Seek(k);
  if (iiter->Valid()) {
      // 找到相关data block
    Slice handle_value = iiter->value();
    FilterBlockReader* filter = rep_->filter;
    BlockHandle handle;
    if (filter != nullptr && handle.DecodeFrom(&handle_value).ok() &&
        !filter->KeyMayMatch(handle.offset(), k)) {	// 应用bloom filter检查给定key是否在这个data block中
      // Not found
    } else {	// 可能在这个data block中
       // 将index的value转换为一个block iter
      Iterator* block_iter = BlockReader(this, options, iiter->value());
      block_iter->Seek(k);
      if (block_iter->Valid()) {	// 确实在这个data block中，调用handle_result函数（这里的arg可以是一个saver，用来保存找到的key和vlaue）
        (*handle_result)(arg, block_iter->key(), block_iter->value());
      }
      s = block_iter->status();
      delete block_iter;
    }
  }
  if (s.ok()) {
    s = iiter->status();
  }
  delete iiter;
  return s;
}

```

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/绘图文件-第 8 页.png" style="zoom: 50%;" />

### 3. Evict

这个相对简单，直接在cache_中Erase即可。

```c++
void TableCache::Evict(uint64_t file_number) {
  char buf[sizeof(file_number)];
  EncodeFixed64(buf, file_number);
  cache_->Erase(Slice(buf, sizeof(buf)));
}
```

### 4.NewIterator

根据cache找到table，根据table返回iterator。

```c++
Iterator* TableCache::NewIterator(const ReadOptions& options,
                                  uint64_t file_number, uint64_t file_size,
                                  Table** tableptr) {
  if (tableptr != nullptr) {
    *tableptr = nullptr;
  }

  Cache::Handle* handle = nullptr;
  Status s = FindTable(file_number, file_size, &handle);
  if (!s.ok()) {
    return NewErrorIterator(s);
  }

  Table* table = reinterpret_cast<TableAndFile*>(cache_->Value(handle))->table;
  Iterator* result = table->NewIterator(options);
  result->RegisterCleanup(&UnrefEntry, cache_, handle);
  if (tableptr != nullptr) {
    *tableptr = table;
  }
  return result;
}
```

这里用到了TwoLevelIterator

```c++
Iterator* Table::NewIterator(const ReadOptions& options) const {
  return NewTwoLevelIterator(
      rep_->index_block->NewIterator(rep_->options.comparator),
      &Table::BlockReader, const_cast<Table*>(this), options);
}
```

**TwoLevelIterator**用index_iter和data_iter来访问数据。

**index_iter为index_block的 Block::Iter。**

**data_iter为 data block的Block::Iter。**

## 6. Cache应用2-BlockCache

BlockCache用于cache sstable中的datablock。系统默认为8M. 

**key为 cahceid+data block的位置信息(offset)。**

**value 为 data block的block封装（解压后）。**

![](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/绘图文件-第 9 页 (1).png)

### 1. BlockCache的初始化

blockcache的默认初始化在SanitizeOptions中：

```c++
Options SanitizeOptions(const std::string& dbname,
                        const InternalKeyComparator* icmp,
                        const InternalFilterPolicy* ipolicy,
                        const Options& src) {
  ...
  if (result.block_cache == nullptr) {
      // block cache 的 capacity 为 8M
    result.block_cache = NewLRUCache(8 << 20);
  }
  return result;
}
```

### 2. Block Cache的读取与插入

	目前仅在Table::BlockReader看到使用。所以以这个函数来讲解。

```c++
// Convert an index iterator value (i.e., an encoded BlockHandle)
// into an iterator over the contents of the corresponding block.
Iterator* Table::BlockReader(void* arg, const ReadOptions& options,
                             const Slice& index_value) {
  Table* table = reinterpret_cast<Table*>(arg);
    // 获取 block_cache
  Cache* block_cache = table->rep_->options.block_cache;
  Block* block = nullptr;
  Cache::Handle* cache_handle = nullptr;

  BlockHandle handle;
  Slice input = index_value;
  Status s = handle.DecodeFrom(&input);
  // We intentionally allow extra stuff in index_value so that we
  // can add more features in the future.

  if (s.ok()) {
    BlockContents contents;
    if (block_cache != nullptr) {
      char cache_key_buffer[16];
      EncodeFixed64(cache_key_buffer, table->rep_->cache_id);
      EncodeFixed64(cache_key_buffer + 8, handle.offset());
      Slice key(cache_key_buffer, sizeof(cache_key_buffer));
      cache_handle = block_cache->Lookup(key);
      if (cache_handle != nullptr) {
        block = reinterpret_cast<Block*>(block_cache->Value(cache_handle));
      } else {
        s = ReadBlock(table->rep_->file, options, handle, &contents);
        if (s.ok()) {
          block = new Block(contents);
          if (contents.cachable && options.fill_cache) {
            cache_handle = block_cache->Insert(key, block, block->size(),
                                               &DeleteCachedBlock);
          }
        }
      }
    } else {
      s = ReadBlock(table->rep_->file, options, handle, &contents);
      if (s.ok()) {
        block = new Block(contents);
      }
    }
  }

  ...
  return iter;
}
```

代码较长，分开来看，核心在

```c++
if (s.ok()) {
    BlockContents contents;
    if (block_cache != nullptr) {
      char cache_key_buffer[16];
      EncodeFixed64(cache_key_buffer, table->rep_->cache_id);
      EncodeFixed64(cache_key_buffer + 8, handle.offset());
      Slice key(cache_key_buffer, sizeof(cache_key_buffer));
      cache_handle = block_cache->Lookup(key);
      if (cache_handle != nullptr) {	// 在block cache中找到了block内容
        block = reinterpret_cast<Block*>(block_cache->Value(cache_handle));
      } else {		// 找不到block内容
         // 从table中读取block,注意此时blockcontents的内容已经是解压后的内容
        s = ReadBlock(table->rep_->file, options, handle, &contents);
        if (s.ok()) {
          block = new Block(contents);
          if (contents.cachable && options.fill_cache) {
            cache_handle = block_cache->Insert(key, block, block->size(),
                                               &DeleteCachedBlock);
          }
        }
      }
    } else {
      s = ReadBlock(table->rep_->file, options, handle, &contents);
      if (s.ok()) {
        block = new Block(contents);
      }
    }
  }
```

如果在block_cache中找到了相关的内容，则直接返回相应的Block。找不到，则从table中读取相应的Block并放入到block_cache中。可以看到，block cahce的key为一个16字节的buffer, 前8个字节存放的cache_id, 后8个字节存放的data block所在sstable中的offset。value为一个block（解压后）。

> cache id的作用：
>
> // Return a new numeric id.  May be used by multiple clients who are
>
> // sharing the same cache to partition the key space.  Typically the
>
> // client will allocate a new id at startup and prepend the id to
>
> // its cache keys.

额外看一下Table是如何ReadBlock的。

```c++
s = ReadBlock(table->rep_->file, options, handle, &contents);
```

### 3. Table的ReadBlock

```c++
Status ReadBlock(RandomAccessFile* file, const ReadOptions& options,
                 const BlockHandle& handle, BlockContents* result) {
  result->data = Slice();
  result->cachable = false;
  result->heap_allocated = false;

   // 读取完整的data block
  // Read the block contents as well as the type/crc footer.
  // See table_builder.cc for the code that built this structure.
  size_t n = static_cast<size_t>(handle.size());
  char* buf = new char[n + kBlockTrailerSize];
  Slice contents;
  Status s = file->Read(handle.offset(), n + kBlockTrailerSize, &contents, buf);
  if (!s.ok()) {
    delete[] buf;
    return s;
  }
  if (contents.size() != n + kBlockTrailerSize) {
    delete[] buf;
    return Status::Corruption("truncated block read");
  }

    // crc校验
  // Check the crc of the type and the block contents
  const char* data = contents.data();  // Pointer to where Read put the data
  if (options.verify_checksums) {
    const uint32_t crc = crc32c::Unmask(DecodeFixed32(data + n + 1));
    const uint32_t actual = crc32c::Value(data, n + 1);
    if (actual != crc) {
      delete[] buf;
      s = Status::Corruption("block checksum mismatch");
      return s;
    }
  }

   // 查看是否需要解压，如果需要解压，则解压数据
  switch (data[n]) {
    case kNoCompression:
      if (data != buf) {
        // File implementation gave us pointer to some other data.
        // Use it directly under the assumption that it will be live
        // while the file is open.
        delete[] buf;
        result->data = Slice(data, n);
        result->heap_allocated = false;
        result->cachable = false;  // Do not double-cache
      } else {
        result->data = Slice(buf, n);
        result->heap_allocated = true;
        result->cachable = true;
      }

      // Ok
      break;
    case kSnappyCompression: {
      size_t ulength = 0;
      if (!port::Snappy_GetUncompressedLength(data, n, &ulength)) {
        delete[] buf;
        return Status::Corruption("corrupted compressed block contents");
      }
      char* ubuf = new char[ulength];
      if (!port::Snappy_Uncompress(data, n, ubuf)) {
        delete[] buf;
        delete[] ubuf;
        return Status::Corruption("corrupted compressed block contents");
      }
      delete[] buf;
      result->data = Slice(ubuf, ulength);
      result->heap_allocated = true;
      result->cachable = true;
      break;
    }
    default:
      delete[] buf;
      return Status::Corruption("bad block type");
  }

  return Status::OK();
}

```