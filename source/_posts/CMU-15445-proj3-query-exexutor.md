---
title: CMU-15445-proj3(QueryExecutor)
date: 2022-01-20 15:54:14
categories: CMU15445
tags:
---



## 0. 前言 & 要求

断更了3个月，终于忙完了毕设，现在开始补课。文本为 CMU15445 - Proj3 的题解 。Proj3 为QueryExecutor，要求实现多个 executors，从而达到以下操作：

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

## 3. 总结

 proj3相对简单，涉及到的内容是如何在database中执行各种操作，**如SELECT, INSERT, UPDATE, DELETE, JOIN, LIMIT 等**。处理模型采用Iterator Model（不过其实内部实现上并没有完全遵守，iterator model的优点在于可以pipeline，但实现起来很多操作都是pipeline breaker）。

 下一个project终于到了最感兴趣的并发控制，加油早点做完。

 最后，贴一个满分通过：

 ![](https://pic.imgdb.cn/item/61e92d242ab3f51d915d2929.jpg)
