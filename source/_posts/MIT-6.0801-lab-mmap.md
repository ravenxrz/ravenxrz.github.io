---
title: MIT6.S081-lab-mmap
categories: MIT-6.S081
abbrlink: 5300ed51
date: 2022-06-09 22:10:31
tags:
---

## 0. 前言

本实验为MIT6.s8081的第十个实验，主题为 **mmap** 相关。实验要求实现简化版本的 mmap/munmap 。

<!--more-->

## 1. 背景知识

mmap为 virtual memory 的扩展应用之一。通过将文件映射到user的virtual memory address中，使得user可以通过内存读写操作来进行文件读写。所以 mmap 涉及到 **内存**和 **文件系统**两部分。

要正确实现mmap，首先要知道mmap的预期行为有哪些。查看 linux 和 mmap 相关系统调用：

mmap相关函数有两个：

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);
```

**mmap执行映射:**

1. addr, 要映射到哪儿的hint（实际上并不一定映射到此处，对于本lab来说，addr始终为0，由内核选择要在user vm addr的哪个位置进行映射)
2. length, 映射的长度。 可小于文件的size， 等于文件的size，或者超过文件的size。如果超过了文件的size，超出的部分填充为0.
3. prot, 对映射的区域的protection属性，本lab仅需支持  PROT_READ  和  PROT_WRITE。前者代表可读，后者代表可写。
4. flags，映射的方式是什么。本lab仅需支持 MAP_SHARED和MAP_PRIVATE。 两者的区别如下：
   1. MAP_SHARED: Share this mapping.  Updates to the mapping are visible to other processes mapping the same region, and (in the case of file-backed mappings) are carried through to  the   underlying file.  即更新对其他process可见，更新会写会被映射的文件。
   2. MAP_PRIVATE： Create  a  private copy-on-write mapping.  Updates to the mapping are not visible to other processes mapping the same file, and are not carried through to the underlying file.  It is unspecified whether changes made to the file after the mmap() call are visible in the mapped region. 即更新对其他process是不可见的，更新不会写回被映射的文件，更新采用都会在自身独有的copy上执行。
5. fd， 要映射的file的fd。
6. offset， file的offset，本lab始终为0.

**munmap取消映射:**

1. addr, 要取消映射的首地址。 该地址不一定等于 mmap 返回的地址。所以取消的地址范围可能不是mmap中的地址范围。对于本lab而言，仅需支持三种情况的munmap， 如下图：

   ![munmap-lab-三种场景](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/munmap-lab-三种场景.svg)

   对于区间中间位置段，留下一个 "hole"的情况，不予考虑。

2. length: 需要取消映射的范围长度。

另外需要注意的是，执行mmap时，并未分配实际的物理页，mmap采用了**lazy allocation**的方案，映射只是更改了页表，后续通过page fault来添加需要映射的page。

## 2. 实现

首先添加 mmap 和 munmap 系统调用，下面仅说下签名：

```c
char *mmap(void *addr, int len, int prot, int flags, int fd, int off);
int munmap(void *addr, int len);

uint64 sys_mmap(void);
uint64 sys_munmap(void);
```

### 1. 基本数据结构

接着实现vma相关结构体，用于表示一个mmap的映射区：

```c
#define NVMA 16		/* how many vma area support */

struct vma_area {
	uint64 addr;		/* start address of mapping */
	uint64 len;
	int prot;
	int flags;	
	uint64 off;
	struct file *file;	
};
```

一个vma_aera代表一次映射。由于xv6内核不支持内核分配器（即内核的malloc), 模仿ftable，**实现一个vma_table**，表示全局可用的的vma结构：

```c
struct
{
  struct spinlock lock; /* protect vma_table  */
  struct vma_area vma_table[NVMA];
} kvma;
```

完成对vma的分配，释放，初始化函数：

```c
void
vmainit(void)
{
  initlock(&kvma.lock, "vma");
}

struct vma_area*
vmaalloc(void)
{
  struct vma_area* vma;

  acquire(&kvma.lock);
  for (vma = kvma.vma_table; vma < kvma.vma_table + NVMA; vma++) {
    if (vma->file == 0) {
      // zero vma
      memset(vma, 0, sizeof(struct vma_area));
      release(&kvma.lock);
      return vma;
    }
  }
  release(&kvma.lock);
  return 0;
}

