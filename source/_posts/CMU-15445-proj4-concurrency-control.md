---
title: CMU-15445-proj4(Concurrency Control)
date: 2022-01-27 15:54:14
categories: CMU15445
tags:
---



## 0. 要求

文本为 CMU15445 - Proj4 的题解 。Proj4 为Concurrency Control，本次proj分为三个任务：

1. Lock Manager
2. Deadlock Detection
3. Concurrent Query Execution

总体来说，难度低于proj2， 高于proj3和proj1。不过还是有不少坑。下面简单来说说。

原文要求：https://15445.courses.cs.cmu.edu/fall2020/project4/

<!--more-->

## 1. TASK #1 - LOCK MANAGER

Task1要求实现一个 Lock Manager，并且支持不同的隔离级别（对proj来说，需要支持 RR, RC和RU三个级别）。这个LockManager会在 `TableHeap` 和 `Executor` 中被使用到。

具体来说，需要实现 `LockManager` 类中的如下接口：

- `LockShared(Transaction, RID)`: Transaction **txn** tries to take a shared lock on record id **rid**. This should be blocked on waiting and should return true when granted. Return false if transaction is rolled back (aborts).
- `LockExclusive(Transaction, RID)`: Transaction **txn** tries to take an exclusive lock on record id **rid**. This should be blocked on waiting and should return true when granted. Return false if transaction is rolled back (aborts).
- `LockUpgrade(Transaction, RID)`: Transaction **txn** tries to upgrade a shared to exclusive lock on record id **rid**. This should be blocked on waiting and should return true when granted. Return false if transaction is rolled back (aborts). This should also abort the transaction and return false if another transaction is already waiting to upgrade their lock.
- `Unlock(Transaction, RID)`: Unlock the record identified by the given record id that is held by the transaction.

现在简要分析下如何实现这个Task。 

既然要区分不同的隔离级别，那就要知道在不同隔离级别下，LM的behavior是如何的。S,X lock和unlock的总结如下：

> 事务隔离级别0（READ UNCOMMITTED） 
> 	1）事务T在修改数据前必须先对其加X锁，直到事务结束才释放。事务结束可以是正常结束（Commit）或非正常结束（Rollback）。
> 	2）事务T读数据不加锁
>
> 事务隔离级别1（READ COMMITTED）：
> 	1）事务T在修改数据前必须先对其加X锁，直到事务结束才释放。
> 	2）事务T读数据前必须先对其加S锁，读完后即释放S锁，而不是到事务结束才释放
>
> 事务隔离级别2（REPEATABLE READ)
> 	1）事务T在修改数据前必须先对其加X锁，直到事务结束才释放。
> 	2）事务T读数据前必须先对其加S锁，直到事务结束才释放。

除了LockShared和LockExclusive，Unlock接口外，还需要实现LockUpgrade接口，什么时候会用到这个接口呢？ 一个简单的场景是， 在proj3中（Query Executors)在执行 UPDATE 操作时，首先要通过 scan_executor 获取到某个tuple的S lock， 然后再在这个tuple上执行UPDATE, 执行应该将S lock升级到X lock。

除此外，还需要了解什么是 2 Phase-Lock, 一个2 Phase-Lock 的示意图如下：

![image-20220127172155885](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220127172155885.png)

>  也就是说，一旦一个txn开始unlock，那它就进入到了 SHRINKING 阶段，不能再获取锁。

下图为LockManager维护的加锁内部数据结构 `lock_table_`。

![cmu15445-proj4-hashmap](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/cmu15445-proj4-hashmap.svg)

`lock_table_` 是一个 RID->list 的映射，也就说每个rid对应到一个加速队列， 队列中中存放的是当前已经获取到锁的事务请求（包括事务号和加锁类型）。

###  LockShared

当一个txn想要获取S lock on rid，步骤如下：

1. txn状态是否已经是ABORTED, 若是，此时不应该再获取任何锁。

2. txn状态是否已经是 SHRINKING， 若是，此时不应该再获取任何锁。

3. txn的隔离级别是否为 RU, 若是，RU无需获取S lock。

4. txn当前是否已经获取了X lock或者S lock。如果已经获取了，则无需再获取S lock。

