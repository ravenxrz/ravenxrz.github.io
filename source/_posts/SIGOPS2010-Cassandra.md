---
title: SIGOPS2010-Cassandra
categories:  论文
abbrlink:
date: 2025-04-06 12:00:31
tags:
---

# Cassandra


> 本文采用wolai制作，原文链接： [https://www.wolai.com/ravenxrz/hSCxJQCsFwGoeGMQz9dAMu](https://www.wolai.com/ravenxrz/hSCxJQCsFwGoeGMQz9dAMu "https://www.wolai.com/ravenxrz/hSCxJQCsFwGoeGMQz9dAMu")

原文名：Cassandra - A Decentralized Structured Storage System

Cassandra论文主要集中讨论了:partitioning, replication, membership, failure handling and scaling

<!--more-->


## 1 Partitioning

Cassandra使用一致性hash来做分区，一致性hash有良好的可扩展性，

优点包括:

1. 数据打散均匀
2. 动态可扩展，添加或者单节点宕机都只影响其后继节点，只需要少部分数据迁移即可

缺点：

1. 不感知集群内的析构性，即有些节点可能能力强，有些节点能力弱，basic的一致性hash都平等对待
2. 数据倾斜问题，通常数据具有热点效应，一致性hash容易带来倾斜问题。

对于“缺点”，dynamo的做法是采用vnode，而Cassandra除了vnode还采用的动态迁移的做法，识别负载的load情况，然后将light的node迁移做hash环迁移，以此来缓解heavy load的node压力。

> 笔者注：具体怎么做迁移? 看后续论文似乎要管理员介入？



## 2 Replication

Replication满足高可用和持久性。

Cassandra中的数据都有N个副本（N为client配置，per isntance的参数），这N的副本的存储方式也提供了多种选择:

1. Rack unaware,  如果是这种策略，则除了hash环的当前coordinator外，将后续的N-1副本放在hash环的N-1个后继节点中
2. Rack aware, datacenter aware, 这种方式更复杂，论文没有说明如何实现

Cassandra使用zookeeper在集群中选主，该leader负责集群告诉各节点（包括新加入的）负责的数据范围，各节点本地也有cache，当节点crash后，也可以和leader联系找回负责的数据范围，集群中的各节点所负责的数据范围不超过N-1个（笔者注：是怕所有数据范围都落在这个节点上然后导致durability不满足吗？）



## 3 Membership

Cassandra的成员基于 **Scuttlebutt**, 一种高效anti-entropy gossip机制。

1.故障检测

采用改造过的**Accrual Failure Detector** 实现故障检测，这种故障检测机制不是简单的使用一个Boolean值来表示一个节点是up or down，而是用了一个 $\Theta
 $值来表示一个节点故障的怀疑程度，好处是可以通过阈值调节来反应网络情况和负载情况，避免被误判定。

> 笔者注： 笔者最近确实也遇到了因为网络和负载情况的原因，某些节点被误判定为故障节点，但是该场景过于特殊，线上是绝对不允许出现的。针对这种场景单独做优化，是否值得？

设当Φ=1时我们判定怀疑节点 A 出现故障，那么做出错误判断的可能性（即该判断在未来因收到延迟的心跳消息而被推翻的概率）约为 10%。当Φ=2时，这个可能性约为 1%；当Φ=3时，可能性约为 0.1%，依此类推。系统中的**每个节点都会维护一个滑动窗口，用于记录来自集群中其他节点的gossip消息到达时间间隔**。通过**确定这些到达时间间隔的分布来计算Φ值**。尽管原始论文(**Scuttlebutt**)提出该分布可以用高斯分布近似，但由于gossip的特性及其对延迟的影响，Cassandra发现指数分布是更好的近似方式。

Φ 采用指数分布



## 4 Bootstrapping

当节点首次启动时，会随机选择一个token，并将这个token在本地持久化，同时也存储在zk中，之后通过gossip将token信息传播到集群，这样其他节点就能知道该节点的位置信息，后续的request可以重定位到该节点。

当一个节点要加入集群时，会读取集群的配置信息，配置信息包含几个可contact节点，也被称为seed节点，这些种子节点也可以从zk中获取。

对于临时故障，不用启用修复（毕竟修复是很好cpu和io的）。为了避免人为误操作，比如一个本应属于A集群的节点加入到B节点了，Cassandra中的每条消息都包含集群名，当集群识别到集群名不匹配时，会拒绝服务。

> 笔者注： 避免人为误操作这个挺重要的，个人经历过集群销毁重建，但是业务没断，后续集群又重建，业务重新写下来将数据写坏的事件。Cassandra为每条消息都加入了集群名，个人工作项目中，每条消息也加入了集群的uuid。



## 5 Scaling the Cluster&#x20;

当一个新节点加入的集群时，它会被分配一个位置token（笔者注：第4节又说是随机选择？？），用于分担heavy node的负载，节点的bootstrapping算法是管理员初始化的（笔者注：也就是说负载分担是又管理员人为识别的？？），在此之后，放弃数据的节点通过kernel-kernel copy)的方式，将数据transfer到新节点，速率大概在40MB/s single node, 作者说后续会优化这一点，方式是通过多个node同时transfer，优点类似于Bittorrent。



## 6 Local Persistence

一个典型的write过程：先将write request写到commit log，当commit log持久化完成后，将write写入到mem。为了优化这个过程，作者采用了一个专用的disk来专门写commit log, 用于最大化吞吐（笔者注：还可以降低写放大，笔者工作中需要尽可能“攒”写WAL的buffer，避免写放大过大，同时尽可能4k对齐，避免读改写）。当内存中的数据大小或者数据个数超过一定阈值，将in-memory 节点dump到disk，这个写入由一个或者多个商用disk承担，同时还会生成index，index同data一起写入，后续还有gc。 整个过程和BigTable类似。

后续的读处理过程和 rocksdb类似，这里不赘述。



## 7 实现细节

消息传输协议选择：

1. 管控命令使用UDP
2. app message和replication协议采用TCP

commit log 回滚size为128MB，没128MB做一次commit log回滚，回滚时，检测comimt log中哪些数据已经持久化，持久化过的即可回收。



## 8 实验

这里记录一些实验LESSONS:

1. Cassandra不支持ACID事务，比如关系型数据中批量原子更新
2. 故障检测更快，对于100个节点的集群，在将$\phi
    $设置为5时，故障检测在15s内可检测，而其他算法需要2min。
3. 虽然Cassandra是完全去中心化的，但是也和zookeper这类分布式协调服务集成（如集群内node的data range管理）


