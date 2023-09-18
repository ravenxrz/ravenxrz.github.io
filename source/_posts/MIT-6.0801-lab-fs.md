---
title: MIT6.S081-lab-fs
categories: MIT-6.S081
abbrlink: 64032a5e
date: 2022-06-05 22:10:31
tags:
---

## 0. 前言

本实验为MIT6.s8081的第九个实验，主题和 **文件系统** 相关。实验任务有两个：

1. 增大xv6的最大文件size限制
2. 实现软链接

整体来看，实验相对简单。 

<!--more-->

## 1. 实现

### 1. Large files

本任务要求增大xv6的最大文件size限制。

原文要求：

> In this assignment you'll increase the maximum size of an xv6 file. Currently xv6 files are limited to 268 blocks, or 268*BSIZE bytes (BSIZE is 1024 in xv6). This limit comes from the fact that an xv6 inode contains 12 "direct" block numbers and one "singly-indirect" block number, which refers to a block that holds up to 256 more block numbers, for a total of 12+256=268 blocks.
>
> The `bigfile` command creates the longest file it can, and reports that size:
>
> ```
> $ bigfile
> ..
> wrote 268 blocks
> bigfile: file is too small
> $
> ```
>
> The test fails because `bigfile` expects to be able to create a file with 65803 blocks, but unmodified xv6 limits files to 268 blocks.
>
> You'll change the xv6 file system code to support a "doubly-indirect" block in each inode, containing 256 addresses of singly-indirect blocks, each of which can contain up to 256 addresses of data blocks. The result will be that a file will be able to consist of up to 65803 blocks, or 256*256+256+11 blocks (11 instead of 12, because we will sacrifice one of the direct block numbers for the double-indirect block).
>
> ### Preliminaries
>
> The `mkfs` program creates the xv6 file system disk image and determines how many total blocks the file system has; this size is controlled by `FSSIZE` in `kernel/param.h`. You'll see that `FSSIZE` in the repository for this lab is set to 200,000 blocks. You should see the following output from `mkfs/mkfs` in the make output:
>
> ```
> nmeta 70 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 25) blocks 199930 total 200000
> ```
>
> This line describes the file system that `mkfs/mkfs` built: it has 70 meta-data blocks (blocks used to describe the file system) and 199,930 data blocks, totaling 200,000 blocks.
> If at any point during the lab you find yourself having to rebuild the file system from scratch, you can run `make clean` which forces make to rebuild fs.img.
>
> ### What to Look At
>
> The format of an on-disk inode is defined by `struct dinode` in `fs.h`. You're particularly interested in `NDIRECT`, `NINDIRECT`, `MAXFILE`, and the `addrs[]` element of `struct dinode`. Look at Figure 8.3 in the xv6 text for a diagram of the standard xv6 inode.
>
> The code that finds a file's data on disk is in `bmap()` in `fs.c`. Have a look at it and make sure you understand what it's doing. `bmap()` is called both when reading and writing a file. When writing, `bmap()` allocates new blocks as needed to hold file content, as well as allocating an indirect block if needed to hold block addresses.
>
> `bmap()` deals with two kinds of block numbers. The `bn` argument is a "logical block number" -- a block number within the file, relative to the start of the file. The block numbers in `ip->addrs[]`, and the argument to `bread()`, are disk block numbers. You can view `bmap()` as mapping a file's logical block numbers into disk block numbers.
>
> ### Your Job
>
> Modify `bmap()` so that it implements a doubly-indirect block, in addition to direct blocks and a singly-indirect block. You'll have to have only 11 direct blocks, rather than 12, to make room for your new doubly-indirect block; you're not allowed to change the size of an on-disk inode. The first 11 elements of `ip->addrs[]` should be direct blocks; the 12th should be a singly-indirect block (just like the current one); the 13th should be your new doubly-indirect block. You are done with this exercise when `bigfile` writes 65803 blocks and `usertests` runs successfully:
>
> ```
> $ bigfile
> ..................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................
> wrote 65803 blocks
> done; ok
> $ usertests
> ...
> ALL TESTS PASSED
> $ 
> ```
>
> `bigfile` will take at least a minute and a half to run.

#### 1. 原理

为了完成本实验，首先需要理解xv6文件系统是如何索引data block的。下图展示了xv6原版的inode:

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220605173637297.png" alt="image-20220605173637297" style="zoom:67%;" />



xv6中，有12个直接寻址地址和一个间接寻址地址。所以最多能支持的data block个数为: 12 + 256=268个。由于xv6中，每个data block的大小为1KB, 所以xv6最大文件大小为268K。

所以，**为了扩大单文件最大大小阈值，需要增加一个二级间接索引地址（类似二级页表）。**

#### 2. 实现

增加二级间接索引宏:

```c
#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
#define DNINDIRECT (NINDIRECT * NINDIRECT)
#define MAXFILE (NDIRECT + NINDIRECT + DNINDIRECT)
```

修改inode和dinode:

```c
  uint addrs[NDIRECT + 1 + 1]; // NDIRECT direct block, 1 indirect block, 1 doubly indirect blocks
```
接着修改bmap函数， bmap为data block的索引逻辑函数，完成file的逻辑块到物理块的映射。

```c
// current layout:
// NDIRECT data blocks
// 1 indirect block        -- 256 data block
// 1 doubly indirect block -- 256 * 256 data block
// total data blocks number = 1 + 256 + 256 * 256
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  } 

  bn -= NINDIRECT;
  if(bn < DNINDIRECT) {
    // Load doubly indirect block, allocating if necessary.
    if ((addr = ip->addrs[NDIRECT + 1]) == 0)
      ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint *)bp->data;
    if ((addr = a[bn / 256]) == 0) {
      a[bn / 256] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);

    // Load indirect block, allocating if necessary.
    bp = bread(ip->dev, addr);
    a = (uint *)bp->data;
    if ((addr = a[bn % 256]) == 0) {
      a[bn % 256] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

最后，完善data block的释放逻辑函数 itrunc:

```c
// Truncate inode (discard contents).
// Caller must hold ip->lock.
void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp;
  uint *a;

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(i = 0; i < NINDIRECT; i++){
      if(a[i])
        bfree(ip->dev, a[i]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  if(ip->addrs[NDIRECT + 1]) {
    bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    a = (uint *)bp->data;
    for(i = 0; i < NDIRECT; i++) {
      if(a[i]) {
        struct buf *tmp_bp = bread(ip->dev, a[i]);
        uint *tmp_a = (uint *)tmp_bp->data;
        for (j = 0; j < NDIRECT; j++) {
          if(tmp_a[j])  {
            bfree(ip->dev, tmp_a[j]);
            tmp_a[j] = 0;
          }
        }
        brelse(tmp_bp);
        bfree(ip->dev, a[i]);
        a[i] = 0;
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
    ip->addrs[NDIRECT + 1] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```

至此，task 1完成。 执行 `bigfile`进行测试。

### 2. Symbolic links

本任务要求实现软链接功能，原文要求如下：

> In this exercise you will add symbolic links to xv6. Symbolic links (or soft links) refer to a linked file by pathname; when a symbolic link is opened, the kernel follows the link to the referred file. Symbolic links resembles hard links, but hard links are restricted to pointing to file on the same disk, while symbolic links can cross disk devices. Although xv6 doesn't support multiple devices, implementing this system call is a good exercise to understand how pathname lookup works.
>
> ### Your job
>
> You will implement the `symlink(char *target, char *path)` system call, which creates a new symbolic link at path that refers to file named by target. For further information, see the man page symlink. To test, add symlinktest to the Makefile and run it. Your solution is complete when the tests produce the following output (including usertests succeeding).
>
> ```
> $ symlinktest
> Start: test symlinks
> test symlinks: ok
> Start: test concurrent symlinks
> test concurrent symlinks: ok
> $ usertests
> ...
> ALL TESTS PASSED
> $ 
> ```

#### 1. 原理

软链接本质上就是一个文件，文件内容存放的是被链接的文件的文件路径。打开软链接时，可根据该路径定位到被链接的文件。

#### 2. 实现

首先需要实现 symlink 系统调用，此部分算是轻车熟路了。略过。主要记录下如何实现该系统调用：

```c
uint64
sys_symlink(void)
{
 char target[MAXPATH];
 char path[MAXPATH];
 // get args
 if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0) {
   return -1;
 }

  // check path exist or not
  // create path and set its type to T_SYMLINK
  begin_op(); 
  struct inode *path_ip = create(path, T_SYMLINK, 0, 0);  // return a path_ip with ilock held
  if(path_ip == 0) {
    end_op();
    return -1;
  }
  
  // write target path into path's inode
  int n = strlen(target) + 1;
  int r = writei(path_ip, 0, (uint64)target, 0, n);
  if(r != n) {
    // NOTE: maybe rollbacking everything is better?
    panic("sys_symlink: writei error");
  }
  iunlockput(path_ip);
  end_op();

  return 0;
}
```

这部分很简单，创建一个软链接文件，并写入 `target`文件路径。值得注意的是，**这里无需判定 `target`对应文件存在**。

另外，修改 `sys_open`系统调用，增加对软链接文件的处理:

```c
  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    if((ip = namei(path)) == 0){
      end_op();
      return -1;
    }
    ilock(ip);
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
     if(ip->type == T_SYMLINK && omode != O_NOFOLLOW) {
      struct inode *tmp_ip;
      if ((tmp_ip = follow_symlink(ip, 0)) == 0) {
        end_op();
        return -1;
      }
      ip = tmp_ip; // the ilock of ip is held
    }
  }
```

如果是软链接，且mode表示需要FOLLOW该软链接，则调用`follow_symlink`函数：

```c
// follow the symlink recursively until finding a non-symlink file
// or the recursive_times is over 10, in which case return 0
// return ip referring to the target file with ilock held
// NOTE: ip->lock must be held
static struct inode*
follow_symlink(struct inode *ip, int recursive_times)
{
  if (recursive_times >= 10) {
    iunlockput(ip);
    return 0;
  }
  // check if the target file this symlink points to exists or not
  char target_path[MAXPATH];
  if (readi(ip, 0, (uint64)target_path, 0, MAXPATH) == 0)
    panic("open: readi");
  iunlockput(ip);
  if ((ip = namei(target_path)) == 0) {
    return 0;
  }
  ilock(ip);
  if(ip->type == T_SYMLINK) {
    return follow_symlink(ip, recursive_times + 1);
  }
  return ip;
}
```

`follow_symlink`为递归函数，如果follow的过程中，发现被链接的文件又是软链接文件，则继续follow，为了避免形成环形follow，增加了递归次数限制。如果递归超过10次，则取消寻找。（**实际上更好的做法，可以采用一个hash table来记录已经遍历过的inode，如果遇到了遍历过的inode，则说明出现了环，进而退出)**.

另外一个需要注意的是，这里的加解锁。 `follow_symlink` 要求传入的 `ip`是持有lock的，而返回的找到的`inode_pointer`也是持有锁的。中间为了避免死锁问题，在往下一级别递归前，需要释放ip的lock。

至此，完成本实验。执行 `symlintest` 检测是否通过。

最后还需要执行 `usertests`判定是否影响了OS的其他功能。

## 2. 总结

本实验相对简单，对文件系统的组织有了一个更好的认识。