5. 尝试获取锁，如果当前 rid 加锁队列中存在其它txn的 X lock，则需等待。 

   实际上，在我个人的实现中，如果存在以下两种情况也需要等待。

   1. 如果在之前存在其它txn正在获取X lock（但还未能获取）
   2. 如果在之前存在其它txn正在upgrading lock（但还未能upgrade）

   这是为了预防上面两种txn饿死。 

6. 成功获取到锁，将本txn加入到rid的加锁队列，同时在更新txn的SharedLockSet。

下面是具体实现：

```c++
bool LockManager::LockShared(Transaction *txn, const RID &rid) {
  if (txn->GetState() == TransactionState::ABORTED) {
    return false;
  }
  if (txn->GetState() == TransactionState ::SHRINKING) {
    txn->SetState(TransactionState::ABORTED);
    throw TransactionAbortException(txn->GetTransactionId(), AbortReason::LOCK_ON_SHRINKING);
  }
  if (txn->GetIsolationLevel() == IsolationLevel::READ_UNCOMMITTED) {
    /* read uncommitted doesn't need any S lock */
    return false;
  }
  if (txn->IsSharedLocked(rid) ||
      txn->IsExclusiveLocked(
          rid)) { /* exclusive lock is larger than shared lock, so we think exclusive lock as a shared lock */
    return true;
  }

  std::unique_lock<std::mutex> lck(latch_);
  LockRequest lck_req(txn->GetTransactionId(), LockMode::SHARED);
  auto &req_queue = lock_table_[rid];
  req_queue.waitting_set_.insert(lck_req);
  txn_id_to_rid_[txn->GetTransactionId()] = rid; /* used for deadlock detection */
  req_queue.cv_.wait(lck, [&req_queue, txn]() {
    /* no exclusive lock exist */
    return txn->GetState() == TransactionState::ABORTED ||
           (!req_queue.upgrading_ && /* 防止upgrading饿死 */
            std::all_of(req_queue.waitting_set_.begin(), req_queue.waitting_set_.end(),
                        [](const LockRequest &x) { return x.lock_mode_ == LockMode::SHARED; }) && /* 防止writer饿死 */
            std::all_of(req_queue.request_queue_.begin(), req_queue.request_queue_.end(),
                        [](const LockRequest &x) { return x.lock_mode_ == LockMode::SHARED; }));
  });
  if (txn->GetState() == TransactionState::ABORTED) {
    txn_id_to_rid_.erase(txn->GetTransactionId());
    req_queue.waitting_set_.erase(lck_req);
    throw TransactionAbortException(txn->GetTransactionId(), AbortReason::DEADLOCK);
  }
  txn_id_to_rid_.erase(txn->GetTransactionId());
  req_queue.waitting_set_.erase(lck_req);
  req_queue.request_queue_.push_back(lck_req);

  /* now we get the shareed lock, put it into txn's shared lock set  */
  txn->GetSharedLockSet()->emplace(rid);
  return true;
}
```

其中 waitting_set_ 成员变量为个人所加：

```c++
  class LockRequestQueue {
   public:
    std::list<LockRequest> request_queue_;
    std::set<LockRequest> waitting_set_;
    std::condition_variable cv_;  // for notifying blocked transactions on this rid
    bool upgrading_ = false;
  };
```

所以在我的实现中，`lock_table_` 为：

![image-20220127204334231](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220127204334231.png)

当某个txn想要获取lock时，首先加入到waitting_set 中，只有真正获取到锁时，才加入到 request_queue_中， 加入 waitting_set的目的主要是为了防止发生写饥饿，也是为了后面upgrade更容易实现而准备。

### LockExclusive

当一个txn想要获取X lock on rid，步骤如下：

1. txn状态是否已经是ABORTED, 若是，此时不应该再获取任何锁。
2. txn状态是否已经是 SHRINKING， 若是，此时不应该再获取任何锁。
3. txn当前是否已经获取了X lock。如果已经获取了，则无需再获取 lock。
4. 尝试获取锁，如果当前 rid 加锁队列中存在其他任何txn lock，则等待。
5. 成功获取到锁，将本txn加入到rid的加锁队列，同时在更新txn的ExclusiveLockSet。

具体代码如下：

