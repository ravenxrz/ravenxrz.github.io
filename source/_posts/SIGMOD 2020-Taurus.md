---
title: SIGMOD(2020)-Taurus
categories:  论文
abbrlink:
date: 2025-02-25 22:10:31
tags:
---

本文采用wolai制作： 原文link: [https://www.wolai.com/ravenxrz/eVCDgKnqMXKwptcKfKnVPo](https://www.wolai.com/ravenxrz/eVCDgKnqMXKwptcKfKnVPo "https://www.wolai.com/ravenxrz/eVCDgKnqMXKwptcKfKnVPo")

原文名: Taurus Database: How to be Fast, Available, and Frugal  in the Cloud

TODO:

- [ ] aurora和polardb的复制方式是什么

重点：

存算分离，和Aurora和Socrates 类似，采用**新的复制和恢复算法**，提供更高可用性, 存储分为logstore和pagestore，logstore采用强一致性写入，pagestore采用最终一致性写入。 架构上：关键路径网络小于等于一跳网络，采用append only存储。

<!--more-->

# 相关工作

**PolarDB** 基于共享分布式文件系统 PolarFS 构建，虽然解决了传统数据库在云端部署的部分问题，但仍存在一些缺点：

1. **写放大问题**：由于存储层 PolarFS 不感知数据库的特点，在数据写入时无法针对数据库的特性进行优化，导致写操作产生额外开销，出现写放大现象，增加了存储资源的消耗 。
2. **高网络负载**：页面刷新时会产生较高的网络负载。数据库运行过程中，修改后的页面需要刷新到存储层，由于 PolarFS 的架构特性，这一过程会导致大量数据在网络中传输，加重网络负担。
3. **性能和可扩展性受限**：主节点承担了所有的处理工作，这种架构模式限制了系统的性能和可扩展性。随着业务量的增长，主节点的处理压力会不断增大，容易成为性能瓶颈，影响整个系统的运行效率，并且在扩展节点以提升性能方面也存在困难。



Taurus 在架构上，分为了两个物理层（logstore和pagestore），logstore和pagestore可以采用不用的复制、一致性技术，降低成本。相比Socrates（4层架构）， 层数更低，网络延迟也更低。page存储在本地，读取延迟也更低（笔者注：但是成本高了）。

复制协议上采用了gossip和中心复制的算法。

# 系统架构

## 传统架构

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_ibgK5yvlav.png)

每个副本都有完成的数据copy（成本浪费，空间浪费，cpu，mem都在浪费， 扩展性也很差，笔者注：但是很稳定）， master节点可以处理读写请求，replica副本可以处理读请求。主备通过binlog同步。



## taurus架构

taurus由四部分组成，logstore， pagestore，Storage Abstraction Layer(SAL)和数据库前端（笔者注：就是计算层mysql）。整体上，分为两个物理层（为了降低网络延迟）。

笔者注： 怎么看出来只有两层的？这里看起来有三层，只是说一次write，或者一次read只跨一次网络。

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_qGnn9NEBEy.png)

- logstore， 负责持久化log record，一旦一个事务的所有log record持久化完成，就可以ack到client。有两个作用：
  1. 负责log持久化
  2. replica可以从logstore中获取（笔者注：replica pull log的模型）log record，然后replica就可以apply log record 了。master节点周期同步最新log位置，replica拿着这个位置信息去logstore pull。master还负责将log分发到pagestore中。（上图中红色的线）
     > 笔者注：这样做，master的负载是否有点高了，另外为什么不让logstore主动push，replica和pagestore可以向logstore注册+心跳，后面logstore主动push即可，或许觉得在logstore维护这个信息太昂贵了，还有push的周期不定，开销更大，所以replica pull log更合理。
- pagestore，taurus数据库被分为多个10GB的分片（called slice), 每个pagestore上处理多个slice，每个slice有3个副本。



## logstore

logstore提供的关键抽象称为Plog, 目前复制副本数设置为3.

在写log上，会从logstore集群中找到3个server来执行写入，并且采用强一致性写入，即只有当三个副本都写入成功时，才算成功，如果有一个server写入不成功，则从重新选择3个server来写入（笔者注：为啥不重新选失败的个数？要全部重新选）。

由于采用强一致性写，所以读取只用从任意一个副本读取即可。读取log的地方有两处：


1. replica读取log用于apply
2. 故障恢复，主节点（笔者注：猜测是主节点，或者某个恢复模块）读取log，然后发送给pagestore

logstore在内存中还用FIFO的淘汰策略存储最近写入的log，用于加速后续的读取请求。