void
vmafree(struct vma_area* vma)
{
  acquire(&kvma.lock);
  vma->file = 0;
  release(&kvma.lock);
}
```

现在xv6支持并发的分配和释放vma。 接下来来看看如何实现vma：

每个进程都有自己的vma映射，且可能有多组映射，因为一个进程mmap多次，所以更改proc结构，添加vma支持：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220609132523318.png" alt="image-20220609132523318" style="zoom:50%;" />

### 2. sys_mmap

以上添加了必须的数据结构，现在来完成mmap主体逻辑：

```c
// char *mmap(void *addr, int len, int prot, int flags, int fd, int off);
// the addr and offset always will be 0
uint64
sys_mmap(void)
{

  int len, prot, flags, fd, off;
  struct proc* cur_proc;
  struct file* file;
  struct vma_area *vma;

  if (argint(1, &len) < 0 || argint(2, &prot) < 0 || argint(3, &flags) < 0 || argint(4, &fd) < 0 || argint(5, &off) < 0)
    return -1;
  // check whether the file is opened or not
  cur_proc = myproc();
  if (fd < 0 || fd >= NOFILE || cur_proc->ofile[fd] == 0) {
    return -1;
  }
  // check whether the perm of prot is the subset of the perm of the file or not
  file = cur_proc->ofile[fd];
  if (prot & PROT_READ) {
    if (!file->readable) {
      return -1;
    }
  }
  if ((flags & MAP_SHARED) && (prot & PROT_WRITE)) {  // MAP_SHARED flag with PROT_WRITE indicates file must be writeable
    if (!file->writable) {
      return -1;
    }
  }

  // allocate a vma 
  if((vma = vmaalloc()) == 0) {
    panic("vmaalloc failed");
    return -1; 
  }
  vma->addr = 0;    // init addr: see code below 
  vma->len = len;
  vma->prot = prot;
  vma->flags = flags;
  vma->off = off;
  vma->file = file;
  filedup(vma->file); // increase file refcnt so that it's impossible to free the file

  acquire(&cur_proc->lock);
  // find the first continuous addr space in user page table so that this mapping can be mapped successfully
  uint64 sz = cur_proc->sz;
  // 向上页对齐, 如果不对齐，后续的page fualt handler使用的va可能低于addr
  uint64 start_addr = PGROUNDUP(sz);
  if(lazy_growproc(cur_proc, len + start_addr - sz) < 0) {
    release(&cur_proc->lock);
    fileclose(vma->file);
    vmafree(vma);
    return -1;
  }
  vma->addr = start_addr;

  // put the vam structure into proc structure
  int i;
  for(i = 0; i< NOFILE; i++) {
    if(cur_proc->vmas[i] == 0) {
      cur_proc->vmas[i] = vma;
      break;
    }
  }
  release(&cur_proc->lock);
  if(i == NOFILE) {
    panic("the vmas in proc is full");
    return -1;
  }
  return start_addr;
}
```

主要逻辑为：‘

1. 获取传入参数，因为本labaddr始终为0，所以从第一个参数开始获取。
2. 做一系列的合法性检查。比如传入的prot包含PROT_READ,则打开的文件也必须支持read。 唯一的注意点是，**如果file为readonly, 但是map的flag为MAP_PRIVATE，prot为 PROT_WRITE， 此时也可以写入的。**因为这样的映射组合，并不会修改到底层的文件，所有的修改都在进程自己的拷贝副本上进行。
3. 分配vma，并执行初始化。内核需要在进程空间中找到一个合适的连续地址空间，用于完成本次映射请求，**找到的地址空间首地址应该是 page-aligned 的**， 如果不是page-aglined的，后续的page_fault_handler中会出现错误。比如当前映射的首地址为2K， 现在要访问的地址也是2K， 对于第一次load page来说，会出现page fault， 在page_fault_hanlder中，懒加载的page需要按照page-aligned的方式加载page, 所以懒加载的页地址为 0K-4K-1。其中0K-2K-1这部分并没有做任何映射，所以会出现问题。
4. 最后在proc中找到合适的vma slot,存放vma结构即可。

第三步中，寻找合适的连续地址空间，采用的是 lazy_growproc的方式来完成：

```c
// Increase proc used addr space without allocating physical pages
// Return 0 on success, -1 on failure.
// NOTE: n must be larger than 0.
// NOTE: p.lock should be held
int lazy_growproc(struct proc *p, int n) 
{
  if(n <= 0) {
    return -1;
  }
  p->sz += n;
  return 0;
}
```



### 3. page_fault_handler

至此完成了sys_mmap, user app执行mmap后，可能会对map的page执行read/write操作，此时会触发page_fault, 下面是page_fault_handler代码：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220609133616808.png" alt="image-20220609133616808" style="zoom:50%;" />

```c
// mmap allocate physical memory lazily. 
// use this function to allocate physical memory and populate the corresponding file content.
static void
page_fault_handler(struct proc* p)
{
  uint64 va = r_stval();
  struct vma_area** vma_slot;
  struct vma_area* vma;
  struct file *file;
  if (va >= p->sz) {
    p->killed = 1;
    return;
  }

  // 检查va落入到哪个 vma area
  va = PGROUNDDOWN(va);
  if((vma_slot = vmalookup(p, va)) == 0) {
    p->killed = 1;
    return;
  }
  vma = *vma_slot;

  // alloc mem
  char* mem = kalloc();
  if (mem == 0) {
    // panic("page fault handler: out of memory");
    p->killed = 1;
    return;
  }
  memset(mem, 0, PGSIZE);

  // populate file content
  uint64 start_off = va - vma->addr + vma->off;  
  file = vma->file;

  ilock(file->ip);
  if(start_off >= file->ip->size) { // mapping size is larger than file size, fill memory with zero
    iunlock(file->ip);
  } else {
    begin_op();
    uint64 ret_bytes = file->ip->size < start_off + PGSIZE ? file->ip->size - start_off : PGSIZE;
    if (readi(file->ip, 0, (uint64)mem, start_off, ret_bytes) != ret_bytes) {
      iunlock(file->ip);
      end_op();
      kfree(mem);
      p->killed = 1;
      return;
    }
    iunlock(file->ip);
    end_op();
  }

  // add map pages
  if (mappages(p->pagetable, va, PGSIZE, (uint64)mem, PTE_W | PTE_R | PTE_U) != 0) {
    kfree(mem);
    p->killed = 1;
  }
}
```

这部分的工作为：

1. 执行简单的合法性校验: 访问va是否超过process的最大申请内存地址边界
2. 找到包含va的vma结构。
3. 分配实际的物理页
4. 注入对应的文件内容。这里的注意点是，有可能mmap传入的map length是大于file size的，对于超过file size部分，全部填充为0。
5. 最后，向页表中添加这个物理page的映射。

**注意，我个人的实现中，只要出现了page fault，就单独分配了物理页给进程，但实际上linux中的实现，如果flags为MAP_SHARED，则并不需要单独分配物理页，直接将映射绑定到page cache中即可，这样可以减少一次页的拷贝，提升性能。 如果flags为MAP_PRIVATE， 则采用COW机制。 更多关于Linux的mmap描述，可参考：[这里](https://zhuanlan.zhihu.com/p/67894878)**

上述代码使用到的 `vmalookup` 函数如下：

```c
// find the vma slot refering to a vma structure which contains 
// va
struct vma_area **
vmalookup(struct proc* p, uint64 va)
{
  acquire(&p->lock);
  for(int i = 0; i< NOFILE; i++) {
    if(p->vmas[i] != 0 && p->vmas[i]->file != 0) {
      if(va >= p->vmas[i]->addr && va < p->vmas[i]->addr + p->vmas[i]->len) {
        release(&p->lock);
        return &p->vmas[i];
      } 
    }
  }
  release(&p->lock);
  return 0;
}
```

代码逻辑很简单，循环遍历p的vma结构，找到包含va的vma。 之所以采用二级指针，是因为后续的 `sys_munmap` 中会使用到。

### 4.  sys_munmap

现在来看看取消映射的代码：

```c
uint64 
do_munmap(uint64 addr, int len)
{
  struct proc* cur_proc;
  struct vma_area** vma_slot;
  struct vma_area* vma;
  struct file* file;

  if(addr % PGSIZE) {
    panic("sys_munmap: addr should be PGSIZE-aligned");
  }
  // debug -- 目前仅支持addr和len都为 PGSIZE-aligned的请求
  if(len % PGSIZE) {
    panic("sys_munamp: len shoulbe be PGSIZE-aligned");
  }

  cur_proc = myproc();
  vma_slot = vmalookup(cur_proc, addr);
  if(vma_slot == 0) {
    return -1;
  }
  vma = *vma_slot;
  if (len > vma->len) {
    panic("sys_munmap: len is larger than the vma");
  }
  file = vma->file;

  // flush modified pages back to file
  // free mapged page
  if (vma->flags & MAP_SHARED) {
    ilock(file->ip);
    begin_op();
  }
  for (uint64 a = addr; a < addr + len; a += PGSIZE) {
    pte_t* pte = walk(cur_proc->pagetable, a, 0);
    if (vma->flags & MAP_SHARED) {
      if (*pte & PTE_D) { // if current page is dirty, wirte it back
        uint64 bytes = a + PGSIZE < vma->addr + vma->len ? PGSIZE : vma->len % PGSIZE;
        uint64 file_off = a - vma->addr + vma->off;
        if (writei(file->ip, 1, a, file_off, bytes) != bytes) {
          end_op();
          panic("munmap: write dirty page back failed");
        }
      }
    }
    uvmunmap(cur_proc->pagetable, a, 1, 1);
  }
  if (vma->flags & MAP_SHARED) {
    iunlock(file->ip);
    end_op();
  }

  // lab 简化了 munmap操作， 只支持映射区域全释放，头释放和尾释放
  // TODO(raven): 暂时没有考虑4k对齐的问题
  // update vma or free it
  if(vma->addr == addr && vma->len == len) { // the whole vma
    fileclose(file);
    vmafree(vma);
    *vma_slot = 0;
  } else if(vma->addr == addr && vma->len > len){ // the start region
    // update vma
    vma->addr = (addr + len);
    vma->len -= len;
  } else if(vma->addr != addr && addr + len == vma->addr + vma->len) { // the end region
    vma->len -=len;
  } else {
    panic("unsupported munmap operation: this will leading to a hole in the va");
  }
  return 0;
}

uint64
sys_munmap(void)
{
  uint64 addr;
  int len;
  if(argaddr(0, &addr) < 0 || argint(1, &len) < 0) {
    return -1;
  }
  return do_munmap(addr, len);
}
```

上述代码完成：

1. 合法性检验
2. 找到包含addr的vma
3. 对于 MAP_SHARED flag的map，需要将写入的数据，回写至文件，所以执行加解锁处理和文件写入。 由于仅需对 DIRTY_PAGE 进行写入，所以添加了 `*pte & PTE_D` 的判定。
4. 取消页映射
5. 更新vma结构：
   1. 如果释放的是整个vma所代表的的地址空间，则释放vma结构
   2. 如果释放的开头/结尾区间，则更新vma结构即可，无需释放

### 5. 其他

 **调整 uvmunmap**, 用于支持lazy allocation:

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220609135201008.png" alt="image-20220609135201008" style="zoom:50%;" />

**调整 fork**, 将父进程的vma继承给子进程：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220609135252478.png" alt="image-20220609135252478" style="zoom:50%;" />

**调整 allocproc**， 完成proc vma的初始化：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220609135450269.png" alt="image-20220609135450269" style="zoom:50%;" />

**调整 uvmcopy**, 以支持fork时的页表复制。

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220609135558496.png" alt="image-20220609135558496" style="zoom:50%;" />

**调整 exit,** 释放进程未主动释放的vma:

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220609135338868.png" alt="image-20220609135338868" style="zoom:50%;" />

**修改main，**完成对全局vma_table的初始化：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220609135707246.png" alt="image-20220609135707246" style="zoom:50%;" />

**至此完成mmap lab**，执行 `mmaptest` 和 `usertests`检验是否通过。

## 3. Linux中mmap-read和read对比

本节是对mmap方式的读取和read系统调用的一点感悟。仅为个人理解，可能有误，欢迎指正。

**以下探讨针对Linux实现，并不是xv6中的实现。**

网络上充斥着大量对mmap的误解，认为mmap方式的读取性能一定比read系统调用的性能好，实际上并不是的。甚至在大数据量下的随机读，mmap性能要更差于read系统调用。

一种最通用的说法是，用户空间发起read系统调用读取请求，数据存在着两次拷贝，一次是从磁盘拷贝到内核空间（page cache)中，另一次是从 page cache拷贝到用户空间中，所以存在两次拷贝。而mmap由于读取时，可以直接将vma映射绑定到page cache中，减少了一次从page cache到用户空间的页拷贝（即mmap读取时，仅需从磁盘拷贝到内核空间(page cache))。 相比之下，似乎mmap更快，毕竟减少了一次数据拷贝。 

但其实不然，mmap需要依赖page fault 、修改页表pte 、 invalidate TLB，这些开销是很大的。**所以需要在 “多拷贝一次page”与“ page fault, 修改页表，invalidate TLB”之间做trade off.  看哪边的成本更高。**

最后谈谈个人经历：之前阅读过LevelDB系统源码，LevelDB底层默认采用mmap的方式随机读，而与此对标的RocksDB底层默认采用read的方式随机读（具体而言是pread)。  在实现我个人的系统时，对mmap和read两种读方式都做了测试。**结果发现read形式的随机读性能是mmap方式的随机读性能的3倍左右。**

## 4. 总结

mmap，曾经困扰我很久的概念。在这个lab之后，对其有了更深的理解与感悟。
