---
title: CMU-15445-proj3(QueryPlan)
date: 2022-01-20 15:54:14
categories: CMU15445
tags:
---



## 0. 前言 & 要求

断更了3个月，终于忙完了毕设，现在开始补课。文本为 CMU15445 - Proj3 的题解 。Proj3 为QueryPlan，要求实现多个 executors，从而达到以下操作：

- **Access Methods:** Sequential Scans, Index Scans (with your B+Tree from [Project #2](https://15445.courses.cs.cmu.edu/fall2020/project2))
- **Modifications:** Inserts, Updates, Deletes
- **Miscellaneous:** Nested Loop Joins, Index Nested Loop Joins, Aggregation, Limit/Offset

所有的执行器，采用 iterator query processing model, **即所有的 executor 向外暴露一个 Next 接口**，ExecutorEngine 通过调用 Next接口得到一条tuple， 这种 processing model 为 pipeline style processing。 

详细要求如下：

- Task 1: 实现一个 System Catalog:  Catalog为数据库的核心数据结构之一，描述着数据库内部表和索引的元数据，存储着数据库中 有哪些table， 每个table的schema是怎么样的，每个table有哪些index， index有哪些元数据等等。
- Task 2: 实现多个执行器，包括：Sequential Scan, Index Scan,  Insert, Update,Delete， Nested Loop Join, Index nested  loop join， Aggregation， Limit.

总体来说，本实验难度比proj1高，比proj2低。

<!--more-->

## 1. Task1 -- SYSTEM CATALOG

Catalog为数据库内部的table和index的元数据，所以这个Task要求实现如下接口：
Table相关：

- `CreateTable(Transaction *txn, const std::string &table_name, const Schema &schema)`
- `GetTable(const std::string &table_name)`
- `GetTable(table_oid_t table_oid)`

Index相关：

- `CreateIndex(txn, index_name, table_name, schema, key_schema key_attrs, keysize)`
- `GetIndex(const std::string &index_name, const std::string &table_name)`
- `GetIndex(index_oid_t index_oid)`,
- `GetTableIndexes(const std::string &table_name)`

整体来说这部分非常简单，只是在实现Index接口部分时，需要注意将table内所有tuples都插入到index(fall2020 中，index为bptree index)。下面贴出 CreateIndex的代码：

```c++
  /**
   * Create a new index, populate existing data of the table and return its metadata.
   * @param txn the transaction in which the table is being created
   * @param index_name the name of the new index
   * @param table_name the name of the table
   * @param schema the schema of the table
   * @param key_schema the schema of the key
   * @param key_attrs key attributes
   * @param keysize size of the key
   * @return a pointer to the metadata of the new table
   */
  template <class KeyType, class ValueType, class KeyComparator>
  IndexInfo *CreateIndex(Transaction *txn, const std::string &index_name, const std::string &table_name,
                         const Schema &schema, const Schema &key_schema, const std::vector<uint32_t> &key_attrs, keysize) {
    /*
      1. 生成 index_oid
      2. 生成 IndexInfo 结构
      3. 保存 table name -> index names -> index id 的mapping关系
     */
    index_oid_t index_oid = next_index_oid_.fetch_add(1, std::memory_order_relaxed);
    IndexMetadata *idx_meta = new IndexMetadata(index_name, table_name, &schema, key_attrs);
    /* NOTE: Index 实例化到BPTree? */
    // Index *idx = new Index(idx_meta);
    Index *idx = new BPlusTreeIndex<KeyType, ValueType, KeyComparator>(idx_meta, bpm_);
    IndexInfo *idx_info =
        new IndexInfo(key_schema, index_name, std::unique_ptr<Index>(idx), index_oid, table_name, keysize);
    /* generate mapping relation */
    indexes_.insert(std::make_pair(index_oid, idx_info));
    if (index_names_.count(table_name) == 0U) {
      index_names_.insert(std::make_pair(table_name, std::unordered_map<std::string, index_oid_t>()));
    }
    index_names_[table_name][index_name] = index_oid;
    /* populate all data */
    // LOG_DEBUG("table name:%s, index name:%s", table_name.c_str(), index_name.c_str());
    TableHeap *table = GetTable(table_name)->table_.get();
    for (auto it = table->Begin(txn); it != table->End(); it++) {
      auto tmp = it->KeyFromTuple(schema, key_schema, key_attrs);
      // LOG_DEBUG("index key:%d", tmp.GetValue(&key_schema, 0).GetAs<uint32_t>());
      idx->InsertEntry(it->KeyFromTuple(schema, key_schema, key_attrs), it->GetRid(), txn);
    }
    return idx_info;
  }
```

其余接口相对简单，这里不赘述。 毕竟本Proj的核心在Executor。

## 2. Task2 -- EXECUTORS

Task2要求实现多个执行器，涉及到的header file如下：

- `src/include/execution/executors/seq_scan_executor.h`
- `src/include/execution/executors/index_scan_executor.h`
- `src/include/execution/executors/insert_executor.h`
- `src/include/execution/executors/update_executor.h`
- `src/include/execution/executors/delete_executor.h`
- `src/include/execution/executors/nested_loop_join_executor.h`
- `src/include/execution/executors/nested_index_join_executor.h`
- `src/include/execution/executors/aggregation_executor.h`
- `src/include/execution/executors/limit_executor.h`

每个file代表了一个Executor。

个人建议，在开始写代码前，详细阅读 executor_test.cpp 对应的测试。特别关注以下几个类或结构：

1. **AbstractExpression及其派生类**，可以说不理解这个继承家族，后面的task没法做， 当然这个继承关系也让我深刻体会到了OOP编程方式的灵魂与恶心（是的，这OOP让人阅读起来非常的不方便）。

   **尤其关注 ComparisonExpression 和 ColumnValueExpression**

2. 每一种Executor对应了一个Plan，每个Plan都需要仔细关注。

3. 每种Executor都可能存在predicate， 这是用来过滤(想想WHERE clause)或者JOIN用（想想 JOIN ON XXX)

4. 每种Executor都有一个OutputSchema，我们从table中拿到的tuple都是全字段的，在executor往上pass tuple的时候，需要按照OutputSchema来提取需要的字段，这个提取操作需要用到 **Expression**， 所以Expression的理解是重中之重。

bustub通过 ExecutionEngine + ExecutorFactory 来完成Executor的执行。

好，下面我们详细分析下每种Executor。

### 1. SEQUENTIAL SCAN

实现类似SQL语句：

```sql
SELECT colA, colB FROM test_1 WHERE colA < 500
```

顺序Scan，这个Executor是最接近的table的Executor之一（另一个是Index SCAN），它的作用是从table中获取tuples。获取方式是通过顺序扫描全表，这里的核心数据结构是 **TableIterator**, 通过该迭代器可以可以遍历全表，然后通过 predicate 过滤掉需要的tuple。

这个Exectuor是所有Executor中最为简单的一个，所以这里不贴出代码，同学可以自行思考如何实现，我要提到的是如何从tuple中按照output schema提出需要的字段。代码如下：

```c++
static Tuple GenerateFinalTuple(const Tuple &tuple, const Schema *output_schema, const Schema *schema) {
  std::vector<Value> res;
  for (const Column &col : output_schema->GetColumns()) {
    res.push_back(col.GetExpr()->Evaluate(&tuple, schema));
  }
  return Tuple(res, output_schema);
}
```

解释一下这里的三个参数：

1. tuple： 需要被提取的tuple
2. output_schema: 想要输出的schema或者说是哪些字段
3. schema： 需要被提取的tuple，当前的schema是如何的。

注意： **schema** 应该是全包含 **output_schema**的。

### 2.  INDEX SCANS

实现类似如下语句：

```sql
SELECT colA, colB FROM test_1 WHERE colA < 500
```

不过colA是有index的。

INDEX SCAN和SEQUENTIAL SCAN类似，也是从table中获取tuple的exectuor。SEQ SCAN通过TableIterator，INDEX SCAN则通过index iterator。

### 3. INSERT

INSERT实现插入操作，而插入操作分为两类：

1. 直接数据插入，如： `   INSERT INTO empty_table2 VALUES (200, 20), (201, 21), (202, 22)`
2. SELECT 子表插入，如： `INSERT INTO empty_table2 SELECT colA, colB FROM test_1 WHERE colA > 500`

需要注意的是，除了向table中插入外，还需要同时向index插入。

这里给出我的实现：

```c++
bool InsertExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) {
  RID inserted_rid;
  Tuple inserted_tuple;
  /* insert value directly */
  if (child_executor_ == nullptr) {
    auto values = plan_->RawValues();
    /* Insert all values int to tables */
    for (const std::vector<Value> &entry : values) {
      inserted_tuple = Tuple(entry, schema_);
      if (!table_heap_->InsertTuple(inserted_tuple, &inserted_rid, exec_ctx_->GetTransaction())) {
        return false;
      }
      /* Update index */
      for (IndexInfo *idx_info : idx_infos_) {
        idx_info->index_->InsertEntry(inserted_tuple, inserted_rid, exec_ctx_->GetTransaction());
      }
    }
  } else {
    /* insert from child_executor */
    while (child_executor_->Next(&inserted_tuple, &inserted_rid)) {
      if (!table_heap_->InsertTuple(inserted_tuple, &inserted_rid, exec_ctx_->GetTransaction())) {
        return false;
      }
      /* Update index */
      for (IndexInfo *idx_info : idx_infos_) {
        idx_info->index_->InsertEntry(inserted_tuple, inserted_rid, exec_ctx_->GetTransaction());
      }
    }
  }
  return false;
}
```

### 4. UPDATE

实现类似如下语句：

```sql
UPDATE empty_table2 SET colA = colA+10 WHERE colA < 50
```

和Insert操作类似，只不过update index时，我是采用先delete再insert的方式：

```c++
bool UpdateExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) {
  Tuple pending_tuple;
  RID pending_rid;
  while (child_executor_->Next(&pending_tuple, &pending_rid)) {
    Tuple updated_tuple = GenerateUpdatedTuple(pending_tuple);
    /* update tuple entry and index */
    if (!table_info_->table_->UpdateTuple(updated_tuple, pending_rid, exec_ctx_->GetTransaction())) {
      return false;
    }
    /*  */
    for (IndexInfo *idx_info : index_infos_) {
      /* Update= Delete old + Insert new */
      pending_tuple =
          pending_tuple.KeyFromTuple(table_info_->schema_, idx_info->key_schema_, idx_info->index_->GetKeyAttrs());
      updated_tuple =
          updated_tuple.KeyFromTuple(table_info_->schema_, idx_info->key_schema_, idx_info->index_->GetKeyAttrs());
      idx_info->index_->DeleteEntry(pending_tuple, pending_rid, exec_ctx_->GetTransaction());
      idx_info->index_->InsertEntry(updated_tuple, pending_rid, exec_ctx_->GetTransaction());
    }
  }
  return false;
}
```

### 5. DELETE

实现类似如下操作：

```sql
DELETE FROM test_1 WHERE colA < 50
```

和前面的操作都类似，通过child executor得到需要删除的tuples，然后在table和index中执行删除即可。

```c++
bool DeleteExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) {
  Tuple pending_tuple;
  RID pending_rid;
  while (child_executor_->Next(&pending_tuple, &pending_rid)) {
    if (!table_info_->table_->MarkDelete(pending_rid, exec_ctx_->GetTransaction())) {
      return false;
    }
    for (IndexInfo *idx_info : index_infos_) {
      idx_info->index_->DeleteEntry(pending_tuple, pending_rid, exec_ctx_->GetTransaction());
    }
  }
  return false;
}
```

### 6. NESTED LOOP JOIN

实现类似如下操作：

```sql
SELECT test_1.colA, test_1.colB, test_2.col1, test_2.col3 FROM test_1 JOIN test_2 ON test_1.colA = test_2.col1 AND test_1.colA < 50
```

这里的难点是涉及到两个table的处理，如何把两个table给连接起来。其实依然还是那个核心Expression类， 通过EvaluateJoin接口即可。

我个人的实现思路为，在Init时就把所有最终的tuple组装保存好，上层在调用Next时，只用从container中pop一个tuple即可。

我的实现如下：

```c++
void NestedLoopJoinExecutor::FetchAllTuples() {
  std::vector<Tuple> left_tuples;
  std::vector<Tuple> right_tuples;
  {
    Tuple left_tuple;
    Tuple right_tuple;
    RID left_rid;
    RID right_rid;
    while (left_executor_->Next(&left_tuple, &left_rid)) {
      left_tuples.push_back(left_tuple);
    }
    while (right_executor_->Next(&right_tuple, &right_rid)) {
      right_tuples.push_back(right_tuple);
    }
  }
  /* now we have fetched all tuples in left and right table, it's time to combine them */
  /* assume left table is smaller */
  for (const Tuple &left : left_tuples) {
    for (const Tuple &right : right_tuples) {
      /* apply predicator */
      if (plan_->Predicate() == nullptr ||
          plan_->Predicate()
              ->EvaluateJoin(&left, left_executor_->GetOutputSchema(), &right, right_executor_->GetOutputSchema())
              .GetAs<bool>()) {
        tmp_tuples_.push_back(GenerateFinalTuple(left, left_executor_->GetOutputSchema(), right,
                                                 right_executor_->GetOutputSchema(), plan_->OutputSchema()));		// save tuple into tmp_tuples_!
      }
    }
  }
}

/**
 * @brief this implementation is a pipeline breaker, we generate all tuples
 * in left and right tables, and combine them together. Then use 2 for loop to
 * save them in "tmp_tuples_". For subsequent request, we only need to pop one
 * tuple from the "tmp_tuples_", and return it back
 */
bool NestedLoopJoinExecutor::Next(Tuple *tuple, RID *rid) {
  if (!tmp_tuples_.empty()) {
    *tuple = tmp_tuples_.back();
    tmp_tuples_.pop_back();
    return true;
  }
  /* no tuples or all tuples have been output */
  return false;
}

```

### 7. INDEX NESTED LOOP JOIN

实现类似如下操作：

```
SELECT test_1.colA, test_1.colB, test_2.col1, test_2.col3 FROM test_1 JOIN test_2 ON test_1.colA = test_2.col1 AND test_1.colA < 50
```

不过在join时，inner table采用index寻找对应匹配的tuple。

如何理解该executor？ outer table通过某种scan方式拿到一个tuple后，将这个tuple转化为inner table的index的一个key，然后用这个key去index中寻找对应匹配的tuple rid， 并根据这个rid到table中获取最终的tuple，最后将outer tuple和inner tuple结合在一起即可。

问题的难点在于如何生成用于索引index的probe key？要做到这部分，需要充分理解 KeyFromTuple 这个函数的三个参数的含义：

```c++
Tuple KeyFromTuple(const Schema &schema, const Schema &key_schema, const std::vector<uint32_t> &key_attrs) 
```

调用方式为：

```c++
tuple.KeyFromTuple(schema, key_schema, key_attrs)
```

分别解释下这三个参数和tuple的含义：

1. tuple调用者： 即当前要被转换的tuple。 对应到下面的代码为 outer table tuple
2. schema， 原tuple的schema布局
3. key_schema, 转换后的tuple的schema布局。 （不要求 schema一定是key_schema的超集（也就是不要求两者字段名字能够对应）， 但是要求 key_schema中的每个column的size在原schema中都能找到对应的column的size，两个size相同）。 如：原schema为 {A(int),B(int),C(big int)}, 转换后的key_schema为 {D(int)}, 这里D的名字不在{A B C} 中，但是D的int size为4， {A,B,C}中A和B都为4，这种情况下是可以转换的。
4. key_attrs， 原tuple的schema布局中，要被抽取出来的column的索引号。 如原schema: {"A","B","C",  "D", 'E'}, 现在要抽取 {"A","C"}, 则key_attrs为 {0,2}

**理解了这个函数，现在我们可以从plan_ 中获取OuterTableSchema，也可以从index中获取index key schema，唯独最重要的 key_attrs 不知道如何获取。**

 **这个关键信息隐藏在 predicate中， predicate是一个ComparisonExpression， 它含有两个children expression。其中按照约定，第0个child是左表达式，而左表达式按照约定是outer table的 ColumnExpression, 在ColumnExpression中，隐藏着这个Column在其Schema中的索引位置，可以通过这个索引位置来构造 key_attr.**

不过可以看出，使用这种方式，要基于一定的假设（如第0个child是左表达式），同时这种实现方式也只能处理 JOIN ON 一个condition的情况，不能有多个condition。举例来说：

可以满足：

```sql
SELECT * FROM A JOIN B ON A.a = B.a	# 一个condition
```

但是不能满足：

```sql
SELECT *FROM A FROM B ON A.a = B.a AND A.b = B.b # 两个condition
```

因为这里有两个condition。

下面是我的实现代码, 我的做法依然是先获取所有tuples保存在一个vector中，当上层调用Next接口时，直接从vector中pop即可。

```c++
void NestIndexJoinExecutor::FetchAllTuples() {
  /**
   * 这里的难点在于如何构造probe key. 网上有说直接通过KeyFromTuple传入index schema，但个人感觉不对
   * 我的做法是，从plan中取出predicate(这是一个comparasion_expression), 然后从predicate中取出column
   * 信息。 但是这种做法也有缺点， 缺点在于只能接受JOIN时有且仅有一个predicate。不过目前的bustub似乎也支持
   * 一个predicate. 说下步骤：
   * 1. loop outer table to get all tuples
   * 2. construct index probe key by "predicate"
   * 3. locate rid coressonding to the index probe key.
   * 4. fetch tuple from inner table using the rid
   * 5. combine inner tuple and outter tuple, formatting them by "output schema"
   */
  /* ok, now fetch inner tuple by index */
  Index *idx = inner_idx_info_->index_.get();
  TableHeap *inner_table = inner_table_meta_->table_.get();
  std::vector<Tuple> outter_tuples;
  std::vector<Tuple> index_probe_keys;
  std::vector<RID> result_rids;
  Tuple outter_tuple;
  Tuple inner_tuple;
  RID outter_rid;

  /* build probe keys */
  const ColumnValueExpression *outter_table_probe_key_column = dynamic_cast<const ColumnValueExpression *>(
      plan_->Predicate()->GetChildAt(0)); /* NOTE: assume child 0 is a outter table's column */
  while (outter_executor_->Next(&outter_tuple, &outter_rid)) {
    outter_tuples.push_back(outter_tuple);
    Tuple probe_key_tuple = outter_tuple.KeyFromTuple(*plan_->OuterTableSchema(), inner_idx_info_->key_schema_,
                                                      {outter_table_probe_key_column->GetColIdx()});
    index_probe_keys.push_back(probe_key_tuple);
  }

  /* fetch tuple from inner table by probe key, then combine inner tuple and outer tuple to form the final tuple */
  for (size_t i = 0; i < index_probe_keys.size(); i++) {
    result_rids.clear();
    idx->ScanKey(index_probe_keys[i], &result_rids, exec_ctx_->GetTransaction());
    assert(result_rids.size() <= 1);
    if (result_rids.empty()) { /* not found */
      continue;
    }
    if (!inner_table->GetTuple(result_rids[0], &inner_tuple, exec_ctx_->GetTransaction())) {
      LOG_ERROR("can't find the corresponding tuple in the inner table index");
      abort(); /* or throw an exception */
    }
    /* format inner tuple */
    inner_tuple = ExtractTupleFromSchema(inner_tuple, &inner_table_meta_->schema_, plan_->InnerTableSchema());
    /* predicate filter */
    if (plan_->Predicate() == nullptr ||
        plan_->Predicate()
            ->EvaluateJoin(&outter_tuples[i], plan_->OuterTableSchema(), &inner_tuple, plan_->InnerTableSchema())
            .GetAs<bool>()) {
      tmp_tuples_.push_back(GenerateFinalTuple(outter_tuples[i], plan_->OuterTableSchema(), inner_tuple,
                                               plan_->InnerTableSchema(), plan_->OutputSchema()));
    }
  }
}

bool NestIndexJoinExecutor::Next(Tuple *tuple, RID *rid) {
  if (tmp_tuples_.empty()) {
    return false;
  }
  *tuple = tmp_tuples_.back();
  tmp_tuples_.pop_back();
  return true;
}

```

### 8. AGGREGATION

aggregation实现聚合操作，如对某一列执行 SUM, MAX, MIN, COUNT 等， 同时需要考虑group by和having子句。

要实现aggregation，需要使用到hashtable，将属于同一个分类的tuples聚集到一起，然后使用 aggregation 操作（如SUM），对这类tuples进行计算。

现在的问题是，**如何分类？**

group by用于分类，属于相同group的，存放在一个hash bucket中。如果没有group by子句，则认为所有tuples属于同一类。

**如何计算？**

详细计算接口请阅读 **SimpleAggregationHashTable** 类代码。

我个人的实现如下：

```c++
AggregationExecutor::AggregationExecutor(ExecutorContext *exec_ctx, const AggregationPlanNode *plan,
                                         std::unique_ptr<AbstractExecutor> &&child)
    : AbstractExecutor(exec_ctx),
      plan_(plan),
      child_(std::move(child)),
      agg_ht_(plan_->GetAggregates(), plan_->GetAggregateTypes()),
      it_({}) /* just a place holder */
{}

const AbstractExecutor *AggregationExecutor::GetChildExecutor() const { return child_.get(); }

void AggregationExecutor::Init() {
  agg_ht_.GenerateInitialAggregateValue();
  child_->Init();
  /* calc agg */
  Tuple tuple;
  RID rid;
  while (child_->Next(&tuple, &rid)) {
    AggregateKey agg_key = MakeKey(&tuple); /* key是按照group by分类 */
    AggregateValue agg_value = MakeVal(&tuple);
    /* populate key and value into hashtable */
    agg_ht_.InsertCombine(agg_key, agg_value);
  }

  /* iterator over agg_ht */
  it_ = agg_ht_.Begin();
}

static Tuple GenerateFinalTuple(const Schema *output_schema, const std::vector<Value> &group,
                                const std::vector<Value> &aggregates) {
  std::vector<Value> res;
  for (const auto &col : output_schema->GetColumns()) {
    res.push_back(col.GetExpr()->EvaluateAggregate(group, aggregates));
  }
  return Tuple(res, output_schema);
}

bool AggregationExecutor::Next(Tuple *tuple, RID *rid) {
  if (it_ == agg_ht_.End()) {
    return false;
  }
  AggregateKey agg_key = it_.Key();
  AggregateValue agg_val = it_.Val();
  if (plan_->GetHaving() != nullptr) {
    /* having clause  */
    while (it_ != agg_ht_.End() &&
           !plan_->GetHaving()->EvaluateAggregate(agg_key.group_bys_, agg_val.aggregates_).GetAs<bool>()) {
      agg_key = it_.Key();
      agg_val = it_.Val();
      ++it_;
    }
    if (it_ == agg_ht_.End()) {
      return false;
    }
  }
  *tuple = GenerateFinalTuple(plan_->OutputSchema(), agg_key.group_bys_, agg_val.aggregates_);
  ++it_;
  return true;
}
```

### 9. LIMIT

LIMIT是限定最后输出的tuple数量，内部实现为采offset，每隔offset，则pop out一次tuple。非常简单，个人实现如下：

```c++
bool LimitExecutor::Next(Tuple *tuple, RID *rid) {
  size_t offset = plan_->GetOffset();
  Tuple tmp_tuple;
  RID tmp_rid;
  while ((--offset != 0) && child_executor_->Next(&tmp_tuple, &tmp_rid)) {
  }
  if (offset == 0 && child_executor_->Next(&tmp_tuple, &tmp_rid)) {
    *tuple = tmp_tuple;
    *rid = tmp_rid;
    return true;
  }
  return false;
}
```

## 3. 测试文件

这里贴一下群里面用的测试文件，这些文件能通过，则最后提交到 gradescope 上也能通过：

### 1. gradding_catalog_test.cpp

```c++
//===----------------------------------------------------------------------===//
//
//                         BusTub
//
// grading_catalog_test.cpp
//
// Identification: test/catalog/grading_catalog_test.cpp
//
// Copyright (c) 2015-2019, Carnegie Mellon University Database Group
//
//===----------------------------------------------------------------------===//

#include <string>
#include <unordered_set>
#include <vector>

#include "buffer/buffer_pool_manager.h"
#include "catalog/catalog.h"
#include "catalog/table_generator.h"
#include "concurrency/transaction_manager.h"
#include "execution/executor_context.h"
#include "gtest/gtest.h"
#include "storage/b_plus_tree_test_util.h"  // NOLINT
#include "type/value_factory.h"

namespace bustub {

// NOLINTNEXTLINE
TEST(GradingCatalogTest, CreateTableTest) {
  auto disk_manager = new DiskManager("catalog_test.db");
  auto bpm = new BufferPoolManager(32, disk_manager);
  auto catalog = new Catalog(bpm, nullptr, nullptr);
  std::string table_name = "potato";

  // The table shouldn't exist in the catalog yet.
  EXPECT_THROW(catalog->GetTable(table_name), std::out_of_range);

  // Put the table into the catalog.
  std::vector<Column> columns;
  columns.emplace_back("A", TypeId::INTEGER);
  columns.emplace_back("B", TypeId::BOOLEAN);

  Schema schema(columns);
  auto *table_metadata = catalog->CreateTable(nullptr, table_name, schema);

  // Catalog lookups should succeed.
  {
    ASSERT_EQ(table_metadata, catalog->GetTable(table_metadata->oid_));
    ASSERT_EQ(table_metadata, catalog->GetTable(table_name));
  }

  // Basic empty table attributes.
  {
    ASSERT_EQ(table_metadata->table_->GetFirstPageId(), 0);
    ASSERT_EQ(table_metadata->name_, table_name);
    ASSERT_EQ(table_metadata->schema_.GetColumnCount(), columns.size());
    for (size_t i = 0; i < columns.size(); i++) {
      ASSERT_EQ(table_metadata->schema_.GetColumns()[i].GetName(), columns[i].GetName());
      ASSERT_EQ(table_metadata->schema_.GetColumns()[i].GetType(), columns[i].GetType());
    }
  }

  // Try inserting a tuple and checking that the catalog lookup gives us the right table.
  {
    std::vector<Value> values;
    values.emplace_back(ValueFactory::GetIntegerValue(15445));
    values.emplace_back(ValueFactory::GetBooleanValue(false));
    Tuple tuple(values, &schema);

    Transaction txn(0);
    RID rid;
    table_metadata->table_->InsertTuple(tuple, &rid, &txn);

    auto table_iter = catalog->GetTable(table_name)->table_->Begin(&txn);
    ASSERT_EQ((*table_iter).GetValue(&schema, 0).CompareEquals(tuple.GetValue(&schema, 0)), CmpBool::CmpTrue);
    ASSERT_EQ((*table_iter).GetValue(&schema, 1).CompareEquals(tuple.GetValue(&schema, 1)), CmpBool::CmpTrue);
    ASSERT_EQ(++table_iter, catalog->GetTable(table_name)->table_->End());
  }

  delete catalog;
  delete bpm;
  delete disk_manager;
}

// NOLINTNEXTLINE
TEST(GradingCatalogTest, CreateIndexTest) {
  auto disk_manager = std::make_unique<DiskManager>("catalog_test.db");
  auto bpm = std::make_unique<BufferPoolManager>(32, disk_manager.get());
  auto catalog = std::make_unique<Catalog>(bpm.get(), nullptr, nullptr);

  Transaction txn(0);

  auto exec_ctx = std::make_unique<ExecutorContext>(&txn, catalog.get(), bpm.get(), nullptr, nullptr);

  TableGenerator gen{exec_ctx.get()};
  gen.GenerateTestTables();

  auto table_info = exec_ctx->GetCatalog()->GetTable("test_1");

  Schema &schema = table_info->schema_;
  auto itr = table_info->table_->Begin(&txn);
  auto tuple = *itr;

  Schema *key_schema = ParseCreateStatement("a bigint");
  GenericComparator<8> comparator(key_schema);
  auto index_info = catalog->CreateIndex<GenericKey<8>, RID, GenericComparator<8>>(&txn, "index1", "test_1", schema,
                                                                                   *key_schema, {0}, 8);
  Tuple index_key = tuple.KeyFromTuple(schema, *key_schema, index_info->index_->GetKeyAttrs());
  std::vector<RID> index_rid;
  index_info->index_->ScanKey(index_key, &index_rid, &txn);
  ASSERT_EQ(tuple.GetRid().Get(), index_rid[0].Get());

  delete key_schema;
}

}  // namespace bustub

```

### 2. grading_exectuor_test.cpp

```c++
//===----------------------------------------------------------------------===//
//
//                         BusTub
//
// executor_test.cpp
//
// Identification: test/execution/executor_test.cpp
//
// Copyright (c) 2015-2019, Carnegie Mellon University Database Group
//
//===----------------------------------------------------------------------===//

#include <cstdio>
#include <memory>
#include <string>
#include <unordered_set>
#include <utility>
#include <vector>

#include "buffer/buffer_pool_manager.h"
#include "catalog/table_generator.h"
#include "concurrency/transaction_manager.h"
#include "execution/execution_engine.h"
#include "execution/executor_context.h"
#include "execution/executors/aggregation_executor.h"
#include "execution/executors/insert_executor.h"
#include "execution/executors/nested_loop_join_executor.h"
#include "execution/expressions/aggregate_value_expression.h"
#include "execution/expressions/column_value_expression.h"
#include "execution/expressions/comparison_expression.h"
#include "execution/expressions/constant_value_expression.h"
#include "execution/plans/delete_plan.h"
#include "execution/plans/index_scan_plan.h"
#include "execution/plans/limit_plan.h"
#include "execution/plans/nested_index_join_plan.h"
#include "execution/plans/seq_scan_plan.h"
#include "execution/plans/update_plan.h"
#include "gtest/gtest.h"
#include "storage/b_plus_tree_test_util.h"  // NOLINT
#include "type/value_factory.h"

namespace bustub {

class GradingExecutorTest : public ::testing::Test {
 public:
  // This function is called before every test.
  void SetUp() override {
    ::testing::Test::SetUp();
    // For each test, we create a new DiskManager, BufferPoolManager, TransactionManager, and Catalog.
    disk_manager_ = std::make_unique<DiskManager>("executor_test.db");
    bpm_ = std::make_unique<BufferPoolManager>(2560, disk_manager_.get());
    page_id_t page_id;
    bpm_->NewPage(&page_id);
    txn_mgr_ = std::make_unique<TransactionManager>(lock_manager_.get(), log_manager_.get());
    catalog_ = std::make_unique<Catalog>(bpm_.get(), lock_manager_.get(), log_manager_.get());
    // Begin a new transaction, along with its executor context.
    txn_ = txn_mgr_->Begin();
    exec_ctx_ =
        std::make_unique<ExecutorContext>(txn_, catalog_.get(), bpm_.get(), txn_mgr_.get(), lock_manager_.get());
    // Generate some test tables.
    TableGenerator gen{exec_ctx_.get()};
    gen.GenerateTestTables();

    execution_engine_ = std::make_unique<ExecutionEngine>(bpm_.get(), txn_mgr_.get(), catalog_.get());
  }

  // This function is called after every test.
  void TearDown() override {
    // Commit our transaction.
    txn_mgr_->Commit(txn_);
    // Shut down the disk manager and clean up the transaction.
    disk_manager_->ShutDown();
    remove("executor_test.db");
    delete txn_;
  };

  /** @return the executor context in our test class */
  ExecutorContext *GetExecutorContext() { return exec_ctx_.get(); }
  ExecutionEngine *GetExecutionEngine() { return execution_engine_.get(); }
  Transaction *GetTxn() { return txn_; }
  TransactionManager *GetTxnManager() { return txn_mgr_.get(); }
  Catalog *GetCatalog() { return catalog_.get(); }
  BufferPoolManager *GetBPM() { return bpm_.get(); }

  // The below helper functions are useful for testing.

  const AbstractExpression *MakeColumnValueExpression(const Schema &schema, uint32_t tuple_idx,
                                                      const std::string &col_name) {
    uint32_t col_idx = schema.GetColIdx(col_name);
    auto col_type = schema.GetColumn(col_idx).GetType();
    allocated_exprs_.emplace_back(std::make_unique<ColumnValueExpression>(tuple_idx, col_idx, col_type));
    return allocated_exprs_.back().get();
  }

  const AbstractExpression *MakeConstantValueExpression(const Value &val) {
    allocated_exprs_.emplace_back(std::make_unique<ConstantValueExpression>(val));
    return allocated_exprs_.back().get();
  }

  const AbstractExpression *MakeComparisonExpression(const AbstractExpression *lhs, const AbstractExpression *rhs,
                                                     ComparisonType comp_type) {
    allocated_exprs_.emplace_back(std::make_unique<ComparisonExpression>(lhs, rhs, comp_type));
    return allocated_exprs_.back().get();
  }

  const AbstractExpression *MakeAggregateValueExpression(bool is_group_by_term, uint32_t term_idx) {
    allocated_exprs_.emplace_back(
        std::make_unique<AggregateValueExpression>(is_group_by_term, term_idx, TypeId::INTEGER));
    return allocated_exprs_.back().get();
  }

  const Schema *MakeOutputSchema(const std::vector<std::pair<std::string, const AbstractExpression *>> &exprs) {
    std::vector<Column> cols;
    cols.reserve(exprs.size());
    for (const auto &input : exprs) {
      if (input.second->GetReturnType() != TypeId::VARCHAR) {
        cols.emplace_back(input.first, input.second->GetReturnType(), input.second);
      } else {
        cols.emplace_back(input.first, input.second->GetReturnType(), MAX_VARCHAR_SIZE, input.second);
      }
    }
    allocated_output_schemas_.emplace_back(std::make_unique<Schema>(cols));
    return allocated_output_schemas_.back().get();
  }

 private:
  std::unique_ptr<TransactionManager> txn_mgr_;
  Transaction *txn_{nullptr};
  std::unique_ptr<DiskManager> disk_manager_;
  std::unique_ptr<LogManager> log_manager_ = nullptr;
  std::unique_ptr<LockManager> lock_manager_ = nullptr;
  std::unique_ptr<BufferPoolManager> bpm_;
  std::unique_ptr<Catalog> catalog_;
  std::unique_ptr<ExecutorContext> exec_ctx_;
  std::unique_ptr<ExecutionEngine> execution_engine_;
  std::vector<std::unique_ptr<AbstractExpression>> allocated_exprs_;
  std::vector<std::unique_ptr<Schema>> allocated_output_schemas_;
  static constexpr uint32_t MAX_VARCHAR_SIZE = 128;
};

// NOLINTNEXTLINE
TEST_F(GradingExecutorTest, SimpleSeqScanTest) {
  // SELECT colA, colB FROM test_1 WHERE colA > 600

  // Construct query plan
  TableMetadata *table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
  Schema &schema = table_info->schema_;
  auto *colA = MakeColumnValueExpression(schema, 0, "colA");
  auto *colB = MakeColumnValueExpression(schema, 0, "colB");
  auto *const600 = MakeConstantValueExpression(ValueFactory::GetIntegerValue(600));
  auto *predicate = MakeComparisonExpression(colA, const600, ComparisonType::GreaterThan);
  auto *out_schema = MakeOutputSchema({{"colA", colA}, {"colB", colB}});
  SeqScanPlanNode plan{out_schema, predicate, table_info->oid_};

  // Execute
  std::vector<Tuple> result_set;
  GetExecutionEngine()->Execute(&plan, &result_set, GetTxn(), GetExecutorContext());

  // Verify

  for (const auto &tuple : result_set) {
    ASSERT_TRUE(tuple.GetValue(out_schema, out_schema->GetColIdx("colA")).GetAs<int32_t>() > 600);
    ASSERT_TRUE(tuple.GetValue(out_schema, out_schema->GetColIdx("colB")).GetAs<int32_t>() < 10);
  }
  ASSERT_EQ(result_set.size(), 399);
}

// NOLINTNEXTLINE
TEST_F(GradingExecutorTest, SimpleIndexScanTest) {
  // SELECT colA, colB FROM test_1 WHERE colA > 500

  // Construct query plan
  TableMetadata *table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
  Schema &schema = table_info->schema_;

  Schema *key_schema = ParseCreateStatement("a bigint");
  GenericComparator<8> comparator(key_schema);
  auto index_info = GetExecutorContext()->GetCatalog()->CreateIndex<GenericKey<8>, RID, GenericComparator<8>>(
      GetTxn(), "index1", "test_1", table_info->schema_, *key_schema, {0}, 8);

  auto *colA = MakeColumnValueExpression(schema, 0, "colA");
  auto *colB = MakeColumnValueExpression(schema, 0, "colB");
  auto *const600 = MakeConstantValueExpression(ValueFactory::GetIntegerValue(600));
  auto *predicate = MakeComparisonExpression(colA, const600, ComparisonType::GreaterThan);
  auto *out_schema = MakeOutputSchema({{"colA", colA}, {"colB", colB}});
  IndexScanPlanNode plan{out_schema, predicate, index_info->index_oid_};

  // Execute
  std::vector<Tuple> result_set;
  GetExecutionEngine()->Execute(&plan, &result_set, GetTxn(), GetExecutorContext());

  // Verify
  for (const auto &tuple : result_set) {
    ASSERT_TRUE(tuple.GetValue(out_schema, out_schema->GetColIdx("colA")).GetAs<int32_t>() > 600);
    ASSERT_TRUE(tuple.GetValue(out_schema, out_schema->GetColIdx("colB")).GetAs<int32_t>() < 10);
  }
  ASSERT_EQ(result_set.size(), 399);

  delete key_schema;
}

// NOLINTNEXTLINE
TEST_F(GradingExecutorTest, SimpleRawInsertWithIndexTest) {
  // INSERT INTO empty_table2 VALUES (200, 20), (201, 21), (202, 22)
  // Create Values to insert
  std::vector<Value> val1{ValueFactory::GetIntegerValue(200), ValueFactory::GetIntegerValue(20)};
  std::vector<Value> val2{ValueFactory::GetIntegerValue(201), ValueFactory::GetIntegerValue(21)};
  std::vector<Value> val3{ValueFactory::GetIntegerValue(202), ValueFactory::GetIntegerValue(22)};
  std::vector<std::vector<Value>> raw_vals{val1, val2, val3};
  // Create insert plan node
  auto table_info = GetExecutorContext()->GetCatalog()->GetTable("empty_table2");
  InsertPlanNode insert_plan{std::move(raw_vals), table_info->oid_};

  Schema *key_schema = ParseCreateStatement("a bigint");
  GenericComparator<8> comparator(key_schema);
  auto index_info = GetExecutorContext()->GetCatalog()->CreateIndex<GenericKey<8>, RID, GenericComparator<8>>(
      GetTxn(), "index1", "empty_table2", table_info->schema_, *key_schema, {0}, 8);

  GetExecutionEngine()->Execute(&insert_plan, nullptr, GetTxn(), GetExecutorContext());

  // Iterate through table make sure that values were inserted.
  // SELECT * FROM empty_table2;
  auto &schema = table_info->schema_;
  auto colA = MakeColumnValueExpression(schema, 0, "colA");
  auto colB = MakeColumnValueExpression(schema, 0, "colB");
  auto out_schema = MakeOutputSchema({{"colA", colA}, {"colB", colB}});
  SeqScanPlanNode scan_plan{out_schema, nullptr, table_info->oid_};

  std::vector<Tuple> result_set;
  GetExecutionEngine()->Execute(&scan_plan, &result_set, GetTxn(), GetExecutorContext());

  // First value
  ASSERT_EQ(result_set[0].GetValue(out_schema, out_schema->GetColIdx("colA")).GetAs<int32_t>(), 200);
  ASSERT_EQ(result_set[0].GetValue(out_schema, out_schema->GetColIdx("colB")).GetAs<int32_t>(), 20);

  // Second value
  ASSERT_EQ(result_set[1].GetValue(out_schema, out_schema->GetColIdx("colA")).GetAs<int32_t>(), 201);
  ASSERT_EQ(result_set[1].GetValue(out_schema, out_schema->GetColIdx("colB")).GetAs<int32_t>(), 21);

  // Third value
  ASSERT_EQ(result_set[2].GetValue(out_schema, out_schema->GetColIdx("colA")).GetAs<int32_t>(), 202);
  ASSERT_EQ(result_set[2].GetValue(out_schema, out_schema->GetColIdx("colB")).GetAs<int32_t>(), 22);

  // Size
  ASSERT_EQ(result_set.size(), 3);
  std::vector<RID> rids;

  // Get RID from index, fetch tuple and then compare
  for (auto &i : result_set) {
    rids.clear();
    auto index_key = i.KeyFromTuple(schema, index_info->key_schema_, index_info->index_->GetKeyAttrs());
    index_info->index_->ScanKey(index_key, &rids, GetTxn());
    Tuple indexed_tuple;
    auto fetch_tuple = table_info->table_->GetTuple(rids[0], &indexed_tuple, GetTxn());

    ASSERT_TRUE(fetch_tuple);
    ASSERT_EQ(indexed_tuple.GetValue(out_schema, out_schema->GetColIdx("colA")).GetAs<int32_t>(),
              i.GetValue(out_schema, out_schema->GetColIdx("colA")).GetAs<int32_t>());
    ASSERT_EQ(indexed_tuple.GetValue(out_schema, out_schema->GetColIdx("colB")).GetAs<int32_t>(),
              i.GetValue(out_schema, out_schema->GetColIdx("colB")).GetAs<int32_t>());
  }
  delete key_schema;
}

// NOLINTNEXTLINE
TEST_F(GradingExecutorTest, SimpleSelectInsertTest) {
  // INSERT INTO empty_table2 SELECT colA, colB FROM test_1 WHERE colA > 500
  std::unique_ptr<AbstractPlanNode> scan_plan1;
  const Schema *out_schema1;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
    auto &schema = table_info->schema_;
    auto colA = MakeColumnValueExpression(schema, 0, "colA");
    auto colB = MakeColumnValueExpression(schema, 0, "colB");
    auto const600 = MakeConstantValueExpression(ValueFactory::GetIntegerValue(600));
    auto predicate = MakeComparisonExpression(colA, const600, ComparisonType::GreaterThan);
    out_schema1 = MakeOutputSchema({{"colA", colA}, {"colB", colB}});
    scan_plan1 = std::make_unique<SeqScanPlanNode>(out_schema1, predicate, table_info->oid_);
  }
  std::unique_ptr<AbstractPlanNode> insert_plan;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("empty_table2");
    insert_plan = std::make_unique<InsertPlanNode>(scan_plan1.get(), table_info->oid_);
  }

  GetExecutionEngine()->Execute(insert_plan.get(), nullptr, GetTxn(), GetExecutorContext());
  Schema *key_schema = ParseCreateStatement("a bigint");
  GenericComparator<8> comparator(key_schema);
  auto index_info = GetExecutorContext()->GetCatalog()->CreateIndex<GenericKey<8>, RID, GenericComparator<8>>(
      GetTxn(), "index1", "empty_table2", GetExecutorContext()->GetCatalog()->GetTable("empty_table2")->schema_,
      *key_schema, {0}, 8);

  // Now iterate through both tables, and make sure they have the same data
  std::unique_ptr<AbstractPlanNode> scan_plan2;
  const Schema *out_schema2;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("empty_table2");
    auto &schema = table_info->schema_;
    auto colA = MakeColumnValueExpression(schema, 0, "colA");
    auto colB = MakeColumnValueExpression(schema, 0, "colB");
    out_schema2 = MakeOutputSchema({{"colA", colA}, {"colB", colB}});
    scan_plan2 = std::make_unique<SeqScanPlanNode>(out_schema2, nullptr, table_info->oid_);
  }

  std::vector<Tuple> result_set1;
  std::vector<Tuple> result_set2;
  GetExecutionEngine()->Execute(scan_plan1.get(), &result_set1, GetTxn(), GetExecutorContext());
  GetExecutionEngine()->Execute(scan_plan2.get(), &result_set2, GetTxn(), GetExecutorContext());

  ASSERT_EQ(result_set1.size(), result_set2.size());
  for (size_t i = 0; i < result_set1.size(); ++i) {
    ASSERT_EQ(result_set1[i].GetValue(out_schema1, out_schema1->GetColIdx("colA")).GetAs<int32_t>(),
              result_set2[i].GetValue(out_schema2, out_schema2->GetColIdx("colA")).GetAs<int32_t>());
    ASSERT_EQ(result_set1[i].GetValue(out_schema1, out_schema1->GetColIdx("colB")).GetAs<int32_t>(),
              result_set2[i].GetValue(out_schema2, out_schema2->GetColIdx("colB")).GetAs<int32_t>());
  }
  ASSERT_EQ(result_set1.size(), 399);

  std::vector<RID> rids;
  for (auto &i : result_set2) {
    rids.clear();
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("empty_table2");
    auto index_key = i.KeyFromTuple(table_info->schema_, index_info->key_schema_, index_info->index_->GetKeyAttrs());
    index_info->index_->ScanKey(index_key, &rids, GetTxn());
    Tuple indexed_tuple;
    auto fetch_tuple = table_info->table_->GetTuple(rids[0], &indexed_tuple, GetTxn());

    ASSERT_TRUE(fetch_tuple);
    ASSERT_EQ(indexed_tuple.GetValue(out_schema2, out_schema2->GetColIdx("colA")).GetAs<int32_t>(),
              i.GetValue(out_schema2, out_schema2->GetColIdx("colA")).GetAs<int32_t>());
    ASSERT_EQ(indexed_tuple.GetValue(out_schema2, out_schema2->GetColIdx("colB")).GetAs<int32_t>(),
              i.GetValue(out_schema2, out_schema2->GetColIdx("colB")).GetAs<int32_t>());
  }

  delete key_schema;
}

// NOLINTNEXTLINE
TEST_F(GradingExecutorTest, SimpleUpdateTest) {
  // INSERT INTO empty_table2 SELECT colA, colA FROM test_1 WHERE colA < 50
  // UPDATE empty_table2 SET colA = colA+10 WHERE colA < 50
  std::unique_ptr<AbstractPlanNode> scan_plan1;
  const Schema *out_schema1;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
    auto &schema = table_info->schema_;
    auto colA = MakeColumnValueExpression(schema, 0, "colA");
    auto const600 = MakeConstantValueExpression(ValueFactory::GetIntegerValue(50));
    auto predicate = MakeComparisonExpression(colA, const600, ComparisonType::LessThan);
    out_schema1 = MakeOutputSchema({{"colA", colA}, {"colA", colA}});
    scan_plan1 = std::make_unique<SeqScanPlanNode>(out_schema1, predicate, table_info->oid_);
  }
  std::unique_ptr<AbstractPlanNode> insert_plan;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("empty_table2");
    insert_plan = std::make_unique<InsertPlanNode>(scan_plan1.get(), table_info->oid_);
  }

  GetExecutionEngine()->Execute(insert_plan.get(), nullptr, GetTxn(), GetExecutorContext());

  // Construct query plan
  auto table_info = GetExecutorContext()->GetCatalog()->GetTable("empty_table2");
  auto &schema = table_info->schema_;
  auto colA = MakeColumnValueExpression(schema, 0, "colA");
  auto colB = MakeColumnValueExpression(schema, 0, "colB");
  auto const50 = MakeConstantValueExpression(ValueFactory::GetIntegerValue(50));
  auto predicate = MakeComparisonExpression(colA, const50, ComparisonType::LessThan);
  auto out_empty_schema = MakeOutputSchema({{"colA", colA}, {"colB", colB}});
  auto scan_empty_plan = std::make_unique<SeqScanPlanNode>(out_empty_schema, predicate, table_info->oid_);

  // Create Indexes for col1 and col2
  Schema *key_schema = ParseCreateStatement("a int");
  GenericComparator<8> comparator(key_schema);
  GetExecutorContext()->GetCatalog()->CreateIndex<GenericKey<8>, RID, GenericComparator<8>>(
      GetTxn(), "index1", "empty_table2", GetExecutorContext()->GetCatalog()->GetTable("empty_table2")->schema_,
      *key_schema, {0}, 8);
  auto index_info_2 = GetExecutorContext()->GetCatalog()->CreateIndex<GenericKey<8>, RID, GenericComparator<8>>(
      GetTxn(), "index2", "empty_table2", GetExecutorContext()->GetCatalog()->GetTable("empty_table2")->schema_,
      *key_schema, {1}, 8);

  std::unique_ptr<AbstractPlanNode> scan_plan2;
  const Schema *out_schema2;
  {
    out_schema2 = MakeOutputSchema({{"colA", colA}, {"colB", colB}});
    scan_plan2 = std::make_unique<SeqScanPlanNode>(out_schema2, nullptr, table_info->oid_);
  }

  std::vector<Tuple> result_set2;
  GetExecutionEngine()->Execute(scan_plan2.get(), &result_set2, GetTxn(), GetExecutorContext());

  auto txn = GetTxnManager()->Begin();
  auto exec_ctx = std::make_unique<ExecutorContext>(txn, GetCatalog(), GetBPM(), GetTxnManager(), nullptr);

  std::unordered_map<uint32_t, UpdateInfo> update_attrs;
  update_attrs.insert(std::make_pair(0, UpdateInfo(UpdateType::Add, 10)));
  std::unique_ptr<AbstractPlanNode> update_plan;
  { update_plan = std::make_unique<UpdatePlanNode>(scan_empty_plan.get(), table_info->oid_, update_attrs); }

  std::vector<Tuple> result_set;
  GetExecutionEngine()->Execute(update_plan.get(), &result_set, txn, exec_ctx.get());

  GetTxnManager()->Commit(txn);
  delete txn;

  std::vector<RID> rids;
  for (int32_t i = 0; i < 50; ++i) {
    rids.clear();
    Tuple key = Tuple({Value(TypeId::INTEGER, i)}, key_schema);
    LOG_DEBUG("key:%d", key.GetValue(key_schema, 0).GetAs<uint32_t>());
    index_info_2->index_->ScanKey(key, &rids, GetTxn());
    Tuple indexed_tuple;
    auto fetch_tuple = table_info->table_->GetTuple(rids[0], &indexed_tuple, GetTxn());
    ASSERT_TRUE(fetch_tuple);
    auto cola_val = indexed_tuple.GetValue(&schema, 0).GetAs<uint32_t>();
    auto colb_val = indexed_tuple.GetValue(&schema, 1).GetAs<uint32_t>();
    ASSERT_TRUE(cola_val == colb_val + 10);
  }
  delete key_schema;
}

// NOLINTNEXTLINE
TEST_F(GradingExecutorTest, SimpleDeleteTest) {
  // SELECT colA FROM test_1 WHERE colA < 50
  // DELETE FROM test_1 WHERE colA < 50
  // SELECT colA FROM test_1 WHERE colA < 50

  // Construct query plan
  auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
  auto &schema = table_info->schema_;
  auto colA = MakeColumnValueExpression(schema, 0, "colA");
  auto const50 = MakeConstantValueExpression(ValueFactory::GetIntegerValue(50));
  auto predicate = MakeComparisonExpression(colA, const50, ComparisonType::LessThan);
  auto out_schema1 = MakeOutputSchema({{"colA", colA}});
  auto scan_plan1 = std::make_unique<SeqScanPlanNode>(out_schema1, predicate, table_info->oid_);
  // index
  Schema *key_schema = ParseCreateStatement("a bigint");
  GenericComparator<8> comparator(key_schema);
  auto index_info = GetExecutorContext()->GetCatalog()->CreateIndex<GenericKey<8>, RID, GenericComparator<8>>(
      GetTxn(), "index1", "test_1", GetExecutorContext()->GetCatalog()->GetTable("test_1")->schema_, *key_schema, {0},
      8);

  // Execute
  std::vector<Tuple> result_set;
  GetExecutionEngine()->Execute(scan_plan1.get(), &result_set, GetTxn(), GetExecutorContext());

  // Verify
  for (const auto &tuple : result_set) {
    ASSERT_TRUE(tuple.GetValue(out_schema1, out_schema1->GetColIdx("colA")).GetAs<int32_t>() < 50);
  }
  ASSERT_EQ(result_set.size(), 50);
  Tuple index_key = Tuple(result_set[0]);

  auto txn = GetTxnManager()->Begin();
  auto exec_ctx = std::make_unique<ExecutorContext>(txn, GetCatalog(), GetBPM(), nullptr, nullptr);
  std::unique_ptr<AbstractPlanNode> delete_plan;
  { delete_plan = std::make_unique<DeletePlanNode>(scan_plan1.get(), table_info->oid_); }
  GetExecutionEngine()->Execute(delete_plan.get(), nullptr, txn, exec_ctx.get());

  GetTxnManager()->Commit(txn);
  delete txn;

  result_set.clear();
  GetExecutionEngine()->Execute(scan_plan1.get(), &result_set, GetTxn(), GetExecutorContext());
  ASSERT_TRUE(result_set.empty());

  std::vector<RID> rids;

  index_info->index_->ScanKey(index_key, &rids, GetTxn());
  ASSERT_TRUE(rids.empty());

  delete key_schema;
}

// NOLINTNEXTLINE
TEST_F(GradingExecutorTest, SimpleNestedLoopJoinTest) {
  // SELECT test_1.colA, test_1.colB, test_2.col1, test_2.col3 FROM test_1 JOIN test_2 ON test_1.colA = test_2.col1 AND
  // test_1.colA < 50
  std::unique_ptr<AbstractPlanNode> scan_plan1;
  const Schema *out_schema1;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
    auto &schema = table_info->schema_;
    auto colA = MakeColumnValueExpression(schema, 0, "colA");
    auto colB = MakeColumnValueExpression(schema, 0, "colB");
    auto const50 = MakeConstantValueExpression(ValueFactory::GetIntegerValue(50));
    auto predicate = MakeComparisonExpression(colA, const50, ComparisonType::LessThan);
    out_schema1 = MakeOutputSchema({{"colA", colA}, {"colB", colB}});
    scan_plan1 = std::make_unique<SeqScanPlanNode>(out_schema1, predicate, table_info->oid_);
  }
  std::unique_ptr<AbstractPlanNode> scan_plan2;
  const Schema *out_schema2;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_2");
    auto &schema = table_info->schema_;
    auto col1 = MakeColumnValueExpression(schema, 0, "col1");
    auto col3 = MakeColumnValueExpression(schema, 0, "col3");
    out_schema2 = MakeOutputSchema({{"col1", col1}, {"col3", col3}});
    scan_plan2 = std::make_unique<SeqScanPlanNode>(out_schema2, nullptr, table_info->oid_);
  }
  std::unique_ptr<NestedLoopJoinPlanNode> join_plan;
  const Schema *out_final;
  {
    // colA and colB have a tuple index of 0 because they are the left side of the join
    auto colA = MakeColumnValueExpression(*out_schema1, 0, "colA");
    auto colB = MakeColumnValueExpression(*out_schema1, 0, "colB");
    // col1 and col2 have a tuple index of 1 because they are the right side of the join
    auto col1 = MakeColumnValueExpression(*out_schema2, 1, "col1");
    auto col3 = MakeColumnValueExpression(*out_schema2, 1, "col3");
    auto predicate = MakeComparisonExpression(colA, col1, ComparisonType::Equal);
    out_final = MakeOutputSchema({{"colA", colA}, {"colB", colB}, {"col1", col1}, {"col3", col3}});
    join_plan = std::make_unique<NestedLoopJoinPlanNode>(
        out_final, std::vector<const AbstractPlanNode *>{scan_plan1.get(), scan_plan2.get()}, predicate);
  }

  std::vector<Tuple> result_set;
  GetExecutionEngine()->Execute(join_plan.get(), &result_set, GetTxn(), GetExecutorContext());
  ASSERT_EQ(result_set.size(), 50);

  for (const auto &tuple : result_set) {
    auto col_a_val = tuple.GetValue(out_final, out_final->GetColIdx("colA")).GetAs<int32_t>();
    auto col_1_val = tuple.GetValue(out_final, out_final->GetColIdx("col1")).GetAs<int16_t>();
    ASSERT_EQ(col_a_val, col_1_val);
    ASSERT_LT(col_a_val, 50);
  }
}

// NOLINTNEXTLINE
TEST_F(GradingExecutorTest, SimpleAggregationTest) {
  // SELECT COUNT(colA), SUM(colA), min(colA), max(colA) from test_1;
  std::unique_ptr<AbstractPlanNode> scan_plan;
  const Schema *scan_schema;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
    auto &schema = table_info->schema_;
    auto colA = MakeColumnValueExpression(schema, 0, "colA");
    scan_schema = MakeOutputSchema({{"colA", colA}});
    scan_plan = std::make_unique<SeqScanPlanNode>(scan_schema, nullptr, table_info->oid_);
  }

  std::unique_ptr<AbstractPlanNode> agg_plan;
  const Schema *agg_schema;
  {
    const AbstractExpression *colA = MakeColumnValueExpression(*scan_schema, 0, "colA");
    const AbstractExpression *countA = MakeAggregateValueExpression(false, 0);
    const AbstractExpression *sumA = MakeAggregateValueExpression(false, 1);
    const AbstractExpression *minA = MakeAggregateValueExpression(false, 2);
    const AbstractExpression *maxA = MakeAggregateValueExpression(false, 3);

    agg_schema = MakeOutputSchema({{"countA", countA}, {"sumA", sumA}, {"minA", minA}, {"maxA", maxA}});
    agg_plan = std::make_unique<AggregationPlanNode>(
        agg_schema, scan_plan.get(), nullptr, std::vector<const AbstractExpression *>{},
        std::vector<const AbstractExpression *>{colA, colA, colA, colA},
        std::vector<AggregationType>{AggregationType::CountAggregate, AggregationType::SumAggregate,
                                     AggregationType::MinAggregate, AggregationType::MaxAggregate});
  }
  std::vector<Tuple> result_set;
  GetExecutionEngine()->Execute(agg_plan.get(), &result_set, GetTxn(), GetExecutorContext());

  auto countA_val = result_set[0].GetValue(agg_schema, agg_schema->GetColIdx("countA")).GetAs<int32_t>();
  auto sumA_val = result_set[0].GetValue(agg_schema, agg_schema->GetColIdx("sumA")).GetAs<int32_t>();
  auto minA_val = result_set[0].GetValue(agg_schema, agg_schema->GetColIdx("minA")).GetAs<int32_t>();
  auto maxA_val = result_set[0].GetValue(agg_schema, agg_schema->GetColIdx("maxA")).GetAs<int32_t>();
  // Should count all tuples
  ASSERT_EQ(countA_val, TEST1_SIZE);
  // Should sum from 0 to TEST1_SIZE
  ASSERT_EQ(sumA_val, TEST1_SIZE * (TEST1_SIZE - 1) / 2);
  // Minimum should be 0
  ASSERT_EQ(minA_val, 0);
  // Maximum should be TEST1_SIZE - 1
  ASSERT_EQ(maxA_val, TEST1_SIZE - 1);

  ASSERT_EQ(result_set.size(), 1);
}

// NOLINTNEXTLINE
TEST_F(GradingExecutorTest, SimpleGroupByAggregation) {
  // SELECT count(colA), colB, sum(colC) FROM test_1 Group By colB HAVING count(colA) > 100
  std::unique_ptr<AbstractPlanNode> scan_plan;
  const Schema *scan_schema;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
    auto &schema = table_info->schema_;
    auto colA = MakeColumnValueExpression(schema, 0, "colA");
    auto colB = MakeColumnValueExpression(schema, 0, "colB");
    auto colC = MakeColumnValueExpression(schema, 0, "colC");
    scan_schema = MakeOutputSchema({{"colA", colA}, {"colB", colB}, {"colC", colC}});
    scan_plan = std::make_unique<SeqScanPlanNode>(scan_schema, nullptr, table_info->oid_);
  }

  std::unique_ptr<AbstractPlanNode> agg_plan;
  const Schema *agg_schema;
  {
    const AbstractExpression *colA = MakeColumnValueExpression(*scan_schema, 0, "colA");
    const AbstractExpression *colB = MakeColumnValueExpression(*scan_schema, 0, "colB");
    const AbstractExpression *colC = MakeColumnValueExpression(*scan_schema, 0, "colC");
    // Make group bys
    std::vector<const AbstractExpression *> group_by_cols{colB};
    const AbstractExpression *groupbyB = MakeAggregateValueExpression(true, 0);
    // Make aggregates
    std::vector<const AbstractExpression *> aggregate_cols{colA, colC};
    std::vector<AggregationType> agg_types{AggregationType::CountAggregate, AggregationType::SumAggregate};
    const AbstractExpression *countA = MakeAggregateValueExpression(false, 0);
    const AbstractExpression *sumC = MakeAggregateValueExpression(false, 1);
    // Make having clause
    const AbstractExpression *having = MakeComparisonExpression(
        countA, MakeConstantValueExpression(ValueFactory::GetIntegerValue(100)), ComparisonType::GreaterThan);

    // Create plan
    agg_schema = MakeOutputSchema({{"countA", countA}, {"colB", groupbyB}, {"sumC", sumC}});
    agg_plan = std::make_unique<AggregationPlanNode>(agg_schema, scan_plan.get(), having, std::move(group_by_cols),
                                                     std::move(aggregate_cols), std::move(agg_types));
  }

  std::vector<Tuple> result_set;
  GetExecutionEngine()->Execute(agg_plan.get(), &result_set, GetTxn(), GetExecutorContext());

  std::unordered_set<int32_t> encountered;
  for (const auto &tuple : result_set) {
    // Should have countA > 100
    ASSERT_GT(tuple.GetValue(agg_schema, agg_schema->GetColIdx("countA")).GetAs<int32_t>(), 100);
    // Should have unique colBs.
    auto colB = tuple.GetValue(agg_schema, agg_schema->GetColIdx("colB")).GetAs<int32_t>();
    ASSERT_EQ(encountered.count(colB), 0);
    encountered.insert(colB);
    // Sanity check: ColB should also be within [0, 10).
    ASSERT_TRUE(0 <= colB && colB < 10);
  }
}

// NOLINTNEXTLINE
TEST_F(GradingExecutorTest, SimpleNestedIndexJoinTest) {
  // SELECT test_1.colA, test_1.colB, test_3.col1, test_3.col3 FROM test_1 JOIN test_3 ON test_1.colA = test_3.col1
  std::unique_ptr<AbstractPlanNode> scan_plan1;
  const Schema *outer_schema1;
  auto &schema_outer = GetExecutorContext()->GetCatalog()->GetTable("test_1")->schema_;
  auto outer_colA = MakeColumnValueExpression(schema_outer, 0, "colA");
  auto outer_colB = MakeColumnValueExpression(schema_outer, 0, "colB");
  auto outer_colC = MakeColumnValueExpression(schema_outer, 0, "colC");
  auto outer_colD = MakeColumnValueExpression(schema_outer, 0, "colD");
  const Schema *outer_out_schema1 =
      MakeOutputSchema({{"colA", outer_colA}, {"colB", outer_colB}, {"colC", outer_colC}, {"colD", outer_colD}});

  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
    auto &schema = table_info->schema_;
    auto colA = MakeColumnValueExpression(schema, 0, "colA");
    auto colB = MakeColumnValueExpression(schema, 0, "colB");
    outer_schema1 = MakeOutputSchema({{"colA", colA}, {"colB", colB}});
    scan_plan1 = std::make_unique<SeqScanPlanNode>(outer_out_schema1, nullptr, table_info->oid_);
  }
  const Schema *out_schema2;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_3");
    auto &schema = table_info->schema_;
    auto col1 = MakeColumnValueExpression(schema, 0, "col1");
    auto col3 = MakeColumnValueExpression(schema, 0, "col3");
    out_schema2 = MakeOutputSchema({{"col1", col1}, {"col3", col3}});
  }
  std::unique_ptr<NestedIndexJoinPlanNode> join_plan;
  const Schema *out_final;
  Schema *key_schema = ParseCreateStatement("a int");
  {
    // colA and colB have a tuple index of 0 because they are the left side of the join
    auto colA = MakeColumnValueExpression(*outer_schema1, 0, "colA");
    auto colB = MakeColumnValueExpression(*outer_schema1, 0, "colB");
    // col1 and col2 have a tuple index of 1 because they are the right side of the join
    auto col1 = MakeColumnValueExpression(*out_schema2, 1, "col1");
    auto col3 = MakeColumnValueExpression(*out_schema2, 1, "col3");
    auto predicate = MakeComparisonExpression(colA, col1, ComparisonType::Equal);
    out_final = MakeOutputSchema({{"colA", colA}, {"colB", colB}, {"col1", col1}, {"col3", col3}});

    auto inner_table_info = GetExecutorContext()->GetCatalog()->GetTable("test_3");
    auto inner_table_oid = inner_table_info->oid_;
    GenericComparator<8> comparator(key_schema);
    // Create index for inner table
    auto index_info = GetExecutorContext()->GetCatalog()->CreateIndex<GenericKey<8>, RID, GenericComparator<8>>(
        GetTxn(), "index1", "test_3", inner_table_info->schema_, *key_schema, {0}, 1);

    join_plan = std::make_unique<NestedIndexJoinPlanNode>(
        out_final, std::vector<const AbstractPlanNode *>{scan_plan1.get()}, predicate, inner_table_oid,
        index_info->name_, outer_schema1, out_schema2);
  }

  std::vector<Tuple> result_set;
  GetExecutionEngine()->Execute(join_plan.get(), &result_set, GetTxn(), GetExecutorContext());
  ASSERT_EQ(result_set.size(), 100);

  for (const auto &tuple : result_set) {
    ASSERT_EQ(tuple.GetValue(out_final, out_final->GetColIdx("colA")).GetAs<int32_t>(),
              tuple.GetValue(out_final, out_final->GetColIdx("col1")).GetAs<int16_t>());
  }

  delete key_schema;
}

// NOLINTNEXTLINE
TEST_F(GradingExecutorTest, SchemaChangeSeqScan) {
  // INSERT INTO empty_table2 SELECT colA, colB FROM test_1 WHERE colA > 600
  // compare: SELECT colA as outA, colB as outB FROM empty_table2
  std::unique_ptr<AbstractPlanNode> scan_plan1;
  const Schema *out_schema1;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
    auto &schema = table_info->schema_;
    auto colA = MakeColumnValueExpression(schema, 0, "colA");
    auto colB = MakeColumnValueExpression(schema, 0, "colB");
    auto const600 = MakeConstantValueExpression(ValueFactory::GetIntegerValue(600));
    auto predicate = MakeComparisonExpression(colA, const600, ComparisonType::GreaterThan);
    out_schema1 = MakeOutputSchema({{"colA", colA}, {"colB", colB}});
    scan_plan1 = std::make_unique<SeqScanPlanNode>(out_schema1, predicate, table_info->oid_);
  }

  std::unique_ptr<AbstractPlanNode> insert_plan;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("empty_table2");
    insert_plan = std::make_unique<InsertPlanNode>(scan_plan1.get(), table_info->oid_);
  }

  std::vector<Tuple> result_set;
  GetExecutionEngine()->Execute(insert_plan.get(), &result_set, GetTxn(), GetExecutorContext());
  ASSERT_TRUE(result_set.empty());

  // Now iterate through both tables, and make sure they have the same data
  std::unique_ptr<AbstractPlanNode> scan_plan2;
  const Schema *out_schema2;
  {
    auto table_info2 = GetExecutorContext()->GetCatalog()->GetTable("empty_table2");
    auto table_info3 = GetExecutorContext()->GetCatalog()->GetTable("empty_table3");
    auto &schema3 = table_info3->schema_;
    auto outA = MakeColumnValueExpression(schema3, 0, "outA");
    auto outB = MakeColumnValueExpression(schema3, 0, "outB");
    out_schema2 = MakeOutputSchema({{"outA", outA}, {"outB", outB}});
    scan_plan2 = std::make_unique<SeqScanPlanNode>(out_schema2, nullptr, table_info2->oid_);
  }

  std::vector<Tuple> result_set1;
  GetExecutionEngine()->Execute(scan_plan1.get(), &result_set1, GetTxn(), GetExecutorContext());

  std::vector<Tuple> result_set2;
  GetExecutionEngine()->Execute(scan_plan2.get(), &result_set2, GetTxn(), GetExecutorContext());

  ASSERT_EQ(result_set1.size(), 399);
  ASSERT_EQ(result_set2.size(), 399);
  for (size_t i = 0; i < result_set1.size(); ++i) {
    ASSERT_EQ(result_set1[i].GetValue(out_schema1, out_schema1->GetColIdx("colA")).GetAs<int32_t>(),
              result_set2[i].GetValue(out_schema2, out_schema2->GetColIdx("outA")).GetAs<int32_t>());
    ASSERT_EQ(result_set1[i].GetValue(out_schema1, out_schema1->GetColIdx("colB")).GetAs<int32_t>(),
              result_set2[i].GetValue(out_schema2, out_schema2->GetColIdx("outB")).GetAs<int32_t>());
  }
}

TEST_F(GradingExecutorTest, IntegratedTest) {
  // scan -> join -> aggregate
  std::unique_ptr<AbstractPlanNode> scan_plan1;
  const Schema *out_schema1;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
    auto &schema = table_info->schema_;
    auto colA = MakeColumnValueExpression(schema, 0, "colA");
    auto colB = MakeColumnValueExpression(schema, 0, "colB");
    out_schema1 = MakeOutputSchema({{"colA", colA}, {"colB", colB}});
    scan_plan1 = std::make_unique<SeqScanPlanNode>(out_schema1, nullptr, table_info->oid_);
  }
  std::unique_ptr<AbstractPlanNode> scan_plan2;
  const Schema *out_schema2;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_2");
    auto &schema = table_info->schema_;
    auto col1 = MakeColumnValueExpression(schema, 0, "col1");
    auto col2 = MakeColumnValueExpression(schema, 0, "col2");
    out_schema2 = MakeOutputSchema({{"col1", col1}, {"col2", col2}});
    scan_plan2 = std::make_unique<SeqScanPlanNode>(out_schema2, nullptr, table_info->oid_);
  }
  std::unique_ptr<NestedLoopJoinPlanNode> join_plan;
  const Schema *out_final;
  {
    // colA and colB have a tuple index of 0 because they are the left side of the join
    auto colA = MakeColumnValueExpression(*out_schema1, 0, "colA");
    auto colB = MakeColumnValueExpression(*out_schema1, 0, "colB");
    // col1 and col2 have a tuple index of 1 because they are the right side of the join
    auto col1 = MakeColumnValueExpression(*out_schema2, 1, "col1");
    auto col2 = MakeColumnValueExpression(*out_schema2, 1, "col2");
    std::vector<const AbstractExpression *> left_keys{colA};
    std::vector<const AbstractExpression *> right_keys{col1};
    auto predicate = MakeComparisonExpression(colA, col1, ComparisonType::Equal);
    out_final = MakeOutputSchema({{"colA", colA}, {"colB", colB}, {"col1", col1}, {"col2", col2}});
    join_plan = std::make_unique<NestedLoopJoinPlanNode>(
        out_final, std::vector<const AbstractPlanNode *>{scan_plan1.get(), scan_plan2.get()}, predicate);
  }

  std::unique_ptr<AbstractPlanNode> agg_plan;
  const Schema *agg_schema;
  {
    const AbstractExpression *colA = MakeColumnValueExpression(*out_final, 0, "colA");
    const AbstractExpression *countA = MakeAggregateValueExpression(false, 0);
    const AbstractExpression *sumA = MakeAggregateValueExpression(false, 1);
    const AbstractExpression *minA = MakeAggregateValueExpression(false, 2);
    const AbstractExpression *maxA = MakeAggregateValueExpression(false, 3);

    agg_schema = MakeOutputSchema({{"countA", countA}, {"sumA", sumA}, {"minA", minA}, {"maxA", maxA}});
    agg_plan = std::make_unique<AggregationPlanNode>(
        agg_schema, join_plan.get(), nullptr, std::vector<const AbstractExpression *>{},
        std::vector<const AbstractExpression *>{colA, colA, colA, colA},
        std::vector<AggregationType>{AggregationType::CountAggregate, AggregationType::SumAggregate,
                                     AggregationType::MinAggregate, AggregationType::MaxAggregate});
  }

  std::vector<Tuple> result_set1;
  GetExecutionEngine()->Execute(agg_plan.get(), &result_set1, GetTxn(), GetExecutorContext());

  ASSERT_EQ(result_set1.size(), 1);
  auto tuple = result_set1[0];
  auto countA_val = tuple.GetValue(agg_schema, agg_schema->GetColIdx("countA")).GetAs<int32_t>();
  auto sumA_val = tuple.GetValue(agg_schema, agg_schema->GetColIdx("sumA")).GetAs<int32_t>();
  auto minA_val = tuple.GetValue(agg_schema, agg_schema->GetColIdx("minA")).GetAs<int32_t>();
  auto maxA_val = tuple.GetValue(agg_schema, agg_schema->GetColIdx("maxA")).GetAs<int32_t>();
  // Should count all tuples
  ASSERT_EQ(countA_val, TEST2_SIZE);
  // Should sum from 0 to TEST2_SIZE
  ASSERT_EQ(sumA_val, TEST2_SIZE * (TEST2_SIZE - 1) / 2);
  // Minimum should be 0
  ASSERT_EQ(minA_val, 0);
  // Maximum should be TEST2_SIZE - 1
  ASSERT_EQ(maxA_val, TEST2_SIZE - 1);
}

}  // namespace bustub

```

## 4. 总结

 proj3相对简单，涉及到的内容是如何在database中执行各种操作，**如SELECT, INSERT, UPDATE, DELETE, JOIN, LIMIT 等**。处理模型采用Iterator Model（不过其实内部实现上并没有完全遵守，iterator model的优点在于可以pipeline，但实现起来很多操作都是pipeline breaker）。

 下一个project终于到了最感兴趣的并发控制，加油早点做完。

 最后，贴一个满分通过：

 ![](https://pic.imgdb.cn/item/61e92d242ab3f51d915d2929.jpg)
