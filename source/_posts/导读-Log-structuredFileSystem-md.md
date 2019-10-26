---
title: 导读-Log-structuredFileSystem
date: 2019-09-23 13:58:48
categories: 论文
toc: true
tags:
thumbnail: /thumbnails/deer.jpg
---


i> 本文转载自：https://zhuanlan.zhihu.com/p/41358013

“[The Design and Implementation of a Log-Structured File System](https://link.zhihu.com/?target=https%3A//people.eecs.berkeley.edu/~brewer/cs262/LFS.pdf)“ 是 Mendel Rosenblum 和 John K. Ousterhout 在90年代初发表的一篇经典论文。且不提论文的两个作者都大名鼎鼎：Rosenblum 是 Vmware 的联合创始人，Ousterhout 是 [Raft](https://link.zhihu.com/?target=https%3A//raft.github.io/raft.pdf)的作者之一（Ongaro 的老板）; 这篇论文在发表之后就引起了长达数年的 [Fast File System](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Unix_File_System) 和 LFS 之间的[口水战](https://link.zhihu.com/?target=http%3A//www.eecs.harvard.edu/~margo/cs261/paper-history/second-submission.pdf)。LFS 在提出后的前10多年里并没有被业界采用（猜猜为什么），但当 SSD 的价格下降并成为主流后，LFS 却焕发了第二春：LFS 被广泛运用在 SSD 的 firmware 中，而且新的文件系统，譬如基于 journal 的 [ext3/ext4](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Ext4)和支持 copy on write 的 [btrfs](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Btrfs)都吸取了LFS 的 idea；甚至我们常用的[LSM](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Log-structured_merge-tree)算法都能看到 LFS 的影子。
<!-- more -->
## 动机

在90年代初，计算机的硬件性能开始了爆发式增长：CPU 的速度越来越快，RAM 也越来越大；然而，虽然硬盘的顺序读写速度也在提高，但是随机读写速度，受制于物理上的寻道时间，却难以短于10ms。另一方面，当时的文件系统不论是 [Unix File System](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Unix_File_System) 还是 FFS，都有大量的随机读写（在 FFS 中创建一个新文件至少需要5次随机写），因此成为整个系统的性能瓶颈。同时因为 Page cache 的存在，作者认为随机读不是 主要问题：随着越来越大的内存，大部分的读操作都能被 cache，因此 LFS 主要要解决的是减少对硬盘的随机写操作。

## File System as a Log

那么LFS是怎么减少随机写入呢？它基于一个很简单的 idea: 把整个磁盘看做一个 append only log,永远都是顺序写入。每当我们写入新文件时，我们总是顺序地追加在log 的最后；不同于 FFS/UFS, LFS 对文件的更改并不修改现有内容，而是把增量追加在硬盘的最后。

![img](https://pic2.zhimg.com/80/v2-5a968180f6bc6c9d97617c7ce46956e9_hd.jpg)



## 管理空余空间：段(Segements)

这样的设计有一个明显的问题：硬盘大小是有限的，当我们的 log 把硬盘写满以后，我们就不能再往硬盘里写入新的数据。但是正如图一所示，我们在操作文件系统时，我们会删除文件，或是用新的内容覆盖旧的内容；因此在 log 中会有过期数据。因此我们需要设计垃圾回收机制和空余空间管理机制。

在 LFS 中，空余空间是用固定大小的段(Segment)来管理的：硬盘被分割成固定大小的段；写操作首先会被写入到内存中；当内存中缓存的数据超过段的大小后，LFS 将数据一次性写入到空闲的段中。

![img](https://pic4.zhimg.com/80/v2-9c4ecd5a9b8c32469e5377c98e7c0c5f_hd.jpg)

## LFS 的读操作

基于段的批量写入解决了随机写入问题，但是LFS 如何实现读操作呢？类似 UFS/FFS, LFS 的在段内存储文件内容时，也存储了文件的索引。具体的来说：

- 在 Segment 中，文件内容存储在固定大小的 data block 中
- Segment 中同时存储了 data block 的索引, a.k.a inode. 每个 inode 存储了对应文件的 data block 的索引和 data block 的地址

在下图的例子中，Segment0 里存储了文件2的两个 data block。而之后的 inode2 中存储了这两个 data block 的索引。

然而不同于 UFS/FFS, LFS 的 inode 是动态分配的，因此 LFS 在每个 Segment 的尾部存储了对 inode 的索引， 称为 inode map。在 LFS 中，所有的 inode map 内容都会被缓存到内容中，从而加快读取速度。


  ![img](https://pic1.zhimg.com/80/v2-e2d18cf2debe20993e1cebc73ac43bd0_hd.jpg)

有了 inode/inode map 和 data block; 在 LFS 中读取一个inode 号为 i 的文件流程如下:

1. 从内存中缓存的 inode map 中找到所有的段中 inode 为 i 的地址
2. 根据不同段中的 inode 找到对应的 data block 地址
3. 读取 data block 中的数据

因为 LFS 是 append only，所以对同一个文件的同一个 data block 可能存在多个版本（在不同段中）。但是通过比较不同段的更新时间，LFS就能判断出哪个 segment 中的 data block 是最新版本。

## 垃圾回收

正如前文所说，LFS 需要设计垃圾回收机制来删除旧数据。 在 LFS 中，多个包含过期数据 block 的段（文件内容被更新或是文件被删除）会被 compact 成新的数据段，同时其中的旧数据会被删除。

![img](https://pic4.zhimg.com/80/v2-82171bfc38138a3f6637ab1b7ba4785f_hd.jpg)

但是 LFS 是如何检查段中的过期 block 呢？LFS 在每个段的头部存储了名为 Segment Summary 的数据结构。在 Segment Summary 中存储了段中每个 data block 的地址，和这个 data block 的 inode number （文件号），以及该 block 在文件中的 offset。对于任意 block，只要对比该 block 的地址，和通过 inode map 查询这个 block 的地址是否相同，就能判断这个 block 是否过期。

![img](https://pic4.zhimg.com/80/v2-a2973b1665f092f0e47ed95bd7ca5a3f_hd.jpg)

## 故障恢复

显然任何文件系统都要能从故障中恢复数据。不同于 UFS 用 [fsck](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Fsck) 命令对故障进行恢复，LFS 对整个硬盘存储了 Checkpoint:

- 因为 LFS 的每个 Segment 都存储了下一个 Segment 的地址，整个文件系统就像一个链表一样被组织在一起。
- 在 Checkpoint 中，LFS 存储了这个链表的第一个 Segment 和最后一个 Segment 的地址，因此只要读取 Checkpoint 就能恢复出整个文件系统。
- LFS 每 30秒更新一次 Checkpoint 中的数据。

![img](https://pic2.zhimg.com/80/v2-c08804c5f9a5c8dd876103c6bfede391_hd.jpg)

现在我们考虑一下系统崩溃的情况：

如果LFS 在创建 Checkpoint 时崩溃，比如只更新了 Checkpoint 的头指针而没有更新尾指针。对此 LFS 的解决方案是：

- LFS 在硬盘的头部和尾部存储了两个 Checkpoint，每次 Checkpoint 时 LFS 交替地存储在头部或是尾部的 Checkpoint 中。 这样即使写入一个 Checkpoint 失败， LFS 也能恢复到上一个 Checkpoint。
- 同时 LFS 利用时间戳来检测 Checkpoint 的失败：在写入 Checkpoint 时，先更新头指针和对应的时间戳，再更新 Checkpoint 中的其它内容，最后更新尾指针和相同的时间戳。如果 LFS 在读取 Checkpoint 时发现头指针和尾指针的时间戳不一致，就知道这个 Checkpoint 并没有完成。

如果 LFS 在创建 Checkpoint 之间失败，显然系统可以恢复到上一次 Checkpoint 时的状态。然而这会丢失一部分数据。对此 LFS 效仿了数据库的 redo log：LFS 会尝试从当前的 segment 链表尾部恢复出已经成功写入但没有被 Checkpoint 的数据段。

## 总结

从今天算起，LFS 已经发布了将近 30 年；然而正式由于作者对未来的正确假设，使得 LFS 的设计思想和理念却仍然深刻地影相了文件系统设计：LFS 的基于 Segment 的设计和 SSD 的物理特性不谋而合，因此被[广泛应用在 SSD 的 firmware 中](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/353411/)；LSM 的 memory table/compaction 与 LFS 的 memeory buffer 和 GC 一脉相承；而新的文件系统例如 btrfs 也基于 LSM append only 的特点实现了 copy-on-write 或是 multi-version 的特性。  参考文献

1. [The Design and Implementation of a Log-Structured File System](https://link.zhihu.com/?target=https%3A//people.eecs.berkeley.edu/~brewer/cs262/LFS.pdf)
2. [Log-structured file systems: There's one in every SSD](https://link.zhihu.com/?target=https%3A//lwn.net/Articles/353411/)
3. [CS 161: Lecture 15](https://link.zhihu.com/?target=http%3A//www.eecs.harvard.edu/~cs161/notes/lfs.pdf)
