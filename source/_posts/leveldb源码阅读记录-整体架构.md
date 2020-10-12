---
title: leveldb源码阅读记录-整体架构
categories: leveldb
abbrlink: 1a545f48
date: 2020-10-12 09:00:06
tags:
---

## 0. 前言

本系列文章是对leveldb源码分析的笔记，基本会包含leveldb中的方方面面。阅读过程中，参照网上各博客并加上了个人的理解，所以难免有错，如有错误还请告知。

如果你是刚开始看leveldb，我希望你对LSM有一定的概念。为了避免过度陷入代码细节，本文将解释leveldb的整体设计架构，各个组件的宏观设计，这样能在心里对leveldb有个轮廓，也能指导我们从哪里入手代码。

本文非写者编辑，转载自：http://cighao.com/2016/08/14/leveldb-source-analysis-02-structure/

<!--more-->

## 1\. 整体架构

LevelDB 作为存储系统，数据记录的存储介质包括内存以及磁盘文件，当 LevelDB 运行了一段时间，此时我们给 LevelDB 进行透视拍照，会看到如下一番景象，这也就是 LevelDB 的结构图。

![LevelDB结构](http://cighao.com/img/articles/leveldb/leveldb-02-01.png)

从图中可以看出，构成 LevelDB 静态结构的包括六个主要部分：内存中的 **MemTable** 和 **Immutable MemTable** 以及磁盘上的几种主要文件：**Current文件**，**Manifest文件**，**log文件**以及 **SSTable 文件**。当然，LevelDB 除了这六个主要部分还有一些辅助的文件，但是以上六个文件和数据结构是 LevelDB 的主体构成元素。

当应用写入一条Key:Value记录的时候，LevelDb会先往log文件里写入，成功后将记录插进Memtable中，这样基本就算完成了写入操作，因为一次写入操作只涉及一次磁盘顺序写和一次内存写入，所以这是为何说LevelDb写入速度极快的主要原因。

SSTable 就是由内存中的数据不断导出并进行 Compaction 操作后形成的，而且 SSTable 的所有文件是一种层级结构，第一层为Level 0，第二层为Level 1，依次类推，层级逐渐增高。

SSTable 中的文件(后缀为.sst)是 key 有序的，就是说在文件中小 key 记录排在大 key 记录之前，除了 level 0，各个 level 的 SSTable 都是如此。

## 2\. manifest文件

SSTable 中的某个文件属于特定层级，而且其存储的记录是 key 有序的，那么必然有文件中的最小 key 和最大 key，这是非常重要的信息，LevelDB 应该记下这些信息。**manifest** 就是干这个的，它记载了 SSTable 各个文件的管理信息，比如属于哪个 level，文件名称叫啥，最小 key 和最大 key 各自是多少。下图是 manifest 所存储内容的示意：

![Manifest存储示意图](http://cighao.com/img/articles/leveldb/leveldb-02-02.png)

图中只显示了两个文件（manifest 会记载所有 SSTable 文件的这些信息），即 level 0 的 Test1.sst 和 Test2.sst 文件，同时记载了这些文件各自对应的 key 范围，比如 Test1.sst 的 key 范围是 “abc” 到 “hello”，而文件 Test2.sst 的 key 范围是 “bbc” 到 “world”，可以看出两者的 key 范围是有重叠的。

## 3\. current 文件

这个文件的内容只有一个信息，就是记载当前的 manifest 文件名。因为在 LevleDB 的运行过程中，随着 compaction 的进行，SSTable 文件会发生变化，会有新的文件产生，老的文件被废弃，manifest 也会跟着反映这种变化，此时往往会新生成 manifest 文件来记载这种变化，而 current 则用来指出哪个 manifest 文件才是我们关心的那个 manifest文件。

## 4\. log 文件

log 文件在 LevelDB 中的主要作用是系统故障恢复时，能够保证不会丢失数据。因为在将记录写入内存的 memtable 之前，会先写入 log 文件，这样即使系统发生故障，memtable 中的数据没有来得及 dump 到磁盘的 SSTable 文件，LevelDB 也可以根据 log 文件恢复内存的 Memtable 数据结构内容，不会造成系统丢失数据。

LevelDB 对于一个 log 文件，会把它切割成以 32K 为单位的物理 Block，每次读取以一个 Block 作为基本读取单位，下图展示的 log 文件由3个 Block 构成，所以从物理布局来讲，一个 log 文件就是由连续的 32K 大小 Block 构成的。

![log文件布局](http://cighao.com/img/articles/leveldb/leveldb-02-03.png)

log 文件中的数据是以 block 为单位组织，写日志时，处于一致性考虑并没有按 block 单位写，每次更新均对 log 文件进行 IO，每次更新写入作为一个 record，每条 record 的结构如下图所示：

![记录结构](http://cighao.com/img/articles/leveldb/leveldb-02-04.png)

`checksum` 记录的是 “type” 和 “data” 字段的CRC校验，为了避免处理不完整或者是被破坏的数据，当 LevelDB 读取记录数据时候会对数据进行校验，如果发现和存储的 checksum 相同，说明数据完整无破坏，可以继续后续流程。

`length` 记录的是 record 内保存的 data 长度(小端对齐)。

`data` 记录的是 Key:Value 数值对.

`type` 字段则指出了每条记录的逻辑结构和 log 文件物理分块结构之间的关系，具体而言，主要有以下四种类型：`FULL`, `FIRST`, `MIDDLE`, `LAST`。如果记录类型是FULL，代表了当前记录内容完整地存储在一个物理Block里，没有被不同的物理Block切割开；如果记录被相邻的物理Block切割开，则类型会是其他三种类型中的一种。我们以图3.1所示的例子来具体说明。

假设目前存在三条记录，Record A，Record B 和 Record C，其中 Record A 大小为10K，Record B 大小为80K，Record C大小为12K，那么其在 log 文件中的逻辑布局会如上面的图所示。Record A是图中蓝色区域所示，因为大小为10K<32K，能够放在一个物理Block中，所以其类型为FULL；Record B 大小为80K，而Block 1因为放入了Record A，所以还剩下22K，不足以放下Record B，所以在Block 1的剩余部分放入Record B的开头一部分，类型标识为FIRST，代表了是一个记录的起始部分；Record B还有58K没有存储，这些只能依次放在后续的物理Block里面，因为Block 2大小只有32K，仍然放不下Record B的剩余部分，所以Block 2全部用来放Record B，且标识类型为MIDDLE，意思是这是Record B中间一段数据；Record B剩下的部分可以完全放在Block 3中，类型标识为LAST，代表了这是Record B的末尾数据；图中黄色的Record C因为大小为12K，Block 3剩下的空间足以全部放下它，所以其类型标识为FULL。

从这个小例子可以看出逻辑记录和物理Block之间的关系，LevelDB一次物理读取为一个Block，然后根据类型情况拼接出逻辑记录，供后续流程处理。

## 5\. SSTable 文件

LevelDB 不同层级有很多 SSTable 文件（以后缀.sst为特征），所有 .sst 文件内部布局都是一样的。上节介绍 log 文件是物理分块的，SSTable也一样会将文件划分为固定大小的物理存储块，但是两者逻辑布局大不相同，根本原因是：log文件中的记录是 Key 无序的，即先后记录的 key 大小没有明确大小关系，而 .sst 文件内部则是根据记录的 Key 由小到大排列的。

![.sst文件的分块结构](http://cighao.com/img/articles/leveldb/leveldb-02-05.png)

上图展示了一个 .sst 文件的物理划分结构，同 log 文件一样，也是划分为固定大小的存储块，每个 Block 分为三个部分：**数据存储区**，**Type 区**用于标识数据存储区是否采用了数据压缩算法（Snappy压缩或者无压缩），**CRC校验**则是数据校验码，用于判别数据在生成和传输中是否出错。

以上是.sst的物理布局，下面介绍.sst文件的逻辑布局，所谓逻辑布局，就是说尽管大家都是物理块，但是每一块存储什么内容，内部又有什么结构等。下图展示了.sst文件的内部逻辑解释。

![逻辑布局](http://cighao.com/img/articles/leveldb/leveldb-02-06.png)

从上图可以看出，从大的方面，可以将 .sst文 件划分为**数据存储区**和**数据管理区**，数据存储区存放实际的 key\-value 数据，数据管理区则提供一些索引指针等管理数据，目的是更快速便捷的查找相应的记录。两个区域都是在上述的分块基础上的，就是说文件的前面若干块实际存储 KV 数据，后面数据管理区存储管理数据。管理数据又分为四种不同类型：**Meta Block**，**MetaBlock 索引**和**数据索引块**以及一个**文件尾部块**。

`data_block`：实际存储的 KV 数据。
`meta_block`：每个 data\_block 对应一个 meta\_block，保存 data\_block 中的 key size/value size/kv counts 之类的统计信息，当前版本未实现。
`metaindex_block`：保存 meta\_block 的索引信息，当前版本未实现。
`index_block`：保存每个 data\_block 的 last\_key 及其在 SSTable 文件中的索引。block 中 entry 的 key 即是 last\_key(依赖于FindShortestSeparator()/FindShortSuccessor()的实现)，value即是该data\_block的BlockHandler（offset/size）。
`footer`：文件末尾固定长度的数据。保存着 metaindex\_block 和 index\_block 的索引信息(BlockHandler)，为达到固定的长度，添加 padding\_bytes。最后有8个字节的 magic 校验。

footer 的结构如下图所示：

![Footer](http://cighao.com/img/articles/leveldb/leveldb-02-07.png)

`metaindex_block_handle` 指出了 metaindex block 的起始位置和大小；`index_block_handle` 指出了 index Block 的起始地址和大小；这两个字段可以理解为索引的索引，是为了正确读出索引值而设立的；为达到固定的长度，添加 `padding_bytes`。最后有8个字节的 `magic` 校验。

下图是数据索引块的内部结构示意图：

![数据索引](http://cighao.com/img/articles/leveldb/leveldb-02-08.png)

Data Block 内的 KV 记录是按照 key 由小到大排列的，数据索引区的每条记录是对某个 Data Block 建立的索引信息，每条索引信息包含三个内容。以上图所示的数据块 i 的索引 index i 来说：

**第一个字段**：记录大于等于数据块i中最大的 key 值的那个 key。在索引里保存的这个 key 值未必一定是某条记录的 key，以上的例子来说，假设数据块i的最小key=“samecity”，最大key=“the best”；数据块i+1的最小key=“the fox”，最大key=“zoo”，那么对于数据块i的索引 index i 来说，其第一个字段记载大于等于数据块i的最大Key(“the best”) 同时要小于数据块i+1的最小Key(“the fox”)，所以例子中 index i 的第一个字段是 “the c”，这个是满足要求的；而 index i+1 的第一个字段则是 “zoo”，即数据块i+1的最大 key。

**第二个字段**：指出数据块 i 在 .sst 文件中的起始位置。

**第三个字段**：指出 Data Block i 的大小（有时候是有数据压缩的）。

上面主要介绍的是数据管理区的内部结构，下图是数据区的一个 Block 的数据部分布局。

![](http://cighao.com/img/articles/leveldb/leveldb-02-09.png)

从图中可以看出，其内部也分为两个部分，前面是一个个 KV 记录，其顺序是根据 key 值由小到大排列的，在Block尾部则是一些“重启点”（Restart Point），其实是一些指针，指出 Block 内容中的一些记录位置。

“重启点”是干什么的呢？Block内容里的KV记录是按照 key 大小有序的，这样的话，相邻的两条记录很可能 key 部分存在重叠，比如 key i=“the Car”，Key i+1=“the color”，那么两者存在重叠部分 “the c”，为了减少 key 的存储量，Key i+1 可以只存储和上一条 key 不同的部分 “olor”，两者的共同部分从 key i 中可以获得。记录的 key 在 Block 内容部分就是这么存储的，主要目的是减少存储开销。“重启点”的意思是：在这条记录开始，不再采取只记载不同的 key 部分，而是重新记录所有的 key 值，假设 key i+1 是一个重启点，那么 key 里面会完整存储 “the color”，而不是采用简略的“olor”方式。Block尾部就是指出哪些记录是这些重启点的。

其中记录的格式如下所示：

![记录格式](http://cighao.com/img/articles/leveldb/leveldb-02-10.png)

每个记录包含5个字段：

**key共享长度**：比如上面的 “olor” 记录， 其 key 和上一条记录共享的 key 部分长度是 “the c” 的长度，即5；

**key非共享长度**：对于“olor”来说，是4；

**value长度**：指出 key\-value 中 value 的长度，在后面的 value 内容字段中存储实际的 value 值；

**key非共享内容**：指实际存储 “olor” 这个 key 字符串；

**value内容**：存储实际的 value 值。

## 6\. Memtable

所有 KV 数据都是存储在 Memtable，Immutable Memtable 和 SSTable 中的，Immutable Memtable从结构上讲和 Memtable 是完全一样的，区别仅仅在于其是只读的，不允许写入操作，而 Memtable 则是允许写入和读取的。当 Memtable 写入的数据占用内存到达指定数量，则自动转换为 Immutable Memtable，等待 Dump 到磁盘中，系统会自动生成新的 Memtable 供写操作写入新数据，理解了 Memtable，那么 Immutable Memtable 自然不在话下。

LevelDB 的 MemTable 提供了将 KV 数据写入，删除以及读取 KV 记录的操作接口，但是事实上 Memtable 并不存在真正的删除操作,删除某个Key的Value在 Memtable 内是作为插入一条记录实施的，但是会打上一个 Key 的删除标记，真正的删除操作是Lazy的，会在以后的 Compaction 过程中去掉这个KV。

需要注意的是，LevelDB 的 Memtable 中KV对是根据key大小有序存储的，在系统插入新的KV时，LevelDB 要把这个KV插到合适的位置上以保持这种 Key 有序性。其实，LevelDB 的 Memtable 类只是一个接口类，真正的操作是通过背后的 SkipList 来做的，包括插入操作和读取操作等，所以 Memtable 的核心数据结构是一个 SkipList。

SkipList是由 William Pugh 发明。他在 Communications of the ACM June 1990, 33(6) 668\-676 发表了 Skip lists: a probabilistic alternative to balanced trees，在该论文中详细解释了SkipList的数据结构和插入删除操作。关于SkipList的详细介绍可以参考这篇文章：[http://www.cnblogs.com/xuqiang/archive/2011/05/22/2053516.html。](http://www.cnblogs.com/xuqiang/archive/2011/05/22/2053516.html。)

## 7\. 读记录

因为写入操作比较简单就不介绍了，LevelDB 的读取流程如下所示：

![LevelDB读取记录流程](http://cighao.com/img/articles/leveldb/leveldb-02-11.png)

LevelDB 首先会去查看内存中的 Memtable，如果 Memtable 中包含 key 及其对应的value，则返回 value 值即可；

如果在 Memtable 没有读到key，则接下来到同样处于内存中的 Immutable Memtable 中去读取，类似地，如果读到就返回，若是没有读到,那么只能万般无奈下从磁盘中的大量SSTable文件中查找。

因为SSTable数量较多，而且分成多个level，所以在SSTable中读数据是相当蜿蜒曲折的一段旅程。总的读取原则是这样的：首先从属于 level 0 的文件中查找，如果找到则返回对应的value值，如果没有找到那么到 level 1 中的文件中去找，如此循环往复，直到在某层 SSTable 文件中找到这个 key 对应的 value 为止（或者查到最高level，查找失败，说明整个系统中不存在这个Key)。

如果给定一个要查询的 key 和某个 key range 包含这个key的 SSTable 文件，那么 LevelDB 是如何进行具体查找过程的呢？LevelDB 一般会先在内存中的Cache中查找是否包含这个文件的缓存记录，如果包含，则从缓存中读取；如果不包含，则打开SSTable文件，同时将这个文件的索引部分加载到内存中并放入Cache中。 这样Cache里面就有了这个SSTable的缓存项，但是只有索引部分在内存中，之后 LevelDB 根据索引可以定位到哪个内容Block会包含这条key，从文件中读出这个Block的内容，在根据记录一一比较，如果找到则返回结果，如果没有找到，那么说明这个level的 SSTable 文件并不包含这个key，所以到下一级别的 SSTable 中去查找。

## 8\. Compaction

LevelDB 包含其中两种 compaction 模式：minor 和 major。所谓 **minor Compaction**，就是把 memtable 中的数据导出到 SSTable 文件中；**major compaction** 就是合并不同层级的 SSTable 文件。

**minor Compaction**

Minor compaction 的目的是当内存中的 memtable 大小到了一定值时，将内容保存到磁盘文件中。其机理如下图所示：

![Minor compaction](http://cighao.com/img/articles/leveldb/leveldb-02-12.png)

当 memtable 数量到了一定程度会转换为 immutable memtable，此时不能往其中写入记录，只能从中读取KV内容。之前介绍过，immutable memtable 其实是一个多层级队列SkipList，其中的记录是根据 key 有序排列的。所以这个 minor compaction 实现起来也很简单，就是按照 immutable memtable 中记录由小到大遍历，并依次写入一个 level 0 的新建 SSTable 文件中，写完后建立文件的 index 数据，这样就完成了一次minor compaction。

从上图中也可以看出，对于被删除的记录，在 minor compaction 过程中并不真正删除这个记录，原因也很简单，这里只知道要删掉 key 记录，但是这个 KV 数据在哪里？那需要复杂的查找，所以在 minor compaction 的时候并不做删除，只是将这个 key 作为一个记录写入文件中，至于真正的删除操作，在以后更高层级的 compaction 中会去做。

**major compaction**

当某个 level 下的 SSTable 文件数目超过一定设置值后，levelDB 会从这个 level 的 SSTable 中选择一个文件（level>0），将其和高一层级的 level+1 的 SSTable 文件合并，这就是 major compaction。

在大于 0 的层级中，每个 SSTable 文件内的key都是由小到大有序存储的，而且不同文件之间的key范围（文件内最小key和最大key之间）不会有任何重叠。level 0 的 SSTable 文件有些特殊，尽管每个文件也是根据Key由小到大排列，但是因为 level 0 的文件是通过 minor compaction 直接生成的，所以任意两个 level 0下的两个 SSTable 文件可能再key范围上有重叠。所以在做 major compaction 的时候，对于大于 level 0 的层级，选择其中一个文件就行，但是对于 level 0 来说，指定某个文件后，本 level 中很可能有其他 SSTable 文件的 key 范围和这个文件有重叠，这种情况下，要找出所有有重叠的文件和 level 1 的文件进行合并，即 level 0 在进行文件选择的时候，可能会有多个文件参与 major compaction。

LevelDB 在选定某个 level 进行 compaction 后，还要选择是具体哪个文件要进行 compaction，LevelDB 在这里有个小技巧， 就是说轮流来，比如这次是文件A进行 compaction，那么下次就是在 key range 上紧挨着文件A的文件B进行 compaction，这样每个文件都会有机会轮流和高层的 level 文件进行合并。

如果选好了 level i 的文件A和 level i+1 层的文件进行合并，那么问题又来了，应该选择 level i+1 哪些文件进行合并？LevelDB 选择 i+1 层中和文件A在 key range 上有重叠的所有文件来和文件A进行合并。

也就是说，选定了 level i 的文件A，之后在 level i+1 中找到了所有需要合并的文件B,C,D… 等等。剩下的问题就是具体是如何进行 major 合并的？就是说给定了一系列文件，每个文件内部是key有序的，如何对这些文件进行合并，使得新生成的文件仍然Key有序，同时抛掉哪些不再有价值的 KV 数据。

下图所示的是合并过程：

![SSTable compaction](http://cighao.com/img/articles/leveldb/leveldb-02-13.png)

major compaction 的过程如下：对多个文件采用多路归并排序的方式，依次找出其中最小的key记录，也就是对多个文件中的所有记录重新进行排序。之后采取一定的标准判断这个key是否还需要保存，如果判断没有保存价值，那么直接抛掉，如果觉得还需要继续保存，那么就将其写入 level i+1 层中新生成的一个 SSTable 文件中。就这样对KV数据一一处理，形成了一系列新的 i+1 层数据文件，之前的 i 层文件和 i+1 层参与 compaction 的文件数据此时已经没有意义了，所以全部删除。这样就完成了 i 层和 i+1 层文件记录的合并过程。

那么在 major compaction 过程中，判断一个KV记录是否抛弃的标准是什么呢？其中一个标准是:对于某个key来说，如果在小于 i 层中存在这个key，那么这个KV在major compaction 过程中可以抛掉。因为，对于层级低于 i 的文件中如果存在同一 key 的记录，那么说明对于 key 来说，有更新鲜的 value 存在，那么过去的 value 就等于没有意义了，所以可以删除。

## 9\. Cache

读取操作如果没有在内存的 memtable 中找到记录，要多次进行磁盘访问操作。假设最优情况，即第一次就在 level 0 中最新的文件中找到了这个 key，那么也需要读取2次磁盘，一次是将 SSTable 的文件中的 index 部分读入内存，这样根据这个 index 可以确定 key 是在哪个 block 中存储；第二次是读入这个 block 的内容，然后在内存中查找 key 对应的 value。

LevelDB 中引入了两个不同的 Cache:Table Cache 和 Block Cache。其中 Block Cache 是配置可选的，即在配置文件中指定是否打开这个功能。

下图是 table cache 的结构：

![SSTable compaction](http://cighao.com/img/articles/leveldb/leveldb-02-14.png)

在 Cache 中，key值是 SSTable 的文件名称，value 部分包含两部分，一个是指向磁盘打开的 SSTable 文件的文件指针，这是为了方便读取内容；另外一个是指向内存中这个 SSTable 文件对应的 Table 结构指针，table结构在内存中，保存了 SSTable 的 index 内容以及用来指示 block cache 用的 cache\_id ，当然除此外还有其它一些内容。

比如在 get(key) 读取操作中，如果 LevelDB 确定了key在某个level下某个文件A的key range范围内，那么需要判断是不是文件A真的包含这个KV。此时，LevelDB 会首先查找 Table Cache，看这个文件是否在缓存里，如果找到了，那么根据 index 部分就可以查找是哪个 block 包含这个 key。如果没有在缓存中找到文件，那么打开 SSTable 文件，将其 index 部分读入内存，然后插入 Cache 里面，去 index 里面定位哪个 block 包含这个 key 。如果确定了文件哪个 block 包含这个 key，那么需要读入 block 内容，这是第二次读取。

![SSTable compaction](http://cighao.com/img/articles/leveldb/leveldb-02-15.png)

Block Cache是为了加快这个过程的。上图是block cache 的结构。其中的key是文件的cache\_id加上这个block在文件中的起始位置block\_offset。而value则是这个Block的内容。

如果 LevelDB发现这个 block 在 block cache 中，那么可以避免读取数据，直接在 cache 里的 block 内容里面查找key的value就行，如果没找到呢？那么读入 block 内容并把它插入 block cache 中。LevelDB 就是这样通过两个 cache 来加快读取速度的。从这里可以看出，如果读取的数据局部性比较好，也就是说要读的数据大部分在cache里面都能读到，那么读取效率应该还是很高的，而如果是对key进行顺序读取效率也应该不错，因为一次读入后可以多次被复用。但是如果是随机读取，您可以推断下其效率如何。

## 10\. Version

**Version** 保存了当前磁盘以及内存中所有的文件信息，一般只有一个 Version 叫做 “current” version（当前版本）。LevelDB还保存了一系列的历史版本，这些历史版本有什么作用呢？

当一个 Iterator 创建后，Iterator 就引用到了 current version(当前版本)，只要这个 Iterator 不被 delete 那么被 Iterator 引用的版本就会一直存活。这就意味着当你用完一个 Iterator 后，需要及时删除它。

当一次 Compaction 结束后（会生成新的文件，合并前的文件需要删除），LevelDB 会创建一个新的版本作为当前版本，原先的当前版本就会变为历史版本。

**VersionSet** 是所有 Version的集合，管理着所有存活的 Version。

**VersionEdit** 表示 Version 之间的变化，相当于 delta 增量，表示有增加了多少文件，删除了文件。他们之间的关系如下：

```
Version0 +VersionEdit-->Version1
```

VersionEdit 会保存到 manifest 文件中，当做数据恢复时就会从 manifest 文件中读出来重建数据。

这其实是一种MVCC思想的实现。