PLog分为data Plog和meta Plog, meta Plog记录了有哪些data Plog。**所有数据库节点中都cache 了meta Plog**，meta  Plog的update是用原子的方式更新的，当meta Plog大小达到阈值时，做切换meta的操作。



## pagestore

pagestore提供任意版本（未回收时）page的读取（主要给备机读，还有故障恢复时读取）。每个Page版本由page id和lsn唯一标识，当cleint想与pgestore交互，SAL提供4个接口：

1. WriteLogs， 用于写入log
2. ReadPage， 用于读取指定版本的Page。&#x20;
3. SetRecycleLSN，同一个数据库实例最老可能访问的lsn。
4. GetPersistentLSN，pagestore可提供的最高的lsn  (笔者注，这里说的最高持久化lsn是连续的最高lsn？还是持久化到的最高lsn？ 答案：根据后续所述，这里应该是指连续最高的LSN）

> 笔者注： pagestore的写入 log采用什么一致性策略？强一致？弱一致？如果是弱一致，怎么保证一致性？网络中断、超时、重试时，pagestore是怎么解决的？
> 后文提到： pagestore采用最终一致性模型（单副本写入成功即成功，如果单副本挂了，则从logstore中重新构建）, 并通过gossip协议来补齐数据

所有接口都需要传入slice id。如当SAL想要ReadPage时，需要传入slice , page id以及page的version（也就是LSN)。 由于存储所有版本需要资源（比如盘空间），所以SQL层通过SetRecycleLSN来指定它可能访问的最老的LSN（这个LSN是SQL层主备机协商得到的），然后pagestore就可以回收了（笔者注：这依赖了SQL层，如果SQL层出问题，pagestore就会有严重的空间浪费，在笔者运维经验中，这个问题经常出，如果可以，尽量不要依赖SQL层，因为至少master node是最新的，replica如果读不到可以考虑重启，但这又会带来性能抖动）。最后，pagestore采用了最终一致性协议，通过GetPersistentLSN接口可以获取每个pagestore server上当前最大的lsn信息。



## 存储抽象层SAL

SAL是一个sdk lib，切入sql层，所有和存储层（logstore和pagestore）的交互都会经过sal，如writelog，readpage接口。slice的创建删除，page的映射都由SAL完成。 在WriteLog时，SQL会做一次聚合再下发给logstore的，当logstore强一致写入成功后，ack给client，之后将日志拆分到不同的slice buffer中，当slice buffer满或者超时后就flush到pagestore中。

此外，SAL还维护了一个cluster visible(CV) LSN。 它是一个数据库一致性点（笔者注：类似于MYSQL MTR边界），代表数据库内部操作的一个一致性点，如B+tree分裂、合并可能涉及到多个page修改，但是这个LSN一定代表的是一次原子操作的边界。

redolog会被拆到多个slice，每个slice有多个副本，当redo log对应的多个slice都被持久化（但不要求slice的所有副本都完成持久化，只用保证至少一个），CV-LSN即可推进，CV-LSN的推进是按照database log buffer的粒度为推进的，也即当一个database log buffer flush完成后(其实还有第二个条件，见下文），CV-LSN等于这个log buffer最后最大的哪个LSN。具体而言，CV-LSN推进需要满足两个条件：

1. database log buffer在logstore中持久化完成
2. 这个log buffer对应的所有slice都全部持久化（只要求每个slice至少有一个副本持久化完成）

> 笔者注：
> log buffer被拆成多个slice buffer, 一个slice buffer可能来自多个log buffer，是一种多对多的映射关系。



## 数据库前端（MYSQL）

Currently, Taurus uses a slightly modified version of MySQL 8.0 as a database front end. Modifications include forwarding log writes and page reads to the SAL layer and disabling buffer pool flushing and redo recovery for the master node. Modifications to read replicas include updating pages in the buffer pool using records read by SAL from Log Stores and a mechanism for assigning read views to transactions



# 复制

## 写入路径

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_PIfKZ38l5i.png)

1. 用户事务导致page变化，生成redo
2. redo写入logstore（3副本），设置Plog的size为64M（拆开粒度，充分利用分布式能力）
3. 一旦logstore写入完成，就可以ack给用户
4. master SAL会将内存中的log buffer拆分到对应的slice buffer，当slice buffer满，或者超时时，将slice buffer刷新到pagestore，并等待其中一个节点ack
5. 当一个pagestore ack后，master SAL就可以释放这个log buffer内存了。
6. slice buffer只用在pagestore的一个节点完成持久化就返回了，所以一个slice的不同replica相互之间数据不是一致的，这通过gossip协议来补齐各自缺失的数据
7. 周期获取一个pagestore replica当前的persistent lsn，用于回收logstore的log
8. 根据第7步获取的lsn，以及Plog的lsn信息来回收Plog

