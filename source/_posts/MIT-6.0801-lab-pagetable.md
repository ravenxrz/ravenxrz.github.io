---
title: MIT6.S081-环境搭建-pagetable
categories: MIT-6.S081
date: 2022-05-08 22:10:31
tags:
---

## 0. 前言

本实验为MIT 6.s081的第三个实验，也是第一个难度较大的实验。主要讨论页表相关的问题。

分为以下三个task:

1. 写一个页表打印函数 vmprint, 打印多级页表的pte。
2. 修改xv 6内核，让每个进程都都持有一个内核页表。
3. 修改xv 6内核，让进程在陷入内核后，无需根据 user page table来解引用进程用户空间地址。

<!--more-->

## 1. 原理

要完成本实验，需要阅读xv6 book的chapter 3。

### 1. 虚拟地址到物理地址的映射

 在risck v中，使用了三级页表。地址翻译过程如下：

<img src="https://pic.imgdb.cn/item/6277ce3909475431294ef32e.jpg" alt="image-20220508125420181" style="zoom:50%;" />

虚拟地址中只会使用末尾39 bits来做VA（虚拟地址）到PA（物理地址）的映射，其中末尾12 bit作为4KB的page。其余9 bit分别作为VPN用于所以该级别的page table。对于每个page table entry(PTE)，占用8  bytes，但是至使用了末尾的54 bit。 其中末尾10 bit作为perm flag，高44 bit为下一级页表的地址。

在 xv 6中，使用 walk函数获取一个 VA 最底层page table的PTE指针，注意这个过程是软件模拟，实际上page table的translation为硬件实现。

```c
// Return the address of the PTE in page table pagetable
// that corresponds to virtual address va.  If alloc!=0,
// create any required page-table pages.
//
// The risc-v Sv39 scheme has three levels of page-table
// pages. A page-table page contains 512 64-bit PTEs.
// A 64-bit virtual address is split into five fields:
//   39..63 -- must be zero.
//   30..38 -- 9 bits of level-2 index.
//   21..29 -- 9 bits of level-1 index.
//   12..20 -- 9 bits of level-0 index.
//    0..11 -- 12 bits of byte offset within the page.
pte_t*
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if (va >= MAXVA)
    panic("walk");

  for (int level = 2; level > 0; level--) {
    pte_t* pte = &pagetable[PX(level, va)];
    if (*pte & PTE_V) { // *pte代表的address，这里既是下一层page
                        // table的物理地址，也是用来做判定的虚拟地址，这里依赖了xv6的direct-mapping方式
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if (!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}
```

### 2. xv 6内核地址空间

下图展示了xv 6内核的地址空间，最底层 0\~0x02000000-1 并未使用。 低于0x80000000（即KERNBASE）的地址主要是一些device的映射，如VIRTIO disk用于磁盘读写。KERNBASE\~PHYSTOP才会实际映射到RAM，或者说物理地址中。值得注意的是，整个内核的映射（除了PHYSTOP之上）到物理地址的映射为一一映射，这简化了OS开发。在PHYSTOP之上为一些内核栈映射，这些内核栈将提供给不同的进程陷入内核执行时来使用，另外每个kstack都配套了一个guard page，该page并不会实际映射到物理内存中，而是作为栈溢出的保护，避免不同进程在执行过程中访问到其他栈。最顶部 trampoline，本lab不用关心。

<img src="https://pic.imgdb.cn/item/6277ce800947543129504076.jpg" alt="image-20220508131437202" style="zoom:50%;" />

### 3. xv 6用户进程地址空间

和内核地址不同，用户进程地址空间从0开始编址。最下面为文本段，而后为data段，stack/heap以及trap frame和tramepoline。这里主要关注stack内部的内容。当调用exec系统调用时，我们会传入要执行的程序名，程序运行所需参数。exec通过ELF加载必要的segment后，会分配该进程需要的用户栈，然后向栈中写入程序运行的参数信息。