```c++

bool LockManager::LockExclusive(Transaction *txn, const RID &rid) {
  if (txn->GetState() == TransactionState::ABORTED) {
    return false;
  }
  if (txn->GetState() == TransactionState ::SHRINKING) {
    txn->SetState(TransactionState::ABORTED);
    throw TransactionAbortException(txn->GetTransactionId(), AbortReason::LOCK_ON_SHRINKING);
  }
  if (txn->IsExclusiveLocked(rid)) {
    return true;
  }

  std::unique_lock<std::mutex> lck(latch_);
  LockRequest lck_req(txn->GetTransactionId(), LockMode::EXCLUSIVE);
  auto &req_queue = lock_table_[rid];
  req_queue.waitting_set_.insert(lck_req);
  txn_id_to_rid_[txn->GetTransactionId()] = rid;    /* used for deadlock detection */
  req_queue.cv_.wait(lck, [&req_queue, txn]() {
    return txn->GetState() == TransactionState::ABORTED || req_queue.request_queue_.empty();
  });
  if (txn->GetState() == TransactionState::ABORTED) {
    txn_id_to_rid_.erase(txn->GetTransactionId());
    req_queue.waitting_set_.erase(lck_req);
    throw TransactionAbortException(txn->GetTransactionId(), AbortReason::DEADLOCK);
  }
  txn_id_to_rid_.erase(txn->GetTransactionId());
  req_queue.waitting_set_.erase(lck_req);
  req_queue.request_queue_.push_back(lck_req);

  /* now we get the shareed lock, put it into txn's shared lock set  */
  txn->GetExclusiveLockSet()->emplace(rid);
  txn->SetState(TransactionState::GROWING);

  return true;
}
```

### LockUpgrade

当一个txn想要upgrade S lock on rid，步骤如下：

1. txn状态是否已经是ABORTED, 若是，此时不能upgrade
2. txn状态是否已经是 SHRINKING， 若是，此时不能upgrade
3. txn是否已经获取了X lock，如是，直接返回true。
4. 是否已经有其他txn在本rid上upgrading，如果有，此时不能upgrade。
5. 开始upgrading，设置rid加锁队列的状态为upgrading， 并等待rid加锁队列中只剩当前txn request为止。
6. upgrade S lock到 X lock，并更新txn的 shared lock set和exclusive lock set。

具体代码如下：

```c++
bool LockManager::LockUpgrade(Transaction *txn, const RID &rid) {
  if (txn->GetState() == TransactionState::ABORTED) {
    return false;
  }
  if (txn->GetState() == TransactionState ::SHRINKING) {
    txn->SetState(TransactionState::ABORTED);
    throw TransactionAbortException(txn->GetTransactionId(), AbortReason::LOCK_ON_SHRINKING);
  }
  if (txn->IsExclusiveLocked(rid)) {
    return true; /* no need to upgrade */
  }
  // LOG_DEBUG("txn shared lock: %d", txn->IsSharedLocked(rid));
  // LOG_DEBUG("txn exclusive lock: %d", txn->IsExclusiveLocked(rid));
  assert(txn->IsSharedLocked(rid));

  std::unique_lock<std::mutex> lck(latch_);
  auto &req_queue = lock_table_[rid];
  if (req_queue.upgrading_) {
    txn->SetState(TransactionState::ABORTED);
    throw TransactionAbortException(txn->GetTransactionId(), AbortReason::UPGRADE_CONFLICT);
  }
  req_queue.upgrading_ = true;
  LockRequest lck_req(txn->GetTransactionId(), LockMode::EXCLUSIVE);
  txn_id_to_rid_[txn->GetTransactionId()] = rid;
  /* wait until queue size = 1 */
  req_queue.cv_.wait(lck, [&req_queue, txn] {
    return txn->GetState() == TransactionState::ABORTED || req_queue.request_queue_.size() == 1;
  });
  if (txn->GetState() == TransactionState::ABORTED) {
    txn_id_to_rid_.erase(txn->GetTransactionId());
    throw TransactionAbortException(txn->GetTransactionId(), AbortReason::DEADLOCK);
  }

  txn_id_to_rid_.erase(txn->GetTransactionId());
  assert(req_queue.request_queue_.front().txn_id_ == txn->GetTransactionId());
  lock_table_[rid].request_queue_.front().lock_mode_ = LockMode::EXCLUSIVE;
  lock_table_[rid].upgrading_ = false;
  /* update lock set of txn */
  txn->GetSharedLockSet()->erase(rid);
  txn->GetExclusiveLockSet()->emplace(rid);
  txn->SetState(TransactionState::GROWING);
  return true;
}
```

