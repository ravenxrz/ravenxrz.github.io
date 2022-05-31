---

title: MIT6.S081-copy-on-write
categories: MIT-6.S081
date: 2022-05-21 22:10:31
tags:
---

## 0. 前言

本实验为MIT 6.s081的第六个实验，主题为 copy-on-write(cow, 写时复制)相关。实现进程在fork时的cow机制。

要求：

更改xv6内核 fork 机制，实现延迟分配物理page给子进程。具体而言，在fork时，只用创建子进程的页表，但无需拷贝实际的物理页，并同时更新父子进程对应页的PTE为 not writable。 当父或子进程对物理页进行实际更新时，CPU产生page fault, 内核此时才分配新物理页，执行拷贝和页表项重映射。

<!--more-->

## 1. 原理

本lab仍为页表的扩展应用之一，要实现本lab，至少需要搞懂以下几个问题：

1. fork时，如何处理父子进程的页表？

修改 `uvmcopy`, 将父子进程的PTE取消 `PTE_W`权限。

2. 产生缺页中断时，如何识别是`COW`的缺页中断，而不是平常的错误缺页？

可以利用RISCV对PTE的保留bit，来标记当前PTE对应page是COW时的page。如下图中的`RSW`位，选择其一用于标记即可。

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220521150426429.png" alt="image-20220521150426429" style="zoom:67%;" />

3. 采用引用计数对每个可用物理进行计数，只有当引用计数变为0时，才回收page。问题是如何表示每个页的引用情况？

这里有两种做法，一种是将引用计数内嵌到每个page中，修改`kalloc`，每次申请内存为4KB+引用计数所占字节的空间。这种做法类似csapp的malloclab。 

![cow-引用计数-链表](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/cow-引用计数-链表.svg)

另一种做法是将所有page的引用计数单独用一个连续内存空间来执行保存，形成类似数组的数据结构。之后可通过数组下标=物理地址/4K来进行索引。

![cow-引用计数-数组](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/cow-引用计数-数组.svg)

个人做法为数组类型做法。

## 2. 实现

**首先在`riscv.h`中增加一个 `PTW_COW`  flag：**

```c
#define PTE_COW (1L << 8)
```

接着修改 `uvmcopy`：

```c
// Given a parent process's page table, copy
// its memory into a child's page table.
// Copies both the page table and the
// physical memory.
// returns 0 on success, -1 on failure.
// frees any allocated pages on failure.
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    // remap parent memory
    *pte = ((*pte) & (~PTE_W)) | PTE_COW;
    // map parent memory to child without allocating new physical page
    pa    = PTE2PA(*pte);
    pg_ref((void *)pa);
    flags = (PTE_FLAGS(*pte) & (~PTE_W)) | PTE_COW; // clear PTE_W
    if (mappages(new, i, PGSIZE, pa, flags) != 0) {
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

这里有两个改动：

1. 修改父进程的pte，取消写权限，增加COW标记
2. 不分配实际物理页给子进程，修改flag

这里的 `pg_ref`暂时忽略，后文再说。

**接着在 `usertrap` 中处理 page fault:**

```c
if (r_scause() == 13 || r_scause() == 15) {
    uint64 va = r_stval();
    va        = PGROUNDDOWN(va);
    pte_t *pte = walk(p->pagetable, va, 0);
    if (pte == 0) {
        p->killed = 1;
        goto killed;
    }
    if(!(*pte & PTE_COW)) {
        p->killed = 1;
        goto killed;
    }
    // allocate a new page for cow
    char* mem = kalloc();
    if (mem == 0) {
        // panic("out of memory");
        p->killed = 1;
        goto killed;
    }
    uint64 pa = PTE2PA(*pte) ;
    uint flag = (PTE_FLAGS(*pte) & (~PTE_COW)) | PTE_W ;
    memcpy(mem, (char*)pa, PGSIZE);
    // remove old map entry.
    uvmunmap(p->pagetable, va, 1, 1);
    mappages(p->pagetable, va, PGSIZE, (uint64)mem, flag);
} else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
}

killed:
if(p->killed)
    exit(-1);
```

13和15为 scause 中的page fault标志，这部分在 lazy allocation lab中已经提到过。通过 walk 函数找到 va 对应的pte。 如果没有对应的pte，则kill当前进程。 否则判定pte中是否有 `COW`标志，如果有则分配新的内存页，初始化，加映射。 **这里有个注意点， 分配的新物理页，不应再带 `COW` 标志，否则会在后续对 `copy_out`函数的处理中，反复分配新页。（如果不理解，看完对 `copyout`的处理后，再来看这句话应该就能理解了）。**

**修改walk函数，增加对VA超过MAXVA的处理，这部分原因未深入研究，在usertests中会存在这样的case：**

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220521221211213.png" alt="image-20220521221211213" style="zoom:67%;" />



**接着增加对物理页的引用计数。**

前文已经提到，我采用的是单独分配一篇内存来做引用计数，每个页的引用计数宽度为一个 `int`。

我才用的方案示意图如下：

![我的引用计数方案-3](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/我的引用计数方案-3.svg)

再贴一个原xv6 kernel space映射，方便对比：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220521165355875.png" alt="image-20220521165355875" style="zoom: 33%;" />

可以看到我将原xv6 space的free memory做了分割，增加了 `ref array`和`padding`两个段，`ref array` 为引用计数数组，`padding`主要是为了新的`free memory`是4K对齐而作的padding。

>  下文的所有变量和说明对应的为我个人方案图中，如`free memory`为个人方案图中的free memory，而不是`xv6`原版中的free memory

为实现本方案，在`kalloc.c`文件中，增加两个全局变量：

```c
int  *pg_ref_arr;     // 实际上应该不用int就可以，毕竟一个页不会被引用这么多次
char *avail_mem_start;  // 可用物理内存开始地址
```

修改 `kinit`函数：

```c
void
kinit()
{
  initlock(&kmem.lock, "kmem");

  // init pg_ref
  char *pa_start = (char *)PGROUNDUP((uint64)end);
  char *pa_end = (char *)PGROUNDDOWN(PHYSTOP);
  pg_ref_arr = (int *)pa_start;
  avail_mem_start = (char*)PGROUNDUP((uint64)(pa_start + (pa_end - pa_start) / 1025));
  printf("pg_ref_arr start:%p, avail mem start:%p\n", pg_ref_arr, avail_mem_start);
  printf("max page:%d\n", (uint64)(pa_end - avail_mem_start) / 4);
  if((uint64)avail_mem_start % PGSIZE != 0) {
    panic("avail mem is not 4K-aligned");
  }
  memset(pg_ref_arr, 0, (uint64)avail_mem_start - (uint64)pg_ref_arr);

  // free avail mem
  freerange(avail_mem_start, pa_end);
}
```

关键的计算为：

```c
 avail_mem_start = (char*)PGROUNDUP((uint64)(pa_start + (pa_end - pa_start) / 1025));
```

计算公式为：
$$
\frac{free\_memory}{4096} = \frac{ref\_array}{4}
$$


因为`ref_array`为int，xv6为64位系统，所以这里除4。
$$
free\_memory = pa\_end - pa\_start - ref\_array - padding
$$
代入后，可求得：
$$
ref\_array = (pa\_end - pa\_start) / 1025
$$
这里直接向下取整，代表对padding的处理。

同时增加 `pg_ref` 和 `pg_unref`函数，用于引用计数：

```c
void
pg_ref(void* pa)
{
  if((char *)pa < avail_mem_start) {
    panic("pg_ref: pa is less than avil_mem_start");
  }
  acquire(&kmem.lock);
  int idx = ((uint64)pa - (uint64)avail_mem_start) / PGSIZE;
  // printf("ref -> pa:%p, page idx:%d, val=%d\n", pa, idx, pg_ref_arr[idx]);
  pg_ref_arr[idx]++;
  release(&kmem.lock);
}

void
pg_unref(void* pa)
{
  if((char *)pa < avail_mem_start) {
    panic("pg_ref: pa is less than avil_mem_start");
  }
  acquire(&kmem.lock);
  int idx = ((uint64)pa - (uint64)avail_mem_start) / PGSIZE;
  // printf("unref -> pa:%p, page idx:%d, val=%d\n", pa, idx, pg_ref_arr[idx]);
  pg_ref_arr[idx]--;
  if(pg_ref_arr[idx] == 0) {
    release(&kmem.lock);
    kfree(pa);
  } else {
    release(&kmem.lock);
  }
}
```

这里的注意点是加解锁问题（虽然感觉不处理应该也能过），因为可能存在同时加减引用计数，所以需要使用锁保护（或许可以使用一个单独的lock?)。

修改 `kalloc`和 `kfree`函数, 增加对引用计数的处理：

```c
// Free the page of physical memory pointed at by v,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  acquire(&kmem.lock);
  int idx = ((uint64)pa - (uint64)avail_mem_start) / PGSIZE;
  if(pg_ref_arr[idx] != 0) {
    release(&kmem.lock);
    printf("kfree: pa:%p, idx:%d, ref cnt:%d\n", pa, idx, pg_ref_arr[idx]);
    panic("kfree a page with ref cnt being non-zero");
  }

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}

// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r) {
    memset((char*)r, 5, PGSIZE); // fill with junk
    pg_ref((void*)r);
  }
  return (void*)r;
}
```



最后修改 `copyout`，将内核空间的内存拷贝到用户空间，处理方式和之前的page fault一致。

可能有同学会问，为什么需要单独处理 `copyout`， 而不是用page fault handler统一处理，因为此时处于内核空间，采用的是内核页表，对于 `dstva`的读写，并不会引起page fault。另外，即使引起了page fault，也不会采用`usertrap`处理，而是使用`kerneltrap`处理。所以此处需要单独做处理。

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220521221358727.png" alt="image-20220521221358727" style="zoom:67%;" />

## 3. 总结

cow可以极大减少一些不必要的拷贝，是page table的扩展应用之一，目前也被许多OS实现。从实现来看，也不算难，唯一的疑问在于对每一个物理页都采用一个引用计数的开销是不是比较大？不过暂时不做深入研究。