> 笔者注： pagestore只用单个replica写入成功即可ack，如果这个replica挂了，那么client就拿不到这个page（只能希望buffer pool里面有了），个人采选哪个后续recovery需要从logstore拉取，这里的时间就比较长了，但是对比微软的socrate架构，这个速度似乎也还行？

pagestore只有单个replica持久化就ack的优势：

1. 写入成功的概率要高得多，因为它只需要三个节点中的一个节点可用即可。这确保了高可用性，且数据持久性不会受到影响，因为日志记录已经在日志存储节点上持久化了。
2. 写入延迟被最小化，因为它只取决于回复速度最快的节点，而不是最慢的节点。(笔者注：长尾更高）
3. 存储抽象层（SAL）不再负责确保每条日志记录都能到达所有相应的页面存储。因此，存储抽象层（SAL）将日志记录保留在内存中的时间更短，重新发送日志记录时消耗的 CPU 和网络带宽也更少。根据需要re-send log的任务被转移到了pagestore自身。



## 读路径

mysql改造了buffer pool的剔除算法，保证buffer pool的dirty page对应的log一定已经被持久化到了至少一个pagestore，才能剔除。

对于每个slice， SAL维护了这个slice当前最新写入到的log LSN，主机的读取总是最新的，主机读就会带上这个lsn，SAL会路由当最佳的pagestore（具有最低的时延）尝试读取，如果读取失败，再换一个节点读。

> 笔者注： 失败了不如直接向其他所有replica同时发起读，不然长尾太高了。



## Log回收

log的回收条件为：

1. 所有的pagestore slice replica都已经持久化了这段log
2. 所有的database read replica都已经看到该段log

具体的回收过程如图3的第7-第8步。

第7步通过GetPersistentLSN获取一个pagestore slice replica当前最大的持久化点，同时统计一段Plog的最大LSN。在8步中，如果Plog的最大LSN小于等于了所有replica的最小Persistent LSN，则该段Plog可以回收。

> 笔者注： 这里并没有提到所有database read replica是否看到该段log的track过程



## 对比quorum协议

本章略。作者只对比了可用性，没有对比性能。对比结果如下图:

x为单机故障概率：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_o1uT9i0CC2.png)





# Recovery

## Logstore Recovery

logstore故障恢复很简单，如果一个logstore节点故障了，则上面对应的所有Plog将被禁写，后续的写入重新从其他节点创建新的Plog接口，缺失的这部分后续通过复制重新修复回来即可。



## pagestore recovery

一个pagestore node临时故障重启后，会通过gossip向其他节点补齐自身缺失的数据，如下图：



![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_qNnkQc_czd.png)

replica 3由于临时故障，丢失了LSN=2的数据，启动后通过gossip从replica 2中获取该段数据。

对于长期故障，该节点会从集群中提出，同时在另一个节点上重新分配该slice， 一旦分配完成，该slice就可以立即接受WriteLogs请求（笔者注：这样真的好吗？pagestore本身只要求一个节点写入就ack，现在这个slice缺了大量其他数据，如果只向这个一个节点成功就返回，后续来读怎么办，没版本立即补齐其余数据）。之后该副本会向其他replica拷贝所有页面的最新版本（笔者注：新问题，只拷贝了最新版本，如果所有replica全挂了一遍，老版本就丢了，上面的sql read replica就会挂，当然这个概率很低）。



还有一种情况，如下：

构造某个lsn的数据在全副本丢失，且persistent lsn回退的场景：

如果最开始replica 2和replica 3都挂了，只有replica 1收到了LSN=2的请求，接着replica 2和3

启动，但是replica1挂了，此时replica 2和3持久LSN=3的数据。replica1是永久故障，会从其他节点上重新新创建起来，此时LSN=2的数据在pagestore这一层就永久丢失了，SAL会检测这种情况并处理（SAL会记录每个pagestore最新持久化的点，如果后续周期通信中，发现某个pagestore的node的LSN回退了，如下图的replica 1最新连续LSN等于1，小于之前记录的2，就会主动从logstore中重新下发，&#x20;

> 笔者注：那这些各个节点LSN信息需要持久化吗？不持久化，如果master SAL也挂了，怎么办？如果要持久化，那么持久化的策略是什么，立即持久化不现实，不立即持久化又没有意义。答案： 后文提到，有兜底策略，周期去扫描每个每个slice 的persistent lsn，和slice的flush lsn对比，如果该persistent lsn小于flush，sal会获取每个slice副本没有收到的日志范围，然后从logstore中获取这些log，并re-send对应的副本上。。

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_aK-BDz5HAD.png)