### Unlock

unlock比较简单，不赘述，贴出代码：

```c++
bool LockManager::Unlock(Transaction *txn, const RID &rid) {
  if (!txn->IsSharedLocked(rid) && !txn->IsExclusiveLocked(rid)) {
    return true; /* no need to unlock */
  }
  /* rc shouldn't get shared lock */
  assert(!(txn->GetIsolationLevel() == IsolationLevel::READ_UNCOMMITTED && txn->IsSharedLocked(rid)));
  std::unique_lock<std::mutex> lck(latch_);
  auto &req_queue = lock_table_[rid].request_queue_;
  assert(!req_queue.empty());
  req_queue.pop_front();
  lock_table_[rid].cv_.notify_all(); /* notify all waitting threads */
  /* change txn state to shrinking if txn isolation is "RR"! */
  if (txn->GetState() != TransactionState::ABORTED && txn->GetState() != TransactionState::COMMITTED) {
    if (txn->GetIsolationLevel() ==
        IsolationLevel::REPEATABLE_READ) { /* RR mode unlocks all locks until txn ends, so no other locks can be get */
      txn->SetState(TransactionState::SHRINKING);
    }
  }
  txn->GetSharedLockSet()->erase(rid);
  txn->GetExclusiveLockSet()->erase(rid);
  return true;
}
```

需要注意的地方有两点：

1. RU 隔离级别不会获取S lock，所以我加了assert。
2. RR模式只要Unlock，设置txn State为SHRINKING，RC和RU则应该在Unlock X lock才设置状态为SHRINKING。

以上为Task 1的个人解法。 感觉这部分写得有点杂乱。

## 2. DEADLOCK DETECTION

Task2实现死锁检测算法，通过开启一个专用死锁检测线程，在后台定时检查全局是否发生死锁。 这个过程中需要使用到 wait-for graph. wait-for graph

**在 wait-for graph中， node代表txn， edge： 从Ti 到 Tj的edge， 代表Ti正在等待Tj释放某个lock。**

下面是一个wait-for graph的示意图：