<img src="https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220508130849006.png" alt="image-20220508130849006" style="zoom:50%;" />

### 4. 其他

在书中，还描述了sbrk和exec的具体实现过程，这里不予赘述。但在实现后续lab的过程中，对这些函数都应该熟悉才能方便实现。

**另外，risc-v中satp存放了当前运行进程的页表地址。在调试过程中可通过打印该寄存器来查看页表情况。**

## 2. 实现

### 1.  Print a page table

第一个task非常简单，实现一个vmprint函数，并打印进程pid=1的页表。实现的效果如下：

```
page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
.. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
.. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
.. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
.. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
```

个人实现如下：

在vm.c中添加如下代码：

```c
static void
vmprint_level(pagetable_t pagetable, int level)
{
  if (level == -1) {
    return;
  }
  if (level < -1 || level > 2) {
    panic("vmprint_level: level out of range");
  }
  // there are 2^9 = 512 PTEs in a page table
  for (int i = 0; i < 512; i++) {
    pte_t pte = pagetable[i];
    if ((pte & PTE_V)) {
      // print tree level
      for (int j = 2; j >= level; j--) {
        if (j == level) {
          printf("..%d: ", i);
        } else {
          printf(".. ");
        }
      }
      printf("pte %p pa %p\n", pte, PTE2PA(pte));
      vmprint_level((pagetable_t)PTE2PA(pte), level - 1);
    }
  }
}

/**
 * for debugging
 */
void
vmprint(pagetable_t pagetable)
{
  printf("page table %p\n", pagetable);
  vmprint_level(pagetable, 2);
}
```

修改exec.c的exec函数

```c
  // lab: pgtbl, print the first process's page table
  if (p->pid == 1) {
    vmprint(p->pagetable);
  }
```

### 2. A kernel page table per process

这个task为每个进程copy一份kernel page table，然后完成scheduler调度进程时的页表切换设置。

首先为 proc 结构体添加一个kernel page table：

