---
title: MIT6.S081-lazy-allocation
categories: MIT-6.S081
abbrlink: 100258ad
date: 2022-05-17 22:10:31
tags:
---

## 0. 前言

本实验为MIT 6.s081的第五个实验，主题和内存的lazy allocation相关，是page table的应用之一。其余应用包括不限于(copy on write, zero fill on demand, demand paging)。

本实验分为三个task:

1. 取消sbrk系统调用中的内存分配。即准备从eagerly分配变为lazy分配。
1. 实现lazy allocation，然后通过echo hi测试。
1. 继续完善lazy allocation，完成所有测试。包括lazytests和usertests。

<!--more-->

## 1. 原理

本次实验非常简单，简单说下目标：xv6使用提供sbrk函数用于扩展或收缩用户进程的可用物理内存，proc结构体中的`sz`字段代表当前所申请过的内存大小。 lazy allocation基于这样一个观察：

> 用户进程通过会申请比实际需要更多的内存，申请后，很多内存并不多被使用。

使用lazy allocation后， sbrk只是标记进程申请过多少内存，但是并不实际申请物理内存给进程。当进程在后续读写内存时，由于页表中并不存在该页映射（因为没有实际申请），会触发page fault， 内存发现page fault后，再分配该内存。大大加快了sbrk调用，降低了实际物理内存占用。 

目前几乎所有主流操作系统，都会采用该策略。比如linux的malloc。

为完成本实现，**要明确的是，内核如何识别page fault?**

我们曾提到trap一共有三种：

1. syscall
2. exeception
3. interrupt

page fault属于exception。所以在xv6中，可在 `usertrap` 函数中做识别处理。那如何区分page fault和其它exception呢？在ricsv中， `scause`寄存器保存了出现trap的原因，具体见下表：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220517124439957.png)

**可以看到13和15都和page fault相关。**

另外：

- `stval`寄存器保存了引发page fault时的虚拟地址。
- `sepc`寄存器保存了引发page fault时的指令地址。

## 2. 实现

### 1. Eliminate allocation from sbrk()

原文要求：

> Your first task is to delete page allocation from the sbrk(n) system call implementation, which is the function sys_sbrk() in sysproc.c. The sbrk(n) system call grows the process's memory size by n bytes, and then returns the start of the newly allocated region (i.e., the old size). Your new sbrk(n) should just increment the process's size (myproc()->sz) by n and return the old size. It should not allocate memory -- so you should delete the call to growproc() (but you still need to increase the process's size!).

修改 `sys_sbrk`系统调用：

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if (argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  myproc()->sz += n;
  /* if(growproc(n) < 0)
    return -1; */
  return addr;
}
```

如上，取消原来的growproc，即不再进行内存分配和映射。

### 2. Lazy allocation 

原文要求：

> Modify the code in trap.c to respond to a page fault from user space by mapping a newly-allocated page of physical memory at the faulting address, and then returning back to user space to let the process continue executing. You should add your code just before the `printf` call that produced the "usertrap(): ..." message. Modify whatever other xv6 kernel code you need to in order to get `echo hi` to work.

首先要增加page fault处理，该处理在 `usertrap` 函数中：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220517124610135.png" alt="image-20220517124610135" style="zoom:67%;" />

`page_fault_handler`的实现如下：

```c
// page fault handler to lazy alloc mem
// if alloc mem failed, set p->killed = 1
static void
page_fault_handler(struct proc* p)
{
  uint64 va = r_stval();
  if (va >= p->sz || va < PGROUNDDOWN(p->trapframe->sp)) {
    p->killed = 1;
    return;
  }
  va = PGROUNDDOWN(va);
  // alloc
  char* mem = kalloc();
  if (mem == 0) {
    // panic("page fault handler: out of memory");
    p->killed = 1;
    return;
  }
  // zero
  memset(mem, 0, PGSIZE);
  // map
  if (mappages(p->pagetable, va, PGSIZE, (uint64)mem, PTE_W | PTE_R | PTE_U) != 0) {
    kfree(mem);
    p->killed = 1;
  }
}
```

另外，还需要修改 `uvmunmap`函数：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220517124933887.png" alt="image-20220517124933887" style="zoom:67%;" />

以`uvmunmap`将在进程销毁时调用，当进程销毁时，其有效va（由proc->sz表示）范围内可能存在一些未分配的页。即 pte 的 `PTE_V`并不有效。这是正常逻辑，不再需要`panic`。

完成这部分代码添加后，执行 `echo hi` 应该能正常通过。

### 3. Lazytests and Usertests

原文要求：

> We've supplied you with `lazytests`, an xv6 user program that tests some specific situations that may stress your lazy memory allocator. Modify your kernel code so that all of both `lazytests` and `usertests` pass.
>
> - Handle negative sbrk() arguments.
> - Kill a process if it page-faults on a virtual memory address higher than any allocated with sbrk().
> - Handle the parent-to-child memory copy in fork() correctly.
> - Handle the case in which a process passes a valid address from sbrk() to a system call such as read or write, but the memory for that address has not yet been allocated.
> - Handle out-of-memory correctly: if kalloc() fails in the page fault handler, kill the current process.
> - Handle faults on the invalid page below the user stack.

原文已经给出了所有的提示，一条条处理即可：

首先处理 `sbrk()` 带负数参数的场景：

```c

uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if (argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  if (n > 0) {
    // lab lazy
    myproc()->sz += n;
  } else if (n < 0) {
    myproc()->sz = uvmdealloc(myproc()->pagetable, addr, addr + n);
  }
  return addr;
}
```

我个人的处理时，对于负数场景，立即释放内存，去除映射。

第二条和倒数第二条和最后一条已经在 `Lazy allocation` 小节中处理。

第三条 `Handle the parent-to-child memory copy in fork() correctly`

修改 `uvmcopy`函数：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220517125610949.png" alt="image-20220517125610949" style="zoom:67%;" />

第四条 `Handle the case in which a process passes a valid address from sbrk() to a system call such as read or write, but the memory for that address has not yet been allocated.`

这一条相对难改，需要追踪下 write和 read系统调用的流程，以`write`为例：

`write -> uservec -> usertrap -> syscall -> sys_write -> file_write -> writei -> either_copyin -> copyin`

来到 `copyin` 函数，添加代码：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220517125952256.png" alt="image-20220517125952256" style="zoom:67%;" />

这是因为 `srcva` 所代表的用户进程虚拟地址所在页，可能并不分配和映射， `walkaddr`无法找到对应页， 返回的 `pa0`将为0，对于这种情况，需要做一次 lazy allocation. 

同理，修改 `copyout` 函数：
<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220517130159684.png" alt="image-20220517130159684" style="zoom:67%;" />

最后运行 `lazytests`  和 `usertests` ，检验是否全部通过。

## 3. 总结

本实验是page table的应用之一，学习了 lazy allocation的设计与实现。回忆起之前在头条上看到别人提问 `为什么反复调用malloc函数，物理内存开销没有增长？`现在能够理解，若只malloc，而没有对申请后的页进行写入，是不会分配实际的物理页的。