![image-20220127192432964](https://pic.imgdb.cn/item/61f293452ab3f51d91e009f7.png)

这个任务中，我们需要具体的API为：

The graph API you must implement and use for your cycle detection along with testing is the following:

- `AddEdge(txn_id_t t1, txn_id_t t2)`: Adds an edge in your graph from t1 to t2. If the edge already exists, you don't have to do anything.
- `RemoveEdge(txn_id_t t1, txn_id_t t2)`: Removes edge t1 to t2 from your graph. If no such edge exists, you don't have to do anything.
- `HasCycle(txn_id_t& txn_id)`: Looks for a cycle by using the **Depth First Search (DFS)** algorithm. If it finds a cycle, `HasCycle` should store the transaction id of the **youngest** transaction in the cycle in `txn_id` and return true. Your function should return the first cycle it finds. If your graph has no cycles, `HasCycle` should return false.
- `GetEdgeList()`: Returns a list of tuples representing the edges in your graph. We will use this to test correctness of your graph. A pair (t1,t2) corresponds to an edge from t1 to t2.
- `RunCycleDetection()`: Contains skeleton code for running cycle detection in the background. You should implement your cycle detection logic in here.

在进行死锁检测时，额外的要求为：

- Your DFS Cycle detection algorithm **must** be deterministic. In order to do achieve this, you must always choose to explore the lowest transaction id first. This means when choosing which **unexplored** node to run DFS from, always choose the node with the lowest transaction id. This also means when exploring neighbors, explore them in sorted order from lowest to highest.
- When you find a cycle, you should abort the **youngest** transaction to break the cycle by setting that transactions state to ABORTED.
- When your detection thread wakes up, it is responsible for breaking **all** cycles that exist. If you follow the above requirements, you will always find the cycles in a **deterministic** order. This also means that when you are building your graph, you should **not** add nodes for aborted transactions or draw edges to aborted transactions.

一句话总结为，使用dfs进行环检测，每次dfs选中的节点为 lowest transaction id的node， 每当发生死锁时，淘汰的是 youngest node。

现在来看看 为了构建 wait-for graph, 我们需要的container:

```
std::unordered_map<txn_id_t, std::vector<txn_id_t>> waits_for_;   
```

显然这是一个邻接链表的graph representation.

不过在我的实现中，我将 vector<txn_id_t> 改用了 set<txn_id_t> 因为这样每次dfs时，能够保证当前搜索到的 txn id是 lowest 的。

```
  std::map<txn_id_t, std::set<txn_id_t>> waits_for_; /* map + set is much better */
```

ok, 下面看看具体实现。

辅助函数相关：

```c++
void LockManager::AddEdge(txn_id_t t1, txn_id_t t2) {
  // LOG_DEBUG("Add edge(%d, %d)", t1, t2);
  waits_for_[t1].insert(t2);
}

bool LockManager::HasCycle(txn_id_t *txn_id) {
  std::vector<txn_id_t> traverse_stk;
  std::unordered_map<txn_id_t, bool> visited;
  while (true) {
    txn_id_t lowest_txn_id = 0x7fffffff;
    /* find the lowest txn_id in "wait_for" graph as the starting point */
    for (const auto &kv : waits_for_) {
      if (!visited[kv.first]) {
        // lowest_txn_id = std::min(lowest_txn_id, kv.first);
        lowest_txn_id = kv.first;
        break;
      }
    }
    if (lowest_txn_id == 0x7fffffff) { /* no unvisited node found, break */
      break;
    }
    /* cycle check from lowest txn id node */
    txn_id_t cur = lowest_txn_id;
    visited[cur] = true;
    traverse_stk.push_back(lowest_txn_id);
    while (!traverse_stk.empty()) {
      cur = traverse_stk.back();
      int old_cur = cur;
      const auto &linked_nodes = waits_for_[cur];
      for (txn_id_t node : linked_nodes) {
        if (std::find(traverse_stk.begin(), traverse_stk.end(), node) != traverse_stk.end()) {
          /* cycler found */
          *txn_id = *std::max_element(traverse_stk.begin(), traverse_stk.end());
          return true;
        }
        if (!visited[node]) {
          visited[node] = true;
          traverse_stk.push_back(node);
          cur = node;
          break;
        }
      }
      /* can't forward, pop from stack */
      if (cur == old_cur) {
        traverse_stk.pop_back();
      }
    }
  }
  return false;
}

std::vector<std::pair<txn_id_t, txn_id_t>> LockManager::GetEdgeList() {
  std::vector<std::pair<txn_id_t, txn_id_t>> edges;
  for (const auto &kv : waits_for_) {
    const auto &linked_nodes = kv.second;
    for (txn_id_t linked_node : linked_nodes) {
      edges.emplace_back(kv.first, linked_node);
    }
  }
  return edges;
}

void LockManager::BuildWaitforGraph() {
  for (const auto &rid_queue : lock_table_) {
    const auto &req_queue = rid_queue.second;
    /* Add edges */
    for (const auto &watting_txn : req_queue.waitting_set_) {
      for (const auto &granted_txn : req_queue.request_queue_) {
        AddEdge(watting_txn.txn_id_, granted_txn.txn_id_);
      }
    }
  }
}

void LockManager::RemoveEdge(txn_id_t t1, txn_id_t t2) { waits_for_[t1].erase(t2); }

void LockManager::RemoveEdges(txn_id_t txn_id) {
  auto linked_set = waits_for_[txn_id]; /* copy set first */
  for (txn_id_t linked_node : linked_set) {
    RemoveEdge(txn_id, linked_node);
  }
}
```

最后是 background thread的主要实现：

```c++
void LockManager::RunCycleDetection() {
  // TransactionManager txn_mgr(this);
  while (enable_cycle_detection_) {
    std::this_thread::sleep_for(cycle_detection_interval);
    {
      std::unique_lock<std::mutex> l(latch_);
      waits_for_.clear();
      /* Build Graph */
      BuildWaitforGraph();
      txn_id_t txn_id = -1;
      while (HasCycle(&txn_id)) {
        /* remove all edges linked to "txn_id" */
        RemoveEdges(txn_id);
        /* abort corresponding txn */
        Transaction *txn = TransactionManager::GetTransaction(txn_id);
        txn->SetState(TransactionState::ABORTED);
        assert(txn_id_to_rid_.count(txn_id) != 0U);
        lock_table_[txn_id_to_rid_[txn_id]].cv_.notify_all();
      }
    }
  }
```

## 3. CONCURRENT QUERY EXECUTION

Task3是在讲proj3 + proj4（上面实现的部分）联合起来，实现并发query execution，但是不用考虑 index 相关的execution, 要求修改如下文件：

- `src/execution/aggregation_executor.cpp`
- `src/execution/delete_executor.cpp`
- `src/execution/insert_executor.cpp`
- `src/execution/nested_loop_executor.cpp`
- `src/execution/seq_scan_executor.cpp`
- `src/execution/update_executor.cpp`

仔细思考一下， task 1和task 2 所实现的代码都是 tuple-level 的lock。 那上面几个executor中，哪些是会具体落实到tuple-level的。 显然 seq_scan_executor, update_executor, insert_executor, delete_executor这几个都会实际操作到tuple，而 aggregation_executor 和 nested_loop_executor 并不会直接操作 tuple, 这两个底层都需要依赖其他 exectuor， 所以我们在修改时，只用修改 seq_scan_executor, update_executor, insert_executor, delete_executor这几个.

### seq scan executor

seq_scan_executor 实际上是最复杂的，但是也是第一个必须实现的exectuor。 

首先seq_scan要考虑不同的隔离级别：

- RU，无需加S lock
- RC, 需要加S lock，但是读完即可释放
- RR， 需要加S lock，但是需要整个txn结束才释放。 (Strict-2PL)

除此外，seq_scan 加锁还需要double check, 也即对rid加锁后，还需要验证rid对应tuple是否还存在。 考虑的场景是，当seq_scan在尝试对rid加锁时，未能成功加锁，因为此时delete_executor成功到X lock，然后删除了该rid对应的tuple，而这个过程对scan executor是无感的，所以 scan_executor需要double check.

具体实现代码如下：

```c++
bool SeqScanExecutor::TryLock(RID rid) {
  LockManager *lck_mgr = exec_ctx_->GetLockManager();
  Transaction *txn = exec_ctx_->GetTransaction();
  try {
    if (txn->GetIsolationLevel() == IsolationLevel::READ_UNCOMMITTED) {
      /* do nothing */
      return true; /* assume no lock means get lock already */
    }
    if (txn->GetIsolationLevel() == IsolationLevel::READ_COMMITTED ||
        txn->GetIsolationLevel() == IsolationLevel::REPEATABLE_READ) {
      return lck_mgr->LockShared(txn, rid);
    }
  } catch (const TransactionAbortException &e) {
    return false;
  }

  LOG_ERROR("unknown isolation level");
  abort();
  return false;
}

void SeqScanExecutor::Unlock() {
  LockManager *lck_mgr = exec_ctx_->GetLockManager();
  Transaction *txn = exec_ctx_->GetTransaction();
  if (txn->GetIsolationLevel() == IsolationLevel::READ_UNCOMMITTED) {
    /* do nothing */
  } else if (txn->GetIsolationLevel() == IsolationLevel::READ_COMMITTED) {
    /* copy lock set */
    auto shared_lock_set = *(txn->GetSharedLockSet().get());
    for (auto rid : shared_lock_set) {
      lck_mgr->Unlock(txn, rid);
      // LOG_DEBUG("scan release lock %s", rid.ToString().c_str());
    }
  } else if (txn->GetIsolationLevel() == IsolationLevel::REPEATABLE_READ) {
    /* unlock only until committed or aborted */
    /* executor doesn't care about this, client txn manager does! */
  } else {
    LOG_ERROR("unknown isolation level");
    abort();
  }
}

bool SeqScanExecutor::Next(Tuple *tuple, RID *rid) {
  /*
      1. RU，无需lock
      2. RC, 每次读取前加锁，读取一次完后，解锁
      3. RR, 每次读取前枷锁，txn结束，解锁
   */
  /* reach the end of iterator */
  if (table_iter_ == table_heap_->End()) {
    Unlock();
    return false;
  }
  Tuple dummy_tuple; /* only used for double check */
  bool double_check_pass = false;
  do {
    /* filter tuples on which predicating is false */
    if (plan_->GetPredicate() != nullptr) {
      while (table_iter_ != table_heap_->End() &&
             !plan_->GetPredicate()->Evaluate(&(*table_iter_), schema_).GetAs<bool>()) {
        table_iter_++;
      }
      /* reach the end */
      if (table_iter_ == table_heap_->End()) {
        Unlock();
        return false;
      }
    }
    *rid = (*table_iter_).GetRid();
    if (!TryLock(*rid)) {
      exec_ctx_->GetTransaction()->SetState(TransactionState::ABORTED);
      return false;
    }
    // LOG_DEBUG("scan get lock %s", rid->ToString().c_str());
    /* format output schema */
    table_iter_++;
    Unlock();
    *tuple = GenerateFinalTuple(*table_iter_, plan_->OutputSchema(), schema_);
    double_check_pass = table_heap_->GetTuple(*rid, &dummy_tuple, exec_ctx_->GetTransaction());
  } while (table_iter_ != table_heap_->End() && !double_check_pass);
  return double_check_pass;
}
```

### insert executor

insert executor是Write类型操作，所以各种隔离级别都只在 txn ends 才释放锁，另外需要主要的是，在插入过程中，需要在 txn 中维护插入过的值， 用于 txn abort时 rollback 用。

```c++

bool InsertExecutor::TryLock(RID rid) {
  try {
    return exec_ctx_->GetLockManager()->LockExclusive(exec_ctx_->GetTransaction(), rid);
  } catch (const TransactionAbortException &e) {
    return false;
  }
  return false;
}

bool InsertExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) {
  /**
   * RR, RC, RU all should release lock when txn aborted or committed
   */
  RID inserted_rid;
  Tuple inserted_tuple;
  std::vector<Tuple> pending_inserted_tuples;

  /* insert value directly */
  if (child_executor_ == nullptr) {
    auto values = plan_->RawValues();
    /*  Insert all values int to tables */
    for (const std::vector<Value> &entry : values) {
      inserted_tuple = Tuple(entry, schema_);
      pending_inserted_tuples.push_back(inserted_tuple);
    }
  } else {
    /* insert from child_executor */
    while (child_executor_->Next(&inserted_tuple, &inserted_rid)) {
      pending_inserted_tuples.push_back(inserted_tuple);
    }
  }

  for (const auto &pending_inserted_tuple : pending_inserted_tuples) {
    /* do insert */
    /* NOTE: function "InsertTuple" maintains the write set which is used to rollback  */

    if (!table_meta_->table_->InsertTuple(pending_inserted_tuple, &inserted_rid, exec_ctx_->GetTransaction())) {
      return false;
    }
    if (!TryLock(inserted_rid)) {
      exec_ctx_->GetTransaction()->SetState(TransactionState::ABORTED);
      return false;
    }

    /* Update index */
    for (IndexInfo *idx_info : idx_infos_) {
      idx_info->index_->InsertEntry(pending_inserted_tuple, inserted_rid, exec_ctx_->GetTransaction());
      /* maintain writeset used to rollback txn(I know it's somewhat a ugly way, but I have no other choose)  */
      IndexWriteRecord index_write_record(inserted_rid, table_meta_->oid_, WType::INSERT, pending_inserted_tuple,
                                          idx_info->index_oid_, exec_ctx_->GetCatalog());
      exec_ctx_->GetTransaction()->AppendTableWriteRecord(index_write_record);
    }
  }
  return false;
}
```

### update executor

update也相对简单：

- RU和RC需要加S lock
- RR由于不释放S lock，所以需要upgrade Lock

另外注意维护 write_set 即可。

代码如下：

```
bool UpdateExecutor::TryLock(RID rid) {
  try {
    if (exec_ctx_->GetTransaction()->GetIsolationLevel() == IsolationLevel::READ_COMMITTED) {
      /* child_executor get shared lock, now we want to update tuples, so upgrade the shared lock to exclusive lock */
      return exec_ctx_->GetLockManager()->LockExclusive(exec_ctx_->GetTransaction(), rid);
    }
    if (exec_ctx_->GetTransaction()->GetIsolationLevel() == IsolationLevel::REPEATABLE_READ) {
      /* child_executor get shared lock, now we want to update tuples, so upgrade the shared lock to exclusive lock */
      return exec_ctx_->GetLockManager()->LockUpgrade(exec_ctx_->GetTransaction(), rid);
    }
  } catch (const TransactionAbortException &e) {
    return false;
  }
  return false;
}

bool UpdateExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) {
  Tuple old_tuple;
  RID old_rid;
  while (child_executor_->Next(&old_tuple, &old_rid)) {
    if (!TryLock(old_rid)) {
      exec_ctx_->GetTransaction()->SetState(TransactionState::ABORTED);
      return false;
    }
    Tuple new_tuple = GenerateUpdatedTuple(old_tuple);
    if (!table_info_->table_->UpdateTuple(new_tuple, old_rid, exec_ctx_->GetTransaction())) {
      return false;
    }
    // LOG_DEBUG("update get lock %s", old_rid.ToString().c_str());
    for (IndexInfo *idx_info : index_infos_) {
      /* Update=Delete old + Insert new */
      old_tuple = old_tuple.KeyFromTuple(*plan_->GetChildPlan()->OutputSchema(), idx_info->key_schema_,
                                         idx_info->index_->GetKeyAttrs());
      new_tuple = new_tuple.KeyFromTuple(*plan_->GetChildPlan()->OutputSchema(), idx_info->key_schema_,
                                         idx_info->index_->GetKeyAttrs());
      idx_info->index_->DeleteEntry(old_tuple, old_rid, exec_ctx_->GetTransaction());
      idx_info->index_->InsertEntry(new_tuple, old_rid, exec_ctx_->GetTransaction());
      /* maintain write set */
      IndexWriteRecord index_write_record(old_rid, plan_->TableOid(), WType::UPDATE, new_tuple, idx_info->index_oid_,
                                          exec_ctx_->GetCatalog());
      index_write_record.old_tuple_ = old_tuple;

      exec_ctx_->GetTransaction()->AppendTableWriteRecord(index_write_record);
    }
  }
  return false;
}
```

### delete executor

delete属于write 操作，在各个隔离级别中，都等待txn结束后再释放 X lock。额外注意维护write set即可。

代码如下：

```c++
bool DeleteExecutor::TryLock(RID rid) {
  try {
    /* when execute deletion operation on rid, the rid has been in the shared lock mode  */
    return exec_ctx_->GetLockManager()->LockUpgrade(exec_ctx_->GetTransaction(), rid);
  } catch (const TransactionAbortException &e) {
    return false;
  }
  return false;
}

bool DeleteExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) {
  /**
   * RR, RC, RU 都是删除时加锁, 等待txn结束时再解锁
   */
  Tuple pending_tuple;
  RID pending_rid;

  while (child_executor_->Next(&pending_tuple, &pending_rid)) {
    if (!TryLock(pending_rid)) {
      exec_ctx_->GetTransaction()->SetState(TransactionState::ABORTED);
      return false;
    }
    // LOG_DEBUG("try to delete tuple %s", pending_tuple.ToString(child_executor_->GetOutputSchema()).c_str());
    if (!table_meta_->table_->MarkDelete(pending_rid, exec_ctx_->GetTransaction())) {
      return false;
    }
    // TableWriteRecord table_write_record(pending_rid, WType::DELETE, pending_tuple, table_meta_->table_.get());
    // exec_ctx_->GetTransaction()->GetWriteSet()->push_back(table_write_record);

    for (IndexInfo *idx_info : index_infos_) {
      idx_info->index_->DeleteEntry(pending_tuple, pending_rid, exec_ctx_->GetTransaction());
      IndexWriteRecord index_write_record(pending_rid, table_meta_->oid_, WType::DELETE, pending_tuple,
                                          idx_info->index_oid_, exec_ctx_->GetCatalog());
      exec_ctx_->GetTransaction()->AppendTableWriteRecord(index_write_record);
    }
    // Unlock();
  }
  return false;
}
```

## 4. 总结

proj 4收获颇多，对 Lock-Baed Isolation 了解得更多深入。比起以前只会背八股文，现在发现手动实现一个 DeadLock Detection Thread发现也是非常简单的。

至此，CMU 15445 所有proj都已完成，不得不佩服CMU的教学质量，课程+pro能够极大加深学生对知识内容的理解。

剩下就是看完和recovery、distributed相关的内容，学学go， 为6.824打基础。另外对CMU15418也非常感兴趣。看后续的时间了。

