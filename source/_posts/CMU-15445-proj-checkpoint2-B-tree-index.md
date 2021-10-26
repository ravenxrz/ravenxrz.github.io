---
title: CMU-15445-proj2-checkpoint2(B+ tree index)
date: 2021-10-23 21:53:14
categories: CMU15445
tags:
---

## 1. 要求

checkpoint2 是在 checkpoint1的基础上，实现B+树的删除、迭代器和并发访问操作（并发访问只用支持point search， insert和delete，不用支持range scan)。

具体要求参见：[这里](https://15445.courses.cs.cmu.edu/fall2020/project2/)

总体来说，这部分相对第一部分会更难一些。

<!--more-->

## 2. 题解

### 2.1 删除操作

删除操作是B+树种最为复杂的操作，因为涉及的情况较多。具体情况可以分为2大类：叶子节点和非叶子节点的删除。每类又可以分为4种情况：左兄弟转移，右兄弟转移，左兄弟合并、右兄弟合并。强烈推荐查看[这篇文章](https://www.mathcs.emory.edu/~cheung/Courses/554/Syllabus/3-index/B-tree=delete.html)，应该是我在全网中看到的最详细的B+树删除过程了。

核心函数为以下三个：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20211024164944749.png" alt="image-20211024164944749" style="zoom:50%;" />

不过我在具体实现过程中，并没有使用 `Coalesce` 和 `Redistribute` 两个函数，而是自实现了以下几个函数:

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20211024165108398.png" alt="image-20211024165108398" style="zoom:50%;" />

简单描述下删除的算法步骤：

1. 判定树是否为空，如果为空，直接返回；否则进入第2步

2. 找到要删除的key可能存在的叶节点N，执行删除操作，如果删除无效（key不在该叶子节点中），直接返回；否则进入第3步；

3. 查看叶节点N当前剩余size是否大于等于最小size，如果大于等于，则直接返回；否则进入第4步；

4. 寻找左兄弟，如果有左兄弟，查看左兄弟是否可以转移一个key value到节点N中，如果不可以，查看右兄弟是否可以转移，转移完成后，更新父节点，返回；如果均不可以，进入第5步。

5. 寻找左兄弟，如果有左兄弟，则合并节点N到左兄弟中，更新父节点，如果父节点当前size 小于 最小size，则递归到第4步，如果不存在左兄弟，寻找右兄弟，如果有右兄弟，则合并右兄弟到节点N中，更新父节点，如果父节点当前size 小于 最小size，则递归到第4步；

   > Note: 如果一直递归到了root节点，由于root节点没有左右兄弟，则需要将root节点合并到孩子节点中。

写完后，可尝试是否能通过 `b_plus_tree_delete_test` 测试， **最好自己多写一些大规模数据量的测试，同时将`tree`的`internal node`和`leaf node`的最小`size`设置得尽量小（我是直接设置的3,3），因为这样可以造成更多的分裂，合并。**

### 2.2 Index iterator

这个就非常好写了，B+ tree暴露了三个函数给上游，分别为：

1. begin()
2. Begin(key), 这个相当于先seek
3. end()

而index iterator中，也只有少量几个函数。核心逻辑为先定位到合适的`leaf node`，然后可以使用一个索引游标`idx`，记录当前的位置，如果`idx`达到一个`node`（假设为`node i`)的最大`size`时，更新游标到下一个`node i+1`，同时释放`node i`。

至于 end()函数的设计，我的做法是 end() 是将当前索引到的 `node` 指针设定为 `nullptr`。

### 2.3 并发index

针对这个问题，最简单的方式是采用 “大锁”的方式， 对全局定义一个 `latch`, 然后针对 `Search`, `Insert`和 `Delete`操作上 `latch`, 这样能够保证同一个时刻只有一个线程在做操作。 不过这样的代价就是B+树的性能过低。 

> 我的建议是，先上大锁，然后查看是否能通过并发测试。
>
> 我在写时，B+树完成delete操作后通过了单线程下的所有默认测试，但是上大锁时并发测试居然没有过，所以强烈建议自己多写几个单线程下的测试，尽量做到较大数据量和 Internal page和 leaf page 的 min size较小。如果一切通过了，再考虑细粒度锁。

那如何提高性能？ 使用 **latch crabbing**. 详细可以参考讲义ppt和教材的914页。这里只贴一下官方的一个简单描述：

- `Search`: Starting with root page, grab read (**R**) latch on child Then release latch on parent as soon as you land on the child page.
- `Insert`: Starting with root page, grab write (**W**) latch on child. Once child is locked, check if it is safe, in this case, not full. If child is safe, release **all** locks on ancestors.
- `Delete`: Starting with root page, grab write (**W**) latch on child. Once child is locked, check if it is safe, in this case, at least half-full. (NOTE: for root page, we need to check with different standards) If child is safe, release **all** locks on ancestors.

拿 `Insert`操作来说， 既然在寻找到叶节点的过程中，可能存在加锁和释放锁，那肯定需要一个数据结构来存放之前已经加过锁的page， 这个数据结构存放在`Transcation`这个类中的`page_set_`, 如下图：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20211024171605862.png" alt="image-20211024171605862" style="zoom:50%;" />

*`delte_page_set_`是给删除操作使用的*

具体实现过程中，核心函数为 `FindLeafPage`， 在这个函数中，需要实现所有的加锁和释放锁过程，并把还未释放锁的 祖先节点们收集起来放到 `page_set_`中。问题是在从root节点到leaf节点的遍历过程中，什么时候能够释放祖先节点的锁？那就是判定当前所处的节点，是否`safe`。 什么是`safe`？不同的节点，有不同的的定义。我的`insert safe_checker`代码如下，注意`safe_checker`的实现与你之前的`Insert`操作和`Delete`操作强相关，所以我也不是标准答案：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20211024172045696.png" alt="image-20211024172045696" style="zoom:50%;" />

最后贴一下加锁和释放锁的核心代码：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20211024172236751.png" alt="image-20211024172236751" style="zoom: 50%;" />

另外，需要思考下`UnpinPage`和 `UnLatch`操作的先后顺序是什么, 其实官网上已经给出了答案：

> - Think carefully about the order and relationship between `UnpinPage(page_id, is_dirty)` method from buffer pool manager class and `UnLock()` methods from page class. You have to release the latch on that page **BEFORE** you unpin the same page from the buffer pool.

原因是如果Unpin在Unlatch之前，即使thread拿到page的latch，一旦page被Unpin，后续这个page很可能会被替换出去，那thread拿到的锁也就失去了意义，因为数据已经发生了改变。

## 3. 注意点记录

上面是整个寻找过程中的加锁，下面再说说具体实现insert和delete过程中，有哪些需要注意的地方：

1. root id的访问问题。由于root id是会在多个线程被访问，且可能被修改。所以首先要保证root id的访问修改是原子且对其它线程是可见的。这里可以通过一个`mutex`来保证(其实最好的做法个人是使用原子变量，但是不方便修改代码中已经默认写好的其他地方，所以采用了mutex)。但除此之外还是可能会存在问题，举个例子：当前root id=10， 存在thread 1执行查询操作， thread 2执行插入操作。 thread 2先被调度，但是在thread 2执行到一半时，切换到了thread 1，此时thread 1拿到root id=10的page，但是还未加上读锁。 此时线程再次切换到thread 2， 而thread 2由于插入操作，造成了原本 root id =10的根节点分裂，生成一个新的根节点，更新root id=11，现在thread 2执行完成，线程切换到thread 1， thread 1还是拿着旧的root id=10的page，加上读锁后执行后续search操作，此时的Search是错误的，因为不是从真正的根节点root id=11的page开始搜索。 那这个该如何解决？

   1. 我个人的做法是，首先记录一次`root` id为`old_root_id`，并获取page加上读锁后，再次比对 `old_root_id`是否等于当前的 `root_id`, 如果等于，则进行后操作，如果不等于，则释放page的锁，重试本次操作。这部分代码如下：

 ```c++
Page *BPLUSTREE_TYPE::FindLeafPageLock(const KeyType &key, OperationType op_type, Transaction *transaction) {
    /* 保存旧root page id */
    root_page_id_lck_.lock();
    int old_root_page_id = root_page_id_;
    root_page_id_lck_.unlock();
    /* 获取page，然后加锁，再double check root page id是否改变 */
    Page *cur_page = buffer_pool_manager_->FetchPage(old_root_page_id);
    if (op_type == OperationType::SEARCH) {
        cur_page->RLatch();
    } else {
        cur_page->WLatch();
    }
    /* 获取锁后，查看当前root_page_id是否和old_root_page_id相同 */
    root_page_id_lck_.lock();
    if (old_root_page_id != root_page_id_) {
        /* 由于其它线程的插入或者删除，造成了root_page 发生修改，所以现在需要重试 */
        buffer_pool_manager_->UnpinPage(cur_page->GetPageId(), false);
        root_page_id_lck_.unlock();
        if (op_type == OperationType::SEARCH) {
            cur_page->RUnlatch();
        } else {
            cur_page->WUnlatch();
        }
        return nullptr;		// 返回nullptr，让调用者重试
    }
    root_page_id_lck_.unlock();
    ...
}
 ```

  在调用`FindLeafPageLock`的地方，这样写：

```c++
/* find the leaf page that may contain the key */
LeafPage *leaf_page = nullptr;
while ((leaf_page = reinterpret_cast<LeafPage *>(FindLeafPageLock(key, OperationType::DELETE, transaction))) ==
       nullptr) {
}
```

 **另一种做法是增加虚拟节点，所有操作应该首先获取虚拟节点，再获取root节点，这样可以将对root节点操作和其它节点的操作统一起来，应该是一种更为精简优雅的做法，也是推荐的做法。**

   2. 删除操作时，兄弟节点也应该加锁！考虑如下场景：

      <img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20211024174634054.png" alt="image-20211024174634054" style="zoom:50%;" />
   
      T1执行插入，并加锁蓝色节点，T2执行删除操作，正在删除旁边的红色，由于红色节点删除元素后，size小于最小size，选择和蓝色节点合并，此时如果不对蓝色节点加锁，那么蓝色节点的数据将得不到一致性保证。T2很可能从T1中拿到错误的数据，这也会造成数据异常。

## 4. 全测试文件

默认本地的测试用例是非常简单的，很不方便调试，在加入知乎大佬给的群后，群文件有人抓取了gradescope的全测试用例，这里给出所有的相关用例，辅助读者调试：

1. `grading_b_plus_tree_checkpoint_2_sequential_test.cpp`

   ```c++
   /**
    * b_plus_tree_sequential_test.cpp
    */
   
   #include <algorithm>
   #include <cstdio>
   
   #include "b_plus_tree_test_util.h"  // NOLINT
   #include "buffer/buffer_pool_manager.h"
   #include "gtest/gtest.h"
   #include "storage/index/b_plus_tree.h"
   
   namespace bustub {
   
   /*
    * Score: 5
    * Description: The same test that has been run for checkpoint 1,
    * but added iterator for value checking
    */
   TEST(BPlusTreeTests, InsertTest1) {
     // create KeyComparator and index schema
     Schema *key_schema = ParseCreateStatement("a bigint");
     GenericComparator<8> comparator(key_schema);
   
     DiskManager *disk_manager = new DiskManager("test.db");
     BufferPoolManager *bpm = new BufferPoolManager(50, disk_manager);
     // create b+ tree
     BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator);
     GenericKey<8> index_key;
     RID rid;
     // create transaction
     Transaction *transaction = new Transaction(0);
   
     // create and fetch header_page
     page_id_t page_id;
     auto header_page = bpm->NewPage(&page_id);
     (void)header_page;
   
     std::vector<int64_t> keys = {1, 2, 3, 4, 5};
     for (auto key : keys) {
       int64_t value = key & 0xFFFFFFFF;
       rid.Set(static_cast<int32_t>(key >> 32), value);
       index_key.SetFromInteger(key);
       tree.Insert(index_key, rid, transaction);
     }
   
     std::vector<RID> rids;
     for (auto key : keys) {
       rids.clear();
       index_key.SetFromInteger(key);
       tree.GetValue(index_key, &rids);
       EXPECT_EQ(rids.size(), 1);
   
       int64_t value = key & 0xFFFFFFFF;
       EXPECT_EQ(rids[0].GetSlotNum(), value);
     }
   
     int64_t start_key = 1;
     int64_t current_key = start_key;
     for (auto pair : tree) {
       auto location = pair.second;
       EXPECT_EQ(location.GetPageId(), 0);
       EXPECT_EQ(location.GetSlotNum(), current_key);
       current_key = current_key + 1;
     }
   
     EXPECT_EQ(current_key, keys.size() + 1);
   
     bpm->UnpinPage(HEADER_PAGE_ID, true);
     delete key_schema;
     delete transaction;
     delete disk_manager;
     delete bpm;
     remove("test.db");
     remove("test.log");
   }
   
   /*
    * Score: 5
    * Description: The same test that has been run for checkpoint 1
    * but added iterator for value checking
    */
   TEST(BPlusTreeTests, InsertTest2) {
     // create KeyComparator and index schema
     Schema *key_schema = ParseCreateStatement("a bigint");
     GenericComparator<8> comparator(key_schema);
   
     DiskManager *disk_manager = new DiskManager("test.db");
     BufferPoolManager *bpm = new BufferPoolManager(50, disk_manager);
     // create b+ tree
     BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator);
     GenericKey<8> index_key;
     RID rid;
     // create transaction
     Transaction *transaction = new Transaction(0);
   
     // create and fetch header_page
     page_id_t page_id;
     auto header_page = bpm->NewPage(&page_id);
     (void)header_page;
   
     std::vector<int64_t> keys = {5, 4, 3, 2, 1};
     for (auto key : keys) {
       int64_t value = key & 0xFFFFFFFF;
       rid.Set(static_cast<int32_t>(key >> 32), value);
       index_key.SetFromInteger(key);
       tree.Insert(index_key, rid, transaction);
     }
   
     std::vector<RID> rids;
     for (auto key : keys) {
       rids.clear();
       index_key.SetFromInteger(key);
       tree.GetValue(index_key, &rids);
       EXPECT_EQ(rids.size(), 1);
   
       int64_t value = key & 0xFFFFFFFF;
       EXPECT_EQ(rids[0].GetSlotNum(), value);
     }
   
     int64_t start_key = 1;
     int64_t current_key = start_key;
     for (auto pair : tree) {
       auto location = pair.second;
       EXPECT_EQ(location.GetPageId(), 0);
       EXPECT_EQ(location.GetSlotNum(), current_key);
       current_key = current_key + 1;
     }
   
     EXPECT_EQ(current_key, keys.size() + 1);
   
     start_key = 3;
     current_key = start_key;
     index_key.SetFromInteger(start_key);
     for (auto iterator = tree.Begin(index_key); !iterator.isEnd(); ++iterator) {
       auto location = (*iterator).second;
       EXPECT_EQ(location.GetPageId(), 0);
       EXPECT_EQ(location.GetSlotNum(), current_key);
       current_key = current_key + 1;
     }
   
     bpm->UnpinPage(HEADER_PAGE_ID, true);
     delete key_schema;
     delete transaction;
     delete disk_manager;
     delete bpm;
     remove("test.db");
     remove("test.log");
   }
   
   /*
    * Score: 10
    * Description: Insert a set of keys, use GetValue and iterator to
    * check the the inserted keys. Then delete a subset of the keys.
    * Finally use the iterator to check the remained keys.
    */
   TEST(BPlusTreeTests, DeleteTest1) {
     // create KeyComparator and index schema
     std::string createStmt = "a bigint";
     Schema *key_schema = ParseCreateStatement(createStmt);
     GenericComparator<8> comparator(key_schema);
   
     DiskManager *disk_manager = new DiskManager("test.db");
     BufferPoolManager *bpm = new BufferPoolManager(50, disk_manager);
     // create b+ tree
     BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator);
     GenericKey<8> index_key;
     RID rid;
     // create transaction
     Transaction *transaction = new Transaction(0);
   
     // create and fetch header_page
     page_id_t page_id;
     auto header_page = bpm->NewPage(&page_id);
     (void)header_page;
   
     std::vector<int64_t> keys = {1, 2, 3, 4, 5};
     for (auto key : keys) {
       int64_t value = key & 0xFFFFFFFF;
       rid.Set(static_cast<int32_t>(key >> 32), value);
       index_key.SetFromInteger(key);
       tree.Insert(index_key, rid, transaction);
     }
   
     std::vector<RID> rids;
     for (auto key : keys) {
       rids.clear();
       index_key.SetFromInteger(key);
       tree.GetValue(index_key, &rids);
       EXPECT_EQ(rids.size(), 1);
   
       int64_t value = key & 0xFFFFFFFF;
       EXPECT_EQ(rids[0].GetSlotNum(), value);
     }
   
     int64_t start_key = 1;
     int64_t current_key = start_key;
     for (auto pair : tree) {
       auto location = pair.second;
       EXPECT_EQ(location.GetPageId(), 0);
       EXPECT_EQ(location.GetSlotNum(), current_key);
       current_key = current_key + 1;
     }
   
     EXPECT_EQ(current_key, keys.size() + 1);
   
     std::vector<int64_t> remove_keys = {1, 5};
     for (auto key : remove_keys) {
       index_key.SetFromInteger(key);
       tree.Remove(index_key, transaction);
     }
   
     start_key = 2;
     current_key = start_key;
     int64_t size = 0;
     for (auto pair : tree) {
       auto location = pair.second;
       EXPECT_EQ(location.GetPageId(), 0);
       EXPECT_EQ(location.GetSlotNum(), current_key);
       current_key = current_key + 1;
       size = size + 1;
     }
   
     EXPECT_EQ(size, 3);
   
     bpm->UnpinPage(HEADER_PAGE_ID, true);
     delete key_schema;
     delete transaction;
     delete disk_manager;
     delete bpm;
     remove("test.db");
     remove("test.log");
   }
   
   /*
    * Score: 10
    * Description: Similar to DeleteTest2, except that, during the Remove step,
    * a different subset of keys are removed.
    */
   TEST(BPlusTreeTests, DeleteTest2) {
     // create KeyComparator and index schema
     Schema *key_schema = ParseCreateStatement("a bigint");
     GenericComparator<8> comparator(key_schema);
   
     DiskManager *disk_manager = new DiskManager("test.db");
     BufferPoolManager *bpm = new BufferPoolManager(50, disk_manager);
     // create b+ tree
     BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator);
     GenericKey<8> index_key;
     RID rid;
     // create transaction
     Transaction *transaction = new Transaction(0);
   
     // create and fetch header_page
     page_id_t page_id;
     auto header_page = bpm->NewPage(&page_id);
     (void)header_page;
   
     std::vector<int64_t> keys = {1, 2, 3, 4, 5};
     for (auto key : keys) {
       int64_t value = key & 0xFFFFFFFF;
       rid.Set(static_cast<int32_t>(key >> 32), value);
       index_key.SetFromInteger(key);
       tree.Insert(index_key, rid, transaction);
     }
   
     std::vector<RID> rids;
     for (auto key : keys) {
       rids.clear();
       index_key.SetFromInteger(key);
       tree.GetValue(index_key, &rids);
       EXPECT_EQ(rids.size(), 1);
   
       int64_t value = key & 0xFFFFFFFF;
       EXPECT_EQ(rids[0].GetSlotNum(), value);
     }
   
     int64_t start_key = 1;
     int64_t current_key = start_key;
     index_key.SetFromInteger(start_key);
     for (auto pair : tree) {
       auto location = pair.second;
       EXPECT_EQ(location.GetPageId(), 0);
       EXPECT_EQ(location.GetSlotNum(), current_key);
       current_key = current_key + 1;
     }
   
     EXPECT_EQ(current_key, keys.size() + 1);
   
     std::vector<int64_t> remove_keys = {1, 5, 3, 4};
     for (auto key : remove_keys) {
       index_key.SetFromInteger(key);
       tree.Remove(index_key, transaction);
     }
   
     start_key = 2;
     current_key = start_key;
     int64_t size = 0;
     for (auto pair : tree) {
       auto location = pair.second;
       EXPECT_EQ(location.GetPageId(), 0);
       EXPECT_EQ(location.GetSlotNum(), current_key);
       current_key = current_key + 1;
       size = size + 1;
     }
   
     EXPECT_EQ(size, 1);
   
     bpm->UnpinPage(HEADER_PAGE_ID, true);
     delete key_schema;
     delete transaction;
     delete disk_manager;
     delete bpm;
     remove("test.db");
     remove("test.log");
   }
   
   /*
    * Score: 10
    * Description: Insert 10000 keys. Use GetValue and the iterator to iterate
    * through the inserted keys. Then remove 9900 inserted keys. Finally, use
    * the iterator to check the correctness of the remaining keys.
    */
   TEST(BPlusTreeTests, ScaleTest) {
     // create KeyComparator and index schema
     Schema *key_schema = ParseCreateStatement("a bigint");
     GenericComparator<8> comparator(key_schema);
   
     DiskManager *disk_manager = new DiskManager("test.db");
     BufferPoolManager *bpm = new BufferPoolManager(30, disk_manager);
     // create b+ tree
     BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator, 3, 3);
     GenericKey<8> index_key;
     RID rid;
     // create transaction
     Transaction *transaction = new Transaction(0);
     // create and fetch header_page
     page_id_t page_id;
     auto header_page = bpm->NewPage(&page_id);
     (void)header_page;
   
     int64_t scale = 10000;
     std::vector<int64_t> keys;
     for (int64_t key = 1; key < scale; key++) {
       keys.push_back(key);
     }
   
     for (auto key : keys) {
       int64_t value = key & 0xFFFFFFFF;
       rid.Set(static_cast<int32_t>(key >> 32), value);
       index_key.SetFromInteger(key);
       tree.Insert(index_key, rid, transaction);
     }
     std::vector<RID> rids;
     for (auto key : keys) {
       rids.clear();
       index_key.SetFromInteger(key);
       tree.GetValue(index_key, &rids);
       EXPECT_EQ(rids.size(), 1);
   
       int64_t value = key & 0xFFFFFFFF;
       EXPECT_EQ(rids[0].GetSlotNum(), value);
     }
   
     int64_t start_key = 1;
     int64_t current_key = start_key;
     for (auto pair : tree) {
       (void)pair;
       current_key = current_key + 1;
     }
     EXPECT_EQ(current_key, keys.size() + 1);
   
     int64_t remove_scale = 9900;
     std::vector<int64_t> remove_keys;
     for (int64_t key = 1; key < remove_scale; key++) {
       remove_keys.push_back(key);
     }
     // std::random_shuffle(remove_keys.begin(), remove_keys.end());
     for (auto key : remove_keys) {
       index_key.SetFromInteger(key);
       tree.Remove(index_key, transaction);
     }
   
     start_key = 9900;
     current_key = start_key;
     int64_t size = 0;
     index_key.SetFromInteger(start_key);
     for (auto pair : tree) {
       (void)pair;
       current_key = current_key + 1;
       size = size + 1;
     }
   
     EXPECT_EQ(size, 100);
   
     bpm->UnpinPage(HEADER_PAGE_ID, true);
     delete key_schema;
     delete transaction;
     delete disk_manager;
     delete bpm;
     remove("test.db");
     remove("test.log");
   }
   
   /*
    * Score: 10
    * Description: Insert a set of keys. Concurrently insert and delete
    * a different set of keys.
    * At the same time, concurrently get the previously inserted keys.
    * Check all the keys get are the same set of keys as previously
    * inserted.
    */
   TEST(BPlusTreeTests, SequentialMixTest) {
     // create KeyComparator and index schema
     Schema *key_schema = ParseCreateStatement("a bigint");
     GenericComparator<8> comparator(key_schema);
   
     DiskManager *disk_manager = new DiskManager("test.db");
     BufferPoolManager *bpm = new BufferPoolManager(50, disk_manager);
     // create b+ tree
     BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator);
     GenericKey<8> index_key;
     RID rid;
     // create transaction
     Transaction *transaction = new Transaction(0);
   
     // create and fetch header_page
     page_id_t page_id;
     auto header_page = bpm->NewPage(&page_id);
     (void)header_page;
     // first, populate index
     std::vector<int64_t> for_insert;
     std::vector<int64_t> for_delete;
     size_t sieve = 2;  // divide evenly
     size_t total_keys = 1000;
     for (size_t i = 1; i <= total_keys; i++) {
       if (i % sieve == 0) {
         for_insert.push_back(i);
       } else {
         for_delete.push_back(i);
       }
     }
   
     // Insert all the keys, including the ones that will remain at the end and
     // the ones that are going to be removed next.
     for (size_t i = 0; i < total_keys / 2; i++) {
       int64_t insert_key = for_insert[i];
       int64_t insert_value = insert_key & 0xFFFFFFFF;
       rid.Set(static_cast<int32_t>(insert_key >> 32), insert_value);
       index_key.SetFromInteger(insert_key);
       tree.Insert(index_key, rid, transaction);
   
       int64_t delete_key = for_delete[i];
       int64_t delete_value = delete_key & 0xFFFFFFFF;
       rid.Set(static_cast<int32_t>(delete_key >> 32), delete_value);
       index_key.SetFromInteger(delete_key);
       tree.Insert(index_key, rid, transaction);
     }
   
     // Remove the keys in for_delete
     for (auto key : for_delete) {
       index_key.SetFromInteger(key);
       tree.Remove(index_key, transaction);
     }
   
     // Only half of the keys should remain
     int64_t start_key = 2;
     int64_t size = 0;
     index_key.SetFromInteger(start_key);
     for (auto pair : tree) {
       EXPECT_EQ((pair.first).ToString(), for_insert[size]);
       size++;
     }
   
     EXPECT_EQ(size, for_insert.size());
   
     bpm->UnpinPage(HEADER_PAGE_ID, true);
     delete key_schema;
     delete transaction;
     delete disk_manager;
     delete bpm;
     remove("test.db");
     remove("test.log");
   }
   }  // namespace bustub
   ```

   2. `grading_b_plus_tree_checkpoint_2_concurrent_test.cpp`

   ```c++
   /**
    * grading_b_plus_tree_checkpoint_2_concurrent_test.cpp
    */
   
   #include <chrono>  // NOLINT
   #include <cstdio>
   #include <functional>
   #include <future>  // NOLINT
   #include <thread>  // NOLINT
   
   #include "b_plus_tree_test_util.h"  // NOLINT
   #include "buffer/buffer_pool_manager.h"
   #include "gtest/gtest.h"
   #include "storage/index/b_plus_tree.h"
   
   // Macro for time out mechanism
   #define TEST_TIMEOUT_BEGIN                           \
     std::promise<bool> promisedFinished;               \
     auto futureResult = promisedFinished.get_future(); \
                                 std::thread([](std::promise<bool>& finished) {
   #define TEST_TIMEOUT_FAIL_END(X)                                                                  \
     finished.set_value(true);                                                                       \
     }, std::ref(promisedFinished)).detach();                                                        \
     EXPECT_TRUE(futureResult.wait_for(std::chrono::milliseconds(X)) != std::future_status::timeout) \
         << "Test Failed Due to Time Out";
   
   namespace bustub {
   // helper function to launch multiple threads
   template <typename... Args>
   void LaunchParallelTest(uint64_t num_threads, uint64_t txn_id_start, Args &&...args) {
     std::vector<std::thread> thread_group;
   
     // Launch a group of threads
     for (uint64_t thread_itr = 0; thread_itr < num_threads; ++thread_itr) {
       thread_group.push_back(std::thread(args..., txn_id_start + thread_itr, thread_itr));
     }
   
     // Join the threads with the main thread
     for (uint64_t thread_itr = 0; thread_itr < num_threads; ++thread_itr) {
       thread_group[thread_itr].join();
     }
   }
   
   // helper function to insert
   void InsertHelper(BPlusTree<GenericKey<8>, RID, GenericComparator<8>> *tree, const std::vector<int64_t> &keys,
                     uint64_t tid, __attribute__((unused)) uint64_t thread_itr = 0) {
     GenericKey<8> index_key;
     RID rid;
     // create transaction
     Transaction *transaction = new Transaction(tid);
     for (auto key : keys) {
       int64_t value = key & 0xFFFFFFFF;
       rid.Set(static_cast<int32_t>(key >> 32), value);
       index_key.SetFromInteger(key);
       tree->Insert(index_key, rid, transaction);
     }
     delete transaction;
   }
   
   // helper function to seperate insert
   void InsertHelperSplit(BPlusTree<GenericKey<8>, RID, GenericComparator<8>> *tree, const std::vector<int64_t> &keys,
                          int total_threads, uint64_t tid, __attribute__((unused)) uint64_t thread_itr) {
     GenericKey<8> index_key;
     RID rid;
     // create transaction
     Transaction *transaction = new Transaction(tid);
     for (auto key : keys) {
       if (static_cast<uint64_t>(key) % total_threads == thread_itr) {
         int64_t value = key & 0xFFFFFFFF;
         rid.Set(static_cast<int32_t>(key >> 32), value);
         index_key.SetFromInteger(key);
         tree->Insert(index_key, rid, transaction);
       }
     }
     delete transaction;
   }
   
   // helper function to delete
   void DeleteHelper(BPlusTree<GenericKey<8>, RID, GenericComparator<8>> *tree, const std::vector<int64_t> &remove_keys,
                     uint64_t tid, __attribute__((unused)) uint64_t thread_itr = 0) {
     GenericKey<8> index_key;
     // create transaction
     Transaction *transaction = new Transaction(tid);
     for (auto key : remove_keys) {
       index_key.SetFromInteger(key);
       tree->Remove(index_key, transaction);
     }
     delete transaction;
   }
   
   // helper function to seperate delete
   void DeleteHelperSplit(BPlusTree<GenericKey<8>, RID, GenericComparator<8>> *tree,
                          const std::vector<int64_t> &remove_keys, int total_threads, uint64_t tid,
                          __attribute__((unused)) uint64_t thread_itr) {
     GenericKey<8> index_key;
     // create transaction
     Transaction *transaction = new Transaction(tid);
     for (auto key : remove_keys) {
       if (static_cast<uint64_t>(key) % total_threads == thread_itr) {
         index_key.SetFromInteger(key);
         tree->Remove(index_key, transaction);
       }
     }
     delete transaction;
   }
   
   void LookupHelper(BPlusTree<GenericKey<8>, RID, GenericComparator<8>> *tree, const std::vector<int64_t> &keys,
                     uint64_t tid, __attribute__((unused)) uint64_t thread_itr = 0) {
     Transaction *transaction = new Transaction(tid);
     GenericKey<8> index_key;
     RID rid;
     for (auto key : keys) {
       int64_t value = key & 0xFFFFFFFF;
       rid.Set(static_cast<int32_t>(key >> 32), value);
       index_key.SetFromInteger(key);
       std::vector<RID> result;
       bool res = tree->GetValue(index_key, &result, transaction);
       EXPECT_EQ(res, true);
       EXPECT_EQ(result.size(), 1);
       EXPECT_EQ(result[0], rid);
     }
     delete transaction;
   }
   
   const size_t NUM_ITERS = 100;
   
   void InsertTest1Call() {
     for (size_t iter = 0; iter < NUM_ITERS; iter++) {
       // create KeyComparator and index schema
       Schema *key_schema = ParseCreateStatement("a bigint");
       GenericComparator<8> comparator(key_schema);
   
       DiskManager *disk_manager = new DiskManager("test.db");
       BufferPoolManager *bpm = new BufferPoolManager(50, disk_manager);
       // create b+ tree
       BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator, 3, 3);
       // create and fetch header_page
       page_id_t page_id;
       auto header_page = bpm->NewPage(&page_id);
       (void)header_page;
       // keys to Insert
       std::vector<int64_t> keys;
       int64_t scale_factor = 100;
       for (int64_t key = 1; key < scale_factor; key++) {
         keys.push_back(key);
       }
       LaunchParallelTest(2, 0, InsertHelper, &tree, keys);
   
       std::vector<RID> rids;
       GenericKey<8> index_key;
       for (auto key : keys) {
         rids.clear();
         index_key.SetFromInteger(key);
         tree.GetValue(index_key, &rids);
         EXPECT_EQ(rids.size(), 1);
   
         int64_t value = key & 0xFFFFFFFF;
         EXPECT_EQ(rids[0].GetSlotNum(), value);
       }
   
       int64_t start_key = 1;
       int64_t current_key = start_key;
   
       for (auto &pair : tree) {
         auto location = pair.second;
         EXPECT_EQ(location.GetPageId(), 0);
         EXPECT_EQ(location.GetSlotNum(), current_key);
         current_key = current_key + 1;
       }
   
       EXPECT_EQ(current_key, keys.size() + 1);
   
       bpm->UnpinPage(HEADER_PAGE_ID, true);
       delete key_schema;
       delete disk_manager;
       delete bpm;
       remove("test.db");
       remove("test.log");
     }
   }
   
   void InsertTest2Call() {
     for (size_t iter = 0; iter < NUM_ITERS; iter++) {
       // create KeyComparator and index schema
       Schema *key_schema = ParseCreateStatement("a bigint");
       GenericComparator<8> comparator(key_schema);
   
       DiskManager *disk_manager = new DiskManager("test.db");
       BufferPoolManager *bpm = new BufferPoolManager(50, disk_manager);
       // create b+ tree
       BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator);
       // create and fetch header_page
       page_id_t page_id;
       auto header_page = bpm->NewPage(&page_id);
       (void)header_page;
       // keys to Insert
       std::vector<int64_t> keys;
       int64_t scale_factor = 1000;
       for (int64_t key = 1; key < scale_factor; key++) {
         keys.push_back(key);
       }
       LaunchParallelTest(2, 0, InsertHelperSplit, &tree, keys, 2);
   
       std::vector<RID> rids;
       GenericKey<8> index_key;
       for (auto key : keys) {
         rids.clear();
         index_key.SetFromInteger(key);
         tree.GetValue(index_key, &rids);
         EXPECT_EQ(rids.size(), 1);
   
         int64_t value = key & 0xFFFFFFFF;
         EXPECT_EQ(rids[0].GetSlotNum(), value);
       }
   
       int64_t start_key = 1;
       int64_t current_key = start_key;
   
       for (auto &pair : tree) {
         auto location = pair.second;
         EXPECT_EQ(location.GetPageId(), 0);
         EXPECT_EQ(location.GetSlotNum(), current_key);
         current_key = current_key + 1;
       }
   
       EXPECT_EQ(current_key, keys.size() + 1);
   
       bpm->UnpinPage(HEADER_PAGE_ID, true);
       delete key_schema;
       delete disk_manager;
       delete bpm;
       remove("test.db");
       remove("test.log");
     }
   }
   
   void DeleteTest1Call() {
     for (size_t iter = 0; iter < NUM_ITERS; iter++) {
       // create KeyComparator and index schema
       Schema *key_schema = ParseCreateStatement("a bigint");
       GenericComparator<8> comparator(key_schema);
   
       DiskManager *disk_manager = new DiskManager("test.db");
       BufferPoolManager *bpm = new BufferPoolManager(50, disk_manager);
       // create b+ tree
       BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator);
       // create and fetch header_page
       page_id_t page_id;
       auto header_page = bpm->NewPage(&page_id);
       (void)header_page;
       // sequential insert
       std::vector<int64_t> keys = {1, 2, 3, 4, 5};
       InsertHelper(&tree, keys, 1);
   
       std::vector<int64_t> remove_keys = {1, 5, 3, 4};
       LaunchParallelTest(2, 1, DeleteHelper, &tree, remove_keys);
   
       int64_t start_key = 2;
       int64_t current_key = start_key;
       int64_t size = 0;
   
       for (auto &pair : tree) {
         auto location = pair.second;
         EXPECT_EQ(location.GetPageId(), 0);
         EXPECT_EQ(location.GetSlotNum(), current_key);
         current_key = current_key + 1;
         size = size + 1;
       }
   
       EXPECT_EQ(size, 1);
   
       bpm->UnpinPage(HEADER_PAGE_ID, true);
       delete key_schema;
       delete disk_manager;
       delete bpm;
       remove("test.db");
       remove("test.log");
     }
   }
   
   void DeleteTest2Call() {
     for (size_t iter = 0; iter < NUM_ITERS; iter++) {
       // create KeyComparator and index schema
       Schema *key_schema = ParseCreateStatement("a bigint");
       GenericComparator<8> comparator(key_schema);
   
       DiskManager *disk_manager = new DiskManager("test.db");
       BufferPoolManager *bpm = new BufferPoolManager(50, disk_manager);
       // create b+ tree
       BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator);
       // create and fetch header_page
       page_id_t page_id;
       auto header_page = bpm->NewPage(&page_id);
       (void)header_page;
   
       // sequential insert
       std::vector<int64_t> keys = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
       InsertHelper(&tree, keys, 1);
   
       std::vector<int64_t> remove_keys = {1, 4, 3, 2, 5, 6};
       LaunchParallelTest(2, 1, DeleteHelperSplit, &tree, remove_keys, 2);
   
       int64_t start_key = 7;
       int64_t current_key = start_key;
       int64_t size = 0;
   
       for (auto &pair : tree) {
         auto location = pair.second;
         EXPECT_EQ(location.GetPageId(), 0);
         EXPECT_EQ(location.GetSlotNum(), current_key);
         current_key = current_key + 1;
         size = size + 1;
       }
   
       EXPECT_EQ(size, 4);
   
       bpm->UnpinPage(HEADER_PAGE_ID, true);
       delete key_schema;
       delete disk_manager;
       delete bpm;
       remove("test.db");
       remove("test.log");
     }
   }
   
   void MixTest1Call() {
     for (size_t iter = 0; iter < NUM_ITERS; iter++) {
       // create KeyComparator and index schema
       Schema *key_schema = ParseCreateStatement("a bigint");
       GenericComparator<8> comparator(key_schema);
   
       DiskManager *disk_manager = new DiskManager("test.db");
       BufferPoolManager *bpm = new BufferPoolManager(50, disk_manager);
       // create b+ tree
       BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator);
   
       // create and fetch header_page
       page_id_t page_id;
       auto header_page = bpm->NewPage(&page_id);
       (void)header_page;
       // first, populate index
       std::vector<int64_t> for_insert;
       std::vector<int64_t> for_delete;
       size_t sieve = 2;  // divide evenly
       size_t total_keys = 1000;
       for (size_t i = 1; i <= total_keys; i++) {
         if (i % sieve == 0) {
           for_insert.push_back(i);
         } else {
           for_delete.push_back(i);
         }
       }
       // Insert all the keys to delete
       InsertHelper(&tree, for_delete, 1);
   
       auto insert_task = [&](int tid) { InsertHelper(&tree, for_insert, tid); };
       auto delete_task = [&](int tid) { DeleteHelper(&tree, for_delete, tid); };
       std::vector<std::function<void(int)>> tasks;
       tasks.emplace_back(insert_task);
       tasks.emplace_back(delete_task);
       std::vector<std::thread> threads;
       size_t num_threads = 10;
       for (size_t i = 0; i < num_threads; i++) {
         threads.emplace_back(std::thread{tasks[i % tasks.size()], i});
       }
       for (size_t i = 0; i < num_threads; i++) {
         threads[i].join();
       }
   
       int64_t size = 0;
   
       for (auto &pair : tree) {
         EXPECT_EQ((pair.first).ToString(), for_insert[size]);
         size++;
       }
   
       EXPECT_EQ(size, for_insert.size());
   
       bpm->UnpinPage(HEADER_PAGE_ID, true);
       delete key_schema;
       delete disk_manager;
       delete bpm;
       remove("test.db");
       remove("test.log");
     }
   }
   
   void MixTest2Call() {
     for (size_t iter = 0; iter < NUM_ITERS; iter++) {
       // create KeyComparator and index schema
       Schema *key_schema = ParseCreateStatement("a bigint");
       GenericComparator<8> comparator(key_schema);
   
       DiskManager *disk_manager = new DiskManager("test.db");
       BufferPoolManager *bpm = new BufferPoolManager(50, disk_manager);
       // create b+ tree
       BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator);
       // create and fetch header_page
       page_id_t page_id;
       auto header_page = bpm->NewPage(&page_id);
       (void)header_page;
   
       // Add perserved_keys
       std::vector<int64_t> perserved_keys;
       std::vector<int64_t> dynamic_keys;
       size_t total_keys = 3000;
       size_t sieve = 5;
       for (size_t i = 1; i <= total_keys; i++) {
         if (i % sieve == 0) {
           perserved_keys.push_back(i);
         } else {
           dynamic_keys.push_back(i);
         }
       }
       InsertHelper(&tree, perserved_keys, 1);
       // Check there are 1000 keys in there
       size_t size;
   
       auto insert_task = [&](int tid) { InsertHelper(&tree, dynamic_keys, tid); };
       auto delete_task = [&](int tid) { DeleteHelper(&tree, dynamic_keys, tid); };
       auto lookup_task = [&](int tid) { LookupHelper(&tree, perserved_keys, tid); };
   
       std::vector<std::thread> threads;
       std::vector<std::function<void(int)>> tasks;
       tasks.emplace_back(insert_task);
       tasks.emplace_back(delete_task);
       tasks.emplace_back(lookup_task);
   
       size_t num_threads = 6;
       for (size_t i = 0; i < num_threads; i++) {
         threads.emplace_back(std::thread{tasks[i % tasks.size()], i});
       }
       for (size_t i = 0; i < num_threads; i++) {
         threads[i].join();
       }
   
       // Check all reserved keys exist
       size = 0;
   
       for (auto &pair : tree) {
         if ((pair.first).ToString() % sieve == 0) {
           size++;
         }
       }
   
       EXPECT_EQ(size, perserved_keys.size());
   
       bpm->UnpinPage(HEADER_PAGE_ID, true);
       delete key_schema;
       delete disk_manager;
       delete bpm;
       remove("test.db");
       remove("test.log");
     }
   }
   
   void MixTest3Call() {
     for (size_t iter = 0; iter < NUM_ITERS; iter++) {
       // create KeyComparator and index schema
       Schema *key_schema = ParseCreateStatement("a bigint");
       GenericComparator<8> comparator(key_schema);
   
       DiskManager *disk_manager = new DiskManager("test.db");
       BufferPoolManager *bpm = new BufferPoolManager(50, disk_manager);
       // create b+ tree
       BPlusTree<GenericKey<8>, RID, GenericComparator<8>> tree("foo_pk", bpm, comparator);
   
       // create and fetch header_page
       page_id_t page_id;
       auto header_page = bpm->NewPage(&page_id);
       (void)header_page;
       // first, populate index
       std::vector<int64_t> for_insert;
       std::vector<int64_t> for_delete;
       size_t total_keys = 1000;
       for (size_t i = 1; i <= total_keys; i++) {
         if (i > 500) {
           for_insert.push_back(i);
         } else {
           for_delete.push_back(i);
         }
       }
       // Insert all the keys to delete
       InsertHelper(&tree, for_delete, 1);
   
       auto insert_task = [&](int tid) { InsertHelper(&tree, for_insert, tid); };
       auto delete_task = [&](int tid) { DeleteHelper(&tree, for_delete, tid); };
       std::vector<std::function<void(int)>> tasks;
       tasks.emplace_back(insert_task);
       tasks.emplace_back(delete_task);
       std::vector<std::thread> threads;
       size_t num_threads = 10;
       for (size_t i = 0; i < num_threads; i++) {
         threads.emplace_back(std::thread{tasks[i % tasks.size()], i});
       }
       for (size_t i = 0; i < num_threads; i++) {
         threads[i].join();
       }
   
       int64_t size = 0;
   
       for (auto &pair : tree) {
         EXPECT_EQ((pair.first).ToString(), for_insert[size]);
         size++;
       }
   
       EXPECT_EQ(size, for_insert.size());
   
       bpm->UnpinPage(HEADER_PAGE_ID, true);
       delete key_schema;
       delete disk_manager;
       delete bpm;
       remove("test.db");
       remove("test.log");
     }
   }
   
   /*
    * Score: 5
    * Description: Concurrently insert a set of keys.
    */
   TEST(BPlusTreeConcurrentTest, InsertTest1) {
     TEST_TIMEOUT_BEGIN
     InsertTest1Call();
     remove("test.db");
     remove("test.log");
     TEST_TIMEOUT_FAIL_END(1000 * 600)
   }
   
   /*
    * Score: 5
    * Description: Split the concurrent insert test to multiple threads
    * without overlap.
    */
   TEST(BPlusTreeConcurrentTest, InsertTest2) {
     TEST_TIMEOUT_BEGIN
     InsertTest2Call();
     remove("test.db");
     remove("test.log");
     TEST_TIMEOUT_FAIL_END(1000 * 600)
   }
   
   /*
    * Score: 5
    * Description: Concurrently delete a set of keys.
    */
   TEST(BPlusTreeConcurrentTest, DeleteTest1) {
     TEST_TIMEOUT_BEGIN
     DeleteTest1Call();
     remove("test.db");
     remove("test.log");
     TEST_TIMEOUT_FAIL_END(1000 * 600)
   }
   
   /*
    * Score: 5
    * Description: Split the concurrent delete task to multiple threads
    * without overlap.
    */
   TEST(BPlusTreeConcurrentTest, DeleteTest2) {
     TEST_TIMEOUT_BEGIN
     DeleteTest2Call();
     remove("test.db");
     remove("test.log");
     TEST_TIMEOUT_FAIL_END(1000 * 600)
   }
   
   /*
    * Score: 5
    * Description: First insert a set of keys.
    * Then concurrently delete those already inserted keys and
    * insert different set of keys. Check if all old keys are
    * deleted and new keys are added correctly.
    */
   TEST(BPlusTreeConcurrentTest, MixTest1) {
     TEST_TIMEOUT_BEGIN
     MixTest1Call();
     remove("test.db");
     remove("test.log");
     TEST_TIMEOUT_FAIL_END(1000 * 600)
   }
   
   /*
    * Score: 5
    * Description: Insert a set of keys. Concurrently insert and delete
    * a differnt set of keys.
    * At the same time, concurrently get the previously inserted keys.
    * Check all the keys get are the same set of keys as previously
    * inserted.
    */
   TEST(BPlusTreeConcurrentTest, MixTest2) {
     TEST_TIMEOUT_BEGIN
     MixTest2Call();
     remove("test.db");
     remove("test.log");
     TEST_TIMEOUT_FAIL_END(1000 * 600)
   }
   
   /*
    * Score: 5
    * Description: First insert a set of keys.
    * Then concurrently delete those already inserted keys and
    * insert different set of keys. Check if all old keys are
    * deleted and new keys are added correctly.
    */
   TEST(BPlusTreeConcurrentTest, MixTest3) {
     TEST_TIMEOUT_BEGIN
     MixTest3Call();
     remove("test.db");
     remove("test.log");
     TEST_TIMEOUT_FAIL_END(1000 * 600)
   }
   
   }  // namespace bustub
   ```

## 5. 总结

整个proj2原本计划两三天搞定，没想到在参考博文代码的基础上，checkpoint1就两天整，checkpoint2又是2天半，前前后后差不多一个星期就没了。实验过程中还是非常有收获的，比如以前从没写过B+树，更没写过会和disk交互的数据结构（以前所有学的数据结构都在内存中），对于并发的调试也多少有了点经验（这里感谢师兄的帮助与建议）。

额外给一点debug的建议：

1. 充分利用好能够绘制出来的 dot file来可视化B+树
2. 在多线程调试时，尽量找到最小的引起错误的数据量和线程数。 比如我在测试MixTest时，最开始的数据量是`1000*10`（操作数据量是1000， 操作线程数是10）， 后来降到了 `20*2`, 然后通过 dot file 可视化推导出问题大致所在，最后通过一定的printf打印log，最终找到了bug。
3. 多线程下发生错误时，可以尝试先使用大锁查看测试是否通过，再降低锁的粒度。如果大锁测试通过，小锁出现问题，那可能是加锁位置出现问题，此时采用gdb多线程调试其实意义不一定大，反而通过LOG分析更为有效。

最后，记录下checkpoint2满分：

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20211024182440733.png" alt="image-20211024182440733" style="zoom:50%;" />