```c
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  pagetable_t kpagetable;      // Kernel page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

现在，在allocproc中为kernel page table分配内存：

```c
// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.
static struct proc*
allocproc(void)
{
  struct proc* p;

  for (p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if (p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();

  // Allocate a trapframe page.
  if ((p->trapframe = (struct trapframe*)kalloc()) == 0) {
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if (p->pagetable == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  // Create kernel page table.
  p->kpagetable = proc_kpagetable(p);
  if (p->kpagetable == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

这里使用到的 `proc_kpagetable(p)`实现如下，和 `proc_pagetable`很相似，不同点在于这里分配了kstack的内存，并且固定了kstack的虚拟地址为 `TRAMPOLINE - 2 * PGSIZE;`

```c
// Create a kernel page table for a give process,
// with no user memory allocation, only allocate kstack pages
// return kernel page table if success, or failed with 0
pagetable_t
proc_kpagetable(struct proc* p)
{
  pagetable_t pagetable;
  // Create an empty page table
  pagetable = uvmcreate();
  if (pagetable == 0)
    return 0;
  // Create kernel stack and kernel stack guard page
  char* pa = kalloc();
  if (pa == 0) {
    panic("alloc kstack failed");
    uvmfree(pagetable, 0);
    return 0;
  }
  uint64 va = TRAMPOLINE - 2 * PGSIZE;
  if (mappages(pagetable, va, PGSIZE, (uint64)pa, PTE_R | PTE_W) < 0) {
    panic("mapping kstack in page table failed");
    uvmfree(pagetable, 0);
    kfree(pa);
    return 0;
  }
  p->kstack = va;
  // init other kernel page mapping, like io devices
  kvminit(pagetable);
  return pagetable;
}
```

由于做了kstack的初始化，固定了kstack的虚拟地址，原来的kstack初始化代码不再需要，所以修改 `proc_init`:

```c
void
procinit(void)
{
  struct proc* p;

  initlock(&pid_lock, "nextpid");
  for (p = proc; p < &proc[NPROC]; p++) {
    initlock(&p->lock, "proc");
  }
}
```

ok，到这里进程内核页表完成了最基本的初始化，不过还需要映射一些device（即内核地址空间那张图的KERN_BASE之下的地址空间）。这里我修改了 `  kvminit`函数，接收一个 page table作为参数，改版实现如下：

```c
/*
 *
 * kernel page table init
 * return 0, means success, non-zero means failure
 */
void
kvminit(pagetable_t pagetable)
{
  // map io devices
  // uart registers
  if (mappages(pagetable, UART0, PGSIZE, UART0, PTE_R | PTE_W) < 0) {
    panic("uart0 mapping failed");
  }
  // virtio mmio disk interface
  if (mappages(pagetable, VIRTIO0, PGSIZE, VIRTIO0, PTE_R | PTE_W) < 0) {
    panic("virtio0 mapping failed");
  }
  // PLIC
  if (mappages(pagetable, PLIC, 0x400000, PLIC, PTE_R | PTE_W) < 0) {
    panic("PLC mapping failed");
  }
  
  // // CLINT
  // if (mappages(kernel_pagetable, CLINT, 0x10000, CLINT, PTE_R | PTE_W) < 0) {
  //   panic("CLINT mapping failed");
  // }

  // map kernel text executable and read-only.
  if (mappages(pagetable, KERNBASE, (uint64)etext - KERNBASE, KERNBASE, PTE_R | PTE_X) < 0) {
    panic("kernel text mapping failed");
  }

  // map kernel data and the physical RAM we'll make use of.
  if (mappages(pagetable, (uint64)etext, PHYSTOP - (uint64)etext, (uint64)etext, PTE_R | PTE_W)) {
    panic("kernel data and free memory mapping failed");
  }

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  if (mappages(pagetable, TRAMPOLINE, PGSIZE, (uint64)trampoline, PTE_R | PTE_X) < 0) {
    panic("kernel TRAMPOLINE mapping failed");
  }
}
```

注意这里移除了对CLINT的映射，不取消之后运行os时会出现问题。该映射仅需要给全局的kernel page table映射即可。原因暂时未知，因为对该部分映射的功能还未知。

修改这部分后，原来的kvminit对全局的kernel page table初始化就被取消了，所以我们需要将该部分还原：

```c
/**
 * create a direct-map page table for the kernel.
 */
void
kpgtbl_init(void)
{
  kernel_pagetable = (pagetable_t)kalloc(); // 分配top level page table entry
  memset(kernel_pagetable, 0, PGSIZE);      // 清空
  kvminit(kernel_pagetable);
  // CLINT
  if (mappages(kernel_pagetable, CLINT, 0x10000, CLINT, PTE_R | PTE_W) < 0) {
    panic("CLINT mapping failed");
  }
}
```

修改main函数

```c
// start() jumps here in supervisor mode on all CPUs.
void
main()
{
  if(cpuid() == 0){
    consoleinit();
#if defined(LAB_PGTBL) || defined(LAB_LOCK)
    statsinit();
#endif
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
    kpgtbl_init();   // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    ...
 }
```

现在完成了对process kernel page table的添加，但是还需要在恰当的地方执行释放。

修改freeproc：

```c

// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc* p)
{
  if (p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if (p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  if (p->kpagetable) {
    proc_kfreepagetable(p->kpagetable, p->sz);
  }
  p->pagetable  = 0;
  p->kpagetable = 0;
  p->sz         = 0;
  p->pid        = 0;
  p->parent     = 0;
  p->name[0]    = 0;
  p->chan       = 0;
  p->killed     = 0;
  p->xstate     = 0;
  p->state      = UNUSED;
}
```

`kfreepagetable`函数实现为：

```c
void
proc_kfreepagetable(pagetable_t pagetable, uint64 sz)
{
  // unmapping
  uvmunmap(pagetable, UART0, 1, 0);
  uvmunmap(pagetable, VIRTIO0, 1, 0);
  // uvmunmap(pagetable, CLINT, 0x10000 / PGSIZE, 0);
  uvmunmap(pagetable, PLIC, 0x400000 / PGSIZE, 0);
  uvmunmap(pagetable, KERNBASE, ((uint64)etext - KERNBASE) / PGSIZE, 0);
  uvmunmap(pagetable, (uint64)etext, (PHYSTOP - (uint64)etext) / PGSIZE, 0);
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);

  // kstack
  uvmunmap(pagetable, TRAMPOLINE - 2 * PGSIZE, 1, 1);
  // page table本身内存
  uvmfree(pagetable, 0);
}
```

取消各类地址的映射，释放kstack物理内存和page table所占内存。

至此，完成了kernel page table的添加与释放，但是还未将其应用起来，修改 `scheduler`:

```c
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run.
//  - swtch to start running that process.
//  - eventually that process transfers control
//    via swtch back to the scheduler.
void
scheduler(void)
{
  struct proc* p;
  struct cpu* c = mycpu();

  c->proc = 0;
  for (;;) {
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();

    int found = 0;
    for (p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if (p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc  = p;
        load_kpgtbl(p->kpagetable);		// 必须要在switch之前切换页表
        swtch(&c->context, &p->context);

         // 切换会内核页表? load_kpgtbl(kernel_pagetable);	
          
        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        found = 1;
      }
      release(&p->lock);
    }
#if !defined(LAB_FS)
    if (found == 0) {
      intr_on();
      asm volatile("wfi");
    }
#else
    ;
#endif
    // TODO: 也许应该将这部分移动到上面的 if(found == 0), 但是暂时未知该处作用，所以重开分支
    if (found == 0) {
      // No running process. use kernel pagetable
      load_kpgtbl(kernel_pagetable);
    }
  }
}

void
load_kpgtbl(pagetable_t pagetable)
{
  w_satp(MAKE_SATP(pagetable));
  sfence_vma();
}
```

其实就是在进程上下文切换时，替换satp寄存器中的页表地址内容。**但这里一定要注意在 swtich 之前切换页表。** switch的工作其实就是将当前CPU的寄存器(c->context)组保存到起来(每个cpu都有自己stack，代码中为stack0数组），然后加载进程的寄存器组（p->context)。

**至于为什么需要在switch前切换页表，将在本小结末尾做详细分析。**

如果没有可以切换的进程，需要使用全局的kernel page table. 即：

```c
    // TODO: 也许应该将这部分移动到上面的 if(found == 0), 但是暂时未知该处作用，所以重开分支
    if (found == 0) {
      // No running process. use kernel pagetable
      load_kpgtbl(kernel_pagetable);
    }
```

最后还有一处修改：

```c
// translate a kernel virtual address to
// a physical address. only needed for
// addresses on the stack.
// assumes va is page aligned.
uint64
kvmpa(uint64 va)
{
  uint64 off = va % PGSIZE;
  pte_t* pte;
  uint64 pa;
  struct proc* p = myproc();

  pte = walk(p->kpagetable, va, 0);		// 修改为每个process的kernel page table
  if (pte == 0)
    panic("kvmpa");
  if ((*pte & PTE_V) == 0)
    panic("kvmpa");
  pa = PTE2PA(*pte);
  return pa + off;
}
```

至此，能够完成usertests的所有测试。该测试时间较长。

#### 为什么switch前需要加载process的kernel page table?

在解释前，让我们看看如果不加载process的kernel page table，会造成什么后果。先看switch.S的代码：

```asm
.globl swtch
swtch:
        sd ra, 0(a0)		// 保存return address, 之后切换到内核进时需要该地址找到调用switch的下一行指令地址, 暂时不用关心该值
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)		// 存放的是forkret函数地址
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret				// 设置pc寄存器到ra，即forkret函数，跳转到forkret函数执行
```

为什么0(a1)指向forkret？ 首先明白 a1寄存器其实就是 &p->context 地址。 也就是说0(a1)在取p->context.ra的值，p->context.ra的初始化在exec函数中的末尾：

```c
  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;
```

我们还注意到p->context.sp初始化为p->kstack+PGSIZE，即指向进程内核栈的栈底（栈是从内存地址由上往下push), 而在上面的汇编中，加载了该值到sp寄存器中。那什么时候会使用到sp寄存器？其实就在 forkret 函数的首行指令：

![image-20220508203941573](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220508203941573.png)

**如果我们在switch前未切换到process的内核页表，那么此时 `satp`寄存器中的将是全局内核页表，全局内核页表中并不包含对每个进程的内核栈地址映射，所以在执行该条指令时，系统将会出现错误。**

**故而，我们必须要在switch前将 `stap`寄存器设置为进程内核页表，才能使用进程的内核栈。**

#### 切换回内核时是否需要重新加载全局内核页表？

我发现网络上几乎所有人的解答都在如下代码中添加切换回内核页表的代码，即schedulerhan's函数中：
<img src="C:\Users\Raven\AppData\Roaming\Typora\typora-user-images\image-20220508205520001.png" alt="image-20220508205520001" style="zoom:67%;" />

但实际上我个人在实现时并没有做这个操作，个人觉得没有必要。首先要明白系统哪里返回到switch之后的指令？这部分其实来自 `sched`函数：

<img src="C:\Users\Raven\AppData\Roaming\Typora\typora-user-images\image-20220508204719959.png" alt="image-20220508204719959" style="zoom: 67%;" />

其实也就是从cpu stack中恢复寄存器组。注意，到此为止没有修改过 `satp`寄存器的值，也就没有切换内核页表。回到问题本身，为什么没有必要切换到全局内核页表，因为当前进程内核页表本身就是一个全局内核页表的副本，只不过新增了一个kernel stack（task 3中将进一步修改）。所以内核的地址空间映射均有，也就没有必要执行切换了。

#### 题外话：什么时候加载进程用户空间页表？

至此，我们知道了内核本身有全局内核页表，每个进程也有自己的内核页表，此外，每个进程也有一个用户空间的页表，那么该用户空间页表什么时候应用到 `satp`中？ 可以猜想到，是在从内核态切换到用户态的时候，那么xv 6的内核态切换到用户态是如何做的？

在`forkret`的最后一步中，有 `usertrapret`函数：

<img src="https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220508205202906.png" alt="image-20220508205202906" style="zoom: 67%;" />

该函数的最后会执行该操作。

<img src="C:\Users\Raven\AppData\Roaming\Typora\typora-user-images\image-20220508205241942.png" alt="image-20220508205241942" style="zoom:67%;" />

### 3. Simplify `copyin/copyinstr` 

此task结合task2，目的是当进程陷入内核后，来自进程的指针能够直接解引用，而不需要依靠 user page table. 在xv 6当前的方案中，如果内核要使用用户进程的地址，需要使用类似如下方法：

```c
int
copyin(pagetable_t pagetable, char* dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;

  while (len > 0) {
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);	// 通过walk addr访问 pagetable 中 srcva对应的pa
    if (pa0 == 0)
      return -1;
    n = PGSIZE - (srcva - va0);
    if (n > len)
      n = len;
    memmove(dst, (void*)(pa0 + (srcva - va0)), n);

    len -= n;
    dst += n;
    srcva = va0 + PGSIZE;
  }
  return 0;
}
```

可以注意到，每次需要访问srcva（进程的虚拟地址），我们都需要通过 pagetable(实际上就是进程的page table)，然后walkaddr（内部需要递归+循环）。这样效率慢，没有充分利用 risc-v的硬件特性。

所以，这个task要求我们修改内核，支持进程指针的直接解引用，即如下：

```c
// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin_new(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  struct proc *p = myproc();

  if (srcva >= p->sz || srcva+len >= p->sz || srcva+len < srcva)
    return -1;
  memmove((void *) dst, (void *)srcva, len);		// 直接使用!
  stats.ncopyin++;   // XXX lock
  return 0;
}
```

为了做到这一点，首先替换`copyin`和`copyintstr`为`copyin_new`和`copyinstr_new`:

```c
int
copyin(pagetable_t pagetable, char* dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable, dst, srcva, len);	// 实际上内部并不会使用pagetable，不知道为什么xv6这样设计接口
}

int
copyinstr(pagetable_t pagetable, char* dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable, dst, srcva, max);
}
```

凡是对user page table添加了映射的，都需要对kernel page table添加：

**先看 `userinit` 函数：**

<img src="C:\Users\Raven\AppData\Roaming\Typora\typora-user-images\image-20220508211648436.png" alt="image-20220508211648436" style="zoom:67%;" />

```c
// Load the user initcode into address 0 of pagetable,
// for the very first process.
// sz must be less than a page.
void
ukvminit(struct proc* proc, uchar* src, uint sz)
{
  char* mem;

  if (sz >= PGSIZE)
    panic("inituvm: more than a page");
  mem = kalloc();
  memset(mem, 0, PGSIZE);
  mappages(proc->pagetable, 0, PGSIZE, (uint64)mem, PTE_W | PTE_R | PTE_X | PTE_U);
  mappages(proc->kpagetable, 0, PGSIZE, (uint64)mem, PTE_W | PTE_R | PTE_X);
  memmove(mem, src, sz);
}
```

其实就是重复添加了一个映射到kernel page table.

**再看fork函数：**

<img src="https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220508210856153.png" alt="image-20220508210856153" style="zoom:50%;" />

其中 `copy_u2k_ptbl`的实现为：

```c
// copy `user` page table mapping to `ken`
// this copying does not allocate any physical mem
// while do kern page table mapping, the PTE_U will be unset
// 0 means success, -1 failed
// NOTE: used by fork and exec
int
copy_u2k_ptbl(pagetable_t user, pagetable_t ken, uint64 sz)
{
  pte_t* pte;
  uint64 pa, i;
  uint flags;

  for (i = 0; i < sz; i += PGSIZE) {
    if ((pte = walk(user, i, 0)) == 0) {
      panic("copy_u2k_ptbl: pte should b exist in user page table");
    }
    if ((*pte & PTE_V) == 0) {
      panic("copy_u2k_ptbl: page not present");
    }
    pa    = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    UNSET_FLAG(flags, PTE_U);
    if (mappages(ken, i, PGSIZE, pa, flags) != 0) {
      panic("copy_u2k_ptbl: mappages failed");
    }
  }

  return 0;
}

// 其中 UNSET_FLAG的宏为
#define UNSET_FLAG(target_flag, flag) (target_flag = (target_flag) & (~flag))
```

这里其实就是将 `user page table` 的所有项映射到 `kernel page table` ， 另外取消 PTE_U 的flag ，否则内核无法使用这些地址。

**sbrk函数**

sbrk为系统内存的扩展或收缩函数。内部依赖 `growproc`  函数，而 `growproc`又依赖 `ukvmalloc`(个人实现)和 `uvmdealloc`函数：

<img src="C:\Users\Raven\AppData\Roaming\Typora\typora-user-images\image-20220508214034560.png" alt="image-20220508214034560" style="zoom:50%;" />

**这里要注意，用户虚存地址不能超过 PLIC，否则会和kernel page table重叠。** 所以添加了第一框出的判定。

其余相关函数的实现如下：

```c
// Allocate PTEs and physical memory to grow process from oldsz to
// newsz, which need not be page aligned.  Returns new size or 0 on error.
// NOTE: the difference between uvmalloc and this is that
// in addition to map user page table, we do kernel page table mapping too
uint64
ukvmalloc(pagetable_t uptbl, pagetable_t kptbl, uint64 oldsz, uint64 newsz)
{
  char* mem;
  uint64 a;

  if (newsz < oldsz)
    return oldsz;

  oldsz = PGROUNDUP(oldsz);
  for (a = oldsz; a < newsz; a += PGSIZE) {
    mem = kalloc();
    if (mem == 0) {
      uvmdealloc(uptbl, a, oldsz);
      kvmdealloc(kptbl, a, oldsz);
      return 0;
    }
    memset(mem, 0, PGSIZE);
    if (mappages(uptbl, a, PGSIZE, (uint64)mem, PTE_W | PTE_X | PTE_R | PTE_U) != 0) {
      kfree(mem);
      uvmdealloc(uptbl, a, oldsz);
      kvmdealloc(kptbl, a, oldsz);
      return 0;
    }
    if (mappages(kptbl, a, PGSIZE, (uint64)mem, PTE_W | PTE_X | PTE_R) != 0) {
      panic("ukvmalloc: mapping kernel page table entry failed");
      // kfree(mem);
      // uvmdealloc(pagetable, a, oldsz);
      return 0;
    }
  }
  return newsz;
}

// Deallocate user pages to bring the process size from oldsz to
// newsz.  oldsz and newsz need not be page-aligned, nor does newsz
// need to be less than oldsz.  oldsz can be larger than the actual
// process size.  Returns the new process size.
uint64
uvmdealloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
{
  if (newsz >= oldsz)
    return oldsz;

  if (PGROUNDUP(newsz) < PGROUNDUP(oldsz)) {
    int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
    uvmunmap(pagetable, PGROUNDUP(newsz), npages, 1);
  }

  return newsz;
}

// TODO: 合并uvmdealloc和kvmdealloc函数，加入一个是否释放内存的标志位
uint64
kvmdealloc(pagetable_t pagetable, uint64 oldsz, uint64 newsz)
{
  if (newsz >= oldsz)
    return oldsz;

  if (PGROUNDUP(newsz) < PGROUNDUP(oldsz)) {
    int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz)) / PGSIZE;
    uvmunmap(pagetable, PGROUNDUP(newsz), npages, 0);
  }

  return newsz;
}
```

**最后看看 exec 函数:**

<img src="https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220508214644027.png" alt="image-20220508214644027" style="zoom: 67%;" />

这里有一个比较容易出现bug的点，也是我注释中的代码。exec我原先的实现为释放kernel page table和user page table, 然后重新将新的user page table拷贝到kernel page table。这里的问题在于，os在执行exec函数时，执行的kernel page table就是当前的p->kpagetable, 如果将其释放，将无法或许需要执行的指令。于是会出现 `kerneltrap panic`。所以这里的正确方式是，不释放kernel page table，只做重映射即可（实际上我想user page table应该也可以这样做，不过暂时先这样改吧)。

另外，记住在添加page table映射时，应该注意 PLIC的limit：

<img src="https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220508215045988.png" alt="image-20220508215045988" style="zoom:50%;" />

由于向process kernel page table添加了user page table的映射项目，所以在freeproc时，应该对该部分映射项目做unmap：

<img src="https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220508215224672.png" alt="image-20220508215224672" style="zoom:67%;" />

注意这里其实我修改了 `proc_kfreepagetable`的签名，所以还需要修改`freeproc`调用处：

<img src="https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220508215353188.png" alt="image-20220508215353188" style="zoom:67%;" />

## 3. 总结

第一个相对有难度的lab，首先当然是对页表映射过程有个复习，然后学习了 kernel address layout 和 user address layout，包括内核栈和用户栈的创建和使用，进程的上下文切换。 简单了解了 RISC-V汇编(感觉还是得提前系统过一遍汇编，不然目前遇到一个不会的指令查一个还是有点麻烦)。