> slice flush lsn表示一个slice曾经被持久化到的最大lsn
> persisten lsn表示一个pagestore node上持久化的但是连续的最大lsn



最后还有一种情况，如下：

构造了一种LSN=3在全副本丢失，且persistent lsn没有回退的场景：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_MGRFpmU5dO.png)

对于这种情况SAL如果发现节点的persistent lsn长期不推进，且小于flush lsn，就会去查询每个slice replica缺失了哪些log，然后从logstore中找到这些log并re-send到这些replica上。

> 笔者注：这里的所有兜底操作都是从logstore中找到丢失log然后重新发送到pagestore，但是一个比较大的缺点是，logstore存放的是实例级别的log，但是pagestore存放的是slice级别的log，从实例级别找到slice级别的部分log，这部分开销可能是较大的，这会很影响时延。

实际上，sal发现某个node没有推进persistent lsn时，会首先尝试触发该节点gossip，如果gossip都无法补齐数据，再从logstore中re-send。



## SAL和mysql

存储抽象层（SAL）会读取数据库持久化日志序列号（database persistent LSN）的最后保存值，并将其作为开始读取日志的起始点。只有那些在所有切片副本中都缺失的日志记录，才会被再次发送到相应的切片中。

在存储抽象层（SAL）恢复完成后，数据库就可以接受新的请求了。在接受新请求的同时，数据库前端会执行undo阶段的操作，即回滚在系统崩溃时尚未提交的事务所做出的更改。在接受新事务之前，必须先完成重做恢复，因为要把数据库恢复到一致性的状态才能再进行修改。



# Read Replica读副本

备机是落后于主机状态的，下面说下备机是如何更新状态的，如下图：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_v2xt7DGIPA.png)

1. 主机将日志写入到log和page store
2. 备机询问（笔者注：还是主机推送？备机询问的话，周期是多少？或者由什么事件触发？）主机log的最新位置和数据库最新lsn
3. 备机从logstore中拉取需要的log信息
4. 如果需要（buffer pool不命中时）从pagestore总读取需要的page

主备之间的交互只有元数据（描述log位置信息，lsn信息，slice信息等）



# PageStore Design

PageStore的主要职责是将收到的log回放成page，并提供给上层读取。

一个回放过程需要 base page + 后续的log，如何快速的找到这些数据成为关键。在taurus中，对每个slice，设计了一个数据结构，称为log directory（由无锁hash实现）, 它保存了所有log和page版本的位置信息。

下图解释了pagestore内部的workflow:

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_KZxOeR0b8N.png)

1. 日志是batch的形式写入
2. pagestore收到log后，会立即执行持久化，并同时在内存中缓存,每条日志记录的位置信息会被追加到log directory中
3. applylog record来生成新page
4. 新page将会添加到buffer pool中，用于后续access
5. buffer pool的数据在后台周期的回写到盘上（可以做聚合，减少io次数）



一个问题：该优先合并哪些page？

taurus最开始选择优先合并长链的page，也就是说优先合并hot page，但是这会导致系统存在大量的cold page，他们只有少量record，但是占用很多的log directory entry，log direcotry采用FIFO淘汰，之后它们被淘汰后，如果有读请求，又需要从盘上将它们bring back to memory,导致合并时间更高。

后续优化成优先合并在内存缓存中的page，如果缓存满了，新的log就直接写入磁盘，然后加入加载队列中，等待后续合并。



# 实验对比

和**aurora**对比，作者认为性能提升主要来自taurus的replication策略。

运行了 SysBench 的read only和write only工作负载，以及带有不同大小数据库的 Percona TPC-C 变体 。图 7 展示了测试结果。
在全部五项基准测试中，Taurus 的表现都超过了 Aurora 团队所公布的结果。在只读测试中，差异较小（16%），但在只写基准测试中，Taurus 的优势超过了 50%，而在 TPC-C 测试中，这一优势达到了 160% 。

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_w2kLlSI6B-.png)

> 笔者注：但是没对比成本，毕竟taurus多了一层logstore



和**socrate**对比，taurus没法直接和socrate直接相比，但是socrate和sql server对比了，所以taurus也可以直接和本地mysql对比。如下图：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage_5kmIQ92O5q.png)

Socrates 所实现的性能略逊于（差 5%）SQL Server。相比之下，Taurus 展现出相对于使用本地存储的原生 MySQL 的性能提升，对于 SysBench 只读工作负载，提升幅度为 50%（第三根实心柱），而对于 SysBench 只写工作负载和 TPC-C 测试，提升幅度达到 200%（第四根和第五根实心柱）。

作者认为这是因为taurus只有两层架构，socrate则是有四层网络。所以有明显差异。

