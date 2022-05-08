---
title: MIT-6.S081-lab-syscall
categories: MIT-6.S081
date: 2022-05-04 22:10:31
tags:
---



## 0. 前言

本实验为MIT 6.S081 的第二个实验。要求在xv6上自行添加两个系统调用： trace和sysinfo。

原文要求：https://pdos.csail.mit.edu/6.828/2020/labs/syscall.html

<!--more-->

## 1. 原理

只需要理解用户程序如何陷入内核，就能知道如何添加系统调用。为了安全隔离， 操作系统分为内核态和用户态，用户态程序不能直接访问系统的一些关键数据，所以每当用户程序想要访问这些数据时，首先需要通过系统调用陷入内核，提升操作权限，然后由操作系统来执行，当操作系统执行完成后，再返回至用户程序。 

再RISC-V中，通过 `ecall` 指令调用系统调用，调用前，将需要使用的系统调用号写入至 a7 寄存器，内核将根据系统调用号选择相应的系统调用函数来执行。当执行完成后，通过 `sret` 指令返回。

几个问题：

1. 如果用户程序想要传递参数给内核，比如open函数，会传递文件路径名。该如何实现？

   在xv6中，这些参数放置于 `trapframe` 中，第0个参数放置于a0，第1个参数放置于a1,以此类推。 

   <img src="https://pic.imgdb.cn/item/627293d70947543129e6ecc2.jpg" alt="image-20220504215901737" style="zoom:50%;" />

2. 内核如何拿取这些参数？

   在xv6中，提供了一系列函数来获取参数： 

   - argint, 获取整数
   - argaddr, 获取地址
   - addgstr，获取字符串
   - ...

3. 传递给内核的参数，内核是直接使用？

   这是不行的，比如用户程序直接传递一个指针给内核使用，而该指针可能是invalid的，那么kernel在使用过程过程中，会出现问题。另外，用户程序的指针采用虚拟地址，虚拟地址到物理地址过程中依赖页表(page table), 而用户程序的page table和 kernel所采用的的page table是不同的。所以也不能直接使用。

   **用户程序传递给内核的数据，或者内核的数据返回给用户程序，需要拷贝一份**。 xv6中，采用`copyinstr`来实现这类拷贝。

**搞懂这些后，来看看如何添加一个系统调用。**

1. 系统调用需要向用户层暴露操作接口，在xv6中，需要在user/user.h中声明该接口。
2. 接口的实现，即陷入内核（调用ecall指令），由汇编实现，xv6中，采用perl脚本自动生成汇编。关注**usys.pl**
3. 添加调用号
4. 陷入内核后，将进入syscall函数，syscall函数，根据 a7 寄存器中的系统调用号，在系统调用表中找到目标系统调用执行。所以我们需要在系统调用表中添加需要的系统调用。
5. 最后实现该系统调用。

来看看，目前系统中持有的系统调用表内容：

```c
static uint64 (*syscalls[])(void) = {
[SYS_fork]    = sys_fork,
[SYS_exit]    = sys_exit,
[SYS_wait]    = sys_wait,
[SYS_pipe]    = sys_pipe,
[SYS_read]    = sys_read,
[SYS_kill]    = sys_kill,
[SYS_exec]    = sys_exec,
[SYS_fstat]   = sys_fstat,
[SYS_chdir]   = sys_chdir,
[SYS_dup]     = sys_dup,
[SYS_getpid]  = sys_getpid,
[SYS_sbrk]    = sys_sbrk,
[SYS_sleep]   = sys_sleep,
[SYS_uptime]  = sys_uptime,
[SYS_open]    = sys_open,
[SYS_write]   = sys_write,
[SYS_mknod]   = sys_mknod,
[SYS_unlink]  = sys_unlink,
[SYS_link]    = sys_link,
[SYS_mkdir]   = sys_mkdir,
[SYS_close]   = sys_close,
};
```

以及syscall的实现：

```c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

额外注意的是，RSIC-V中，a0寄存器保存的是函数调用的返回值。

## 2. trace

要求:

> In this assignment you will add a system call tracing feature that may help you when debugging later labs. You'll create a new `trace` system call that will control tracing. It should take one argument, an integer "mask", whose bits specify which system calls to trace. For example, to trace the fork system call, a program calls `trace(1 << SYS_fork)`, where `SYS_fork` is a syscall number from `kernel/syscall.h`. You have to modify the xv6 kernel to print out a line when each system call is about to return, if the system call's number is set in the mask. The line should contain the process id, the name of the system call and the return value; you don't need to print the system call arguments. The `trace` system call should enable tracing for the process that calls it and any children that it subsequently forks, but should not affect other processes.

测试：

通过`make qemu`进入系统后，可执行：

```
$ trace 32 grep hello README
3: syscall read -> 1023
3: syscall read -> 966
3: syscall read -> 70
3: syscall read -> 0
$
$ trace 2147483647 grep hello README
4: syscall trace -> 0
4: syscall exec -> 3
4: syscall open -> 3
4: syscall read -> 1023
4: syscall read -> 966
4: syscall read -> 70
4: syscall read -> 0
4: syscall close -> 0
$
$ grep hello README
$
$ trace 2 usertests forkforkfork
usertests starting
test forkforkfork: 407: syscall fork -> 408
408: syscall fork -> 409
409: syscall fork -> 410
410: syscall fork -> 411
409: syscall fork -> 412
410: syscall fork -> 413
409: syscall fork -> 414
411: syscall fork -> 415
...
$   
```

首先在 `user.h`中添加接口：

```c
int trace(int);
```

然后在 `usys.pl` 中添加：

```perl
entry("trace");
```

在`syscall.h`中添加系统调用号：

```c
#define SYS_trace   22
```

接着在syscall.c中做修改：

```
extern uint64 sys_trace(void);	// 声明

static uint64 (*syscalls[])(void) = {		// 系统调用表
[SYS_fork]    = sys_fork,
[SYS_exit]    = sys_exit,
[SYS_wait]    = sys_wait,
[SYS_pipe]    = sys_pipe,
[SYS_read]    = sys_read,
[SYS_kill]    = sys_kill,
[SYS_exec]    = sys_exec,
[SYS_fstat]   = sys_fstat,
[SYS_chdir]   = sys_chdir,
[SYS_dup]     = sys_dup,
[SYS_getpid]  = sys_getpid,
[SYS_sbrk]    = sys_sbrk,
[SYS_sleep]   = sys_sleep,
[SYS_uptime]  = sys_uptime,
[SYS_open]    = sys_open,
[SYS_write]   = sys_write,
[SYS_mknod]   = sys_mknod,
[SYS_unlink]  = sys_unlink,
[SYS_link]    = sys_link,
[SYS_mkdir]   = sys_mkdir,
[SYS_close]   = sys_close,
[SYS_trace]   = sys_trace,
};

static char *syscall_names[] = {
[SYS_fork]    = "sys_fork",
[SYS_exit]    = "sys_exit",
[SYS_wait]    = "sys_wait",
[SYS_pipe]    = "sys_pipe",
[SYS_read]    = "sys_read",
[SYS_kill]    = "sys_kill",
[SYS_exec]    = "sys_exec",
[SYS_fstat]   = "sys_fstat",
[SYS_chdir]   = "sys_chdir",
[SYS_dup]     = "sys_dup",
[SYS_getpid]  = "sys_getpid",
[SYS_sbrk]    = "sys_sbrk",
[SYS_sleep]   = "sys_sleep",
[SYS_uptime]  = "sys_uptime",
[SYS_open]    = "sys_open",
[SYS_write]   = "sys_write",
[SYS_mknod]   = "sys_mknod",
[SYS_unlink]  = "sys_unlink",
[SYS_link]    = "sys_link",
[SYS_mkdir]   = "sys_mkdir",
[SYS_close]   = "sys_close",
[SYS_trace]   = "sys_trace",
};
```

trace的实现实际上就是为proc添加一个trace的mask，所以可为proc结构体添加一个 `trace_mask `变量：

```C
struct proc {
  struct spinlock lock;
  XXX
  // lab syscall: trace
  int trace_mask;               // which sys calls need to traced
};
```

现在来实现 trace, 注意如何拿到mask参数。

```c
uint64
sys_trace(void)
{
  int mask;
  if (argint(0, &mask) < 0)
    return -1;
  struct proc* p = myproc();
  p->trace_mask = mask;
  return 0;
}
```

除此外，lab要求整个祖父链需要继承该mask，所以还需要修改fork：

```
// in fork function
// copy trace mask 
np->trace_mask = p->trace_mask;
```

最后回到syscall.c:

```c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    if(p->trace_mask & (1<< num))
      printf("%d: %s -> %d\n", p->pid, syscall_names[num], p->trapframe->a0);
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

整体来说比较简单。借助官网的hint，相信很快能够实现。

## 3. Sysinfo 

这里要求实现一个类似返回当前系统可用资源的系统调用。即返回当前的活动进程数量和可用内存。

> In this assignment you will add a system call, `sysinfo`, that collects information about the running system. The system call takes one argument: a pointer to a `struct sysinfo` (see `kernel/sysinfo.h`). The kernel should fill out the fields of this struct: the `freemem` field should be set to the number of bytes of free memory, and the `nproc` field should be set to the number of processes whose `state` is not `UNUSED`. We provide a test program `sysinfotest`; you pass this assignment if it prints "sysinfotest: OK".

添加系统调用的过程和trace一样，这里主要说一下如何获取 **活动进程数和可用内存**。

由于我没怎么看6.S081的原课程视频，在实现时，没有看到关于如何获取 **活动进程数量和可用内存**。 这两个功能只能通过分析源码来实现。

### 1. 活动进程数

在xv6中，全局数组`proc[NPROC]`保存了可运行的所有进程，每个数组元素都是一个进程，每个进程的state表明当前进程的活动状态。所以可写出如下代码来获取活动进程数：

```c
int
used_proc_num(void)
{
  int cnt = 0;
  struct proc *p;
  for(p = &proc[0]; p < &proc[NPROC]; p++){
    if(p->state != UNUSED) {
      cnt++;
    }
  }
  return cnt;
}
```

### 2. 可用内存

这部分就更难看出一点，在xv6中，采用freelist的方式保存可用内存，每一个list node都是一个page，一个PAGE为4K。所以可写出如下代码：

```c
int
free_mem_bytes() {
  struct run *r;
  int free_bytes = 0;
  acquire(&kmem.lock);
  r = kmem.freelist;
  while(r){
    free_bytes += PGSIZE;
    r = r->next;
  }
  release(&kmem.lock);
  return free_bytes;
}
```

值得注意的是，在获取可用内存时，应该对kmem节点加锁，否则在并发情况下可能出现问题（不过目前还没学到并发）。

## 4. 其他

xv6的代码格式应该是基于Mozilla， 这里给一份我用的clang-format，帮助格式化。

```
---
BasedOnStyle: Mozilla
AlignConsecutiveMacros: 'true'
AlignConsecutiveAssignments: 'true'
AlignConsecutiveDeclarations: 'false'
AlignOperands: 'true'
AlignTrailingComments: 'true'
SortIncludes: 'false'

...
```

## 5. 总结

总体来说，本lab不算难。但能够帮助我们理解用户程序陷入内核，从内核返回的过程，知道了添加系统调用其实是个很简单的工作。
