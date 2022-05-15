---
title: MIT6.S081-traps
categories: MIT-6.S081
date: 2022-05-13 22:10:31
tags:
---

## 0. 前言

本实验为MIT 6.s081的第四个实验，实验相对简单，主题和trap相关，目的是使学生了解syscall是如何从用户态到内核态，又是如何内核态回到用户态的。

本实验分为三个task:

1. 检验学生是否熟悉RISCV-V汇编。出了几个问答题目。
2. Backtrace，实现类似gdb中bt命令的打印结果
3. Alarm，实现sigalarm系统调用，使得内核周期性回调用户提供handler函数。类似于一个定时器。

<!--more-->

## 1. 原理

要完成本实验，需要阅读xv6 book的chapter 4。

### 1. Traps

Traps可以理解成一种事件，这个事件会使得CPU临时保存当前执行指令现场，切换到其余指令处执行然后再切回来。（看过csapp的同学，理解这个应该很简单）。**目前这类事件有三类：**

1. 系统调用，syscall，用户程序使用 `ecall` 指令陷入内核执行相应的syscall
2. 异常，exception，比如出现了除0运算，OS会触发exception，并执行相应handler来处理这类问题。
3. 中断，interrupt，比如硬盘完成了read或write，要求OS响应。

**trap对于用户程序来说通常是透明的，常见的trap处理过程为：**

1. 某种事件使得CPU陷入内核
2. 内核保存用户程序的各类寄存器和状态（比如用户态之前的执行指令地址是多少）
3. 内核执行恰当的handler代码（比如系统调用）
4. 内核恢复用户程序的各类寄存器和状态，并返回到用户态
5. 用户程序继续执行

### 2. Control registers

和trap机制有关的还有一组特殊寄存器，这些寄存器被称为control registers,  包含如下寄存器：

- stvec: 内核向其中保存了trap handler函数的地址。RISCV将会跳转至此处理trap
- sepc：当一个trap发生时，RISCV保存当时的PC寄存器至至此（如用户程序发起系统调用后，之后从内核态返回时，需要继续执行当前的指令，所以需要将用户程序发起系统调用时的指令地址保存起来，riscv使用sepc寄存器来保存该值）。RISCV使用 `sret` 指令从内核态返回至用户态，sepc值将重设至pc寄存器
- scause：保存一个number，用于描述发生trap的原因
- sscratch：从书中没有看书该寄存器的作用，但是在xv6中，使用该寄存器保存了trapframe的内存地址，trapframe用于保存用户程序的所有寄存器。
- sstatus：The SIE bit in sstatus controls whether device interrupts are enabled. If the kernel clears SIE, the RISC-V will defer device interrupts until the kernel sets SIE. The SPP bit indicates whether a trap came from user mode or supervisor mode, and controls to what mode sret returns.

以上寄存器只会在supervisor模式下被修改读写。实际上在machine mode中也有类似的control registers，xv6使用这些寄存器来处理timer interrupt

每个CPU都有一组这些寄存器，所有对于多CPU（多核？）来说，可在同一时间处理多个trap。

当发生trap时，RISCV硬件将执行以下步骤(除了timer interrupts):

1. If the trap is a device interrupt, and the sstatus SIE bit is clear, don’t do any of the following.
2. Disable interrupts by clearing SIE.
3. Copy the pc to sepc.
4. Save the current mode (user or supervisor) in the SPP bit in sstatus.
5. Set scause to reﬂect the trap’s cause.
6. Set the mode to supervisor.
7. Copy stvec to the pc.
8. Start executing at the new pc.

**值得注意的是，CPU并不会切换内核页表，内核栈，也没有保存任何寄存器**。所有这些工具将在软件层来做。

### 3. Traps整体流程

下图展示了一个write系统调用的流程：

![trap过程-系统调用为例](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/trap过程-系统调用为例.svg)

#### 1. uservec

write系统调用通过ecall指令陷入内核，ecall指令完成如下工作：

1. 将模式从user模式，切换到supervisor模式

2. 将当前pc寄存器value保存到 sepc 寄存器中 (sepc代表 supervisor exeception program counter)
3. 将当前pc寄存器设置为 stvec 寄存器中的value
3. 关闭中断

*sepc和stvec的初始化由内核初始化*

接着来到汇编trampoline.S的uservec函数，uservec将保存进程的各类寄存器，代码如下：

```asm
uservec:    
	#
        # trap.c sets stvec to point here, so
        # traps from user space start here,
        # in supervisor mode, but with a
        # user page table.
        #
        # sscratch points to where the process's p->trapframe is
        # mapped into user space, at TRAPFRAME.
        #
        
	# swap a0 and sscratch
        # so that a0 is TRAPFRAME
        csrrw a0, sscratch, a0		# sscratch保存的是TRAPFRAME地址

        # save the user registers in TRAPFRAME
        sd ra, 40(a0)
        sd sp, 48(a0)
        sd gp, 56(a0)
        sd tp, 64(a0)
        sd t0, 72(a0)
        sd t1, 80(a0)
        sd t2, 88(a0)
        sd s0, 96(a0)
        sd s1, 104(a0)
        sd a1, 120(a0)
        sd a2, 128(a0)
        sd a3, 136(a0)
        sd a4, 144(a0)
        sd a5, 152(a0)
        sd a6, 160(a0)
        sd a7, 168(a0)
        sd s2, 176(a0)
        sd s3, 184(a0)
        sd s4, 192(a0)
        sd s5, 200(a0)
        sd s6, 208(a0)
        sd s7, 216(a0)
        sd s8, 224(a0)
        sd s9, 232(a0)
        sd s10, 240(a0)
        sd s11, 248(a0)
        sd t3, 256(a0)
        sd t4, 264(a0)
        sd t5, 272(a0)
        sd t6, 280(a0)

	# save the user a0 in p->trapframe->a0
        csrr t0, sscratch
        sd t0, 112(a0)

        # restore kernel stack pointer from p->trapframe->kernel_sp
        ld sp, 8(a0)		# 切换内核栈

        # make tp hold the current hartid, from p->trapframe->kernel_hartid
        ld tp, 32(a0)		# 获取core number

        # load the address of usertrap(), p->trapframe->kernel_trap
        ld t0, 16(a0)	    # 加载usertrap地址

        # restore kernel page table from p->trapframe->kernel_satp
        ld t1, 0(a0)		# 加载内核页表
        csrw satp, t1		# 切换内核页表
        sfence.vma zero, zero

        # a0 is no longer valid, since the kernel page
        # table does not specially map p->tf.

        # jump to usertrap(), which does not return
        jr t0				# 跳转至usertrap

```

这里的代码非常重要，有几个重要的问题先说明：

**1.ecall到uservec时，页表仍然为user page table，使用user page table后，CPU为什么可以执行supervisor模式下的代码呢？**

回忆user process的page table layout和kernel page table layout：

![image-20220508130849006](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220508130849006.png)

![image-20220508131437202](https://pic.imgdb.cn/item/6277ce800947543129504076.jpg)

可以发现有个trampoline page，用户进程页表和内核页表都在相同地址处映射了这段指令代码（即上文的trampoline.S)，所以即使使用用户进程页表也可以执行这部分代码。

**2. 各个寄存器是保存在哪里的？**

观察user process page table中有个trapframe，所有寄存器将保存在该frame中。 

```asm
# so that a0 is TRAPFRAME
csrrw a0, sscratch, a0		# sscratch保存的是TRAPFRAME地址
```

这里的代码使得a0指向trapframe的地址。之后就可使用 `sd` 相关指令保存寄存器。

**3. 如何恢复到内核状态?**

```asm
# restore kernel stack pointer from p->trapframe->kernel_sp
ld sp, 8(a0)		# 切换内核栈

# make tp hold the current hartid, from p->trapframe->kernel_hartid
ld tp, 32(a0)		# 获取core number

# load the address of usertrap(), p->trapframe->kernel_trap
ld t0, 16(a0)	    # 加载usertrap地址

# restore kernel page table from p->trapframe->kernel_satp
ld t1, 0(a0)		# 加载内核页表
csrw satp, t1		# 切换内核页表
sfence.vma zero, zero
```

这部分代码即在恢复内核状态，注释写得非常清楚，仅强调下为什么切换了页表，指令仍然可以正常执行。原理还是一样，因为内核页表和用户进程页表的顶端 trampoline page 映射地址完全相同，所以且切换页表不影响执行。

#### 2. usertrap

来到usertrap后，就相对简单了，因为都是C代码。

```c
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if ((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);			// 设置stvec为kernelvec，如果在内核态发生trap，CPU在kernelvec中执行

  struct proc* p = myproc();

  // save user program counter.
  p->trapframe->epc = r_sepc();			// 保存sepc

  if (r_scause() == 8) {		// 系统调用响应分支
    // system call

    if (p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;		// 注意这里将pc+4了，因为回到用户态时，是在用户态pc的下一条指令执行

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if ((which_dev = devintr()) != 0) {		// device interrupt 响应
    // ok
  } else {		// exception 响应
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if (p->killed)
    exit(-1);
    yield();
  }

  usertrapret();		// 	回到用户态
}
```

之后就会到syscall，syscall的内容，做过前面的syscall lab就非常熟悉了，略过如何添加syscall。注意这里在syscall之前需要开启中断。

#### 3. usertrapret

```c
//
// return to user space
//
void
usertrapret(void)
{
  struct proc* p = myproc();

  // we're about to switch the destination of traps from
  // kerneltrap() to usertrap(), so turn off interrupts until
  // we're back in user space, where usertrap() is correct.
  intr_off();

  // send syscalls, interrupts, and exceptions to trampoline.S
  w_stvec(TRAMPOLINE + (uservec - trampoline));		// 指向uservec

  // set up trapframe values that uservec will need when
  // the process next re-enters the kernel.
  p->trapframe->kernel_satp   = r_satp();           // kernel page table
  p->trapframe->kernel_sp     = p->kstack + PGSIZE; // process's kernel stack
  p->trapframe->kernel_trap   = (uint64)usertrap;
  p->trapframe->kernel_hartid = r_tp(); // hartid for cpuid()

  // set up the registers that trampoline.S's sret will use
  // to get to user space.

  // set S Previous Privilege mode to User.
  unsigned long x = r_sstatus();
  x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
  x |= SSTATUS_SPIE; // enable interrupts in user mode
  w_sstatus(x);

  // set S Exception Program Counter to the saved user pc.
  w_sepc(p->trapframe->epc);

  // tell trampoline.S the user page table to switch to.
  uint64 satp = MAKE_SATP(p->pagetable);

  // jump to trampoline.S at the top of memory, which
  // switches to the user page table, restores user registers,
  // and switches to user mode with sret.
  uint64 fn = TRAMPOLINE + (userret - trampoline);
  ((void (*)(uint64, uint64))fn)(TRAPFRAME, satp);
}

```

过程：

1. 重设stvec到 uservec，下一次trap时，将到uservec中处理
2. 设置trapframe的几个kernel寄存器值，下一次在uservec中使用（恢复内核状态时使用）
3. 重设sstatus寄存器值，从supervisor或者说privilege mode 切换到user mode。
4. 重设sepc寄存器，之后在 `sret`指令中才可以恢复到用户进程的执行地址。
5. 获取用户进程页表
6. 根据函数指针，调用userret

#### 4. userret

userret相对简单，代码如下：

```asm
userret:
        # userret(TRAPFRAME, pagetable)
        # switch from kernel to user.
        # usertrapret() calls here.
        # a0: TRAPFRAME, in user page table.
        # a1: user page table, for satp.

        # switch to the user page table.
        csrw satp, a1
        sfence.vma zero, zero

        # put the saved user a0 in sscratch, so we
        # can swap it with our a0 (TRAPFRAME) in the last step.
        ld t0, 112(a0)			# 加载a0，syscall的返回值
        csrw sscratch, t0		

        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld gp, 56(a0)
        ld tp, 64(a0)
        ld t0, 72(a0)
        ld t1, 80(a0)
        ld t2, 88(a0)
        ld s0, 96(a0)
        ld s1, 104(a0)
        ld a1, 120(a0)
        ld a2, 128(a0)
        ld a3, 136(a0)
        ld a4, 144(a0)
        ld a5, 152(a0)
        ld a6, 160(a0)
        ld a7, 168(a0)
        ld s2, 176(a0)
        ld s3, 184(a0)
        ld s4, 192(a0)
        ld s5, 200(a0)
        ld s6, 208(a0)
        ld s7, 216(a0)
        ld s8, 224(a0)
        ld s9, 232(a0)
        ld s10, 240(a0)
        ld s11, 248(a0)
        ld t3, 256(a0)
        ld t4, 264(a0)
        ld t5, 272(a0)
        ld t6, 280(a0)

	# restore user a0, and save TRAPFRAME in sscratch
        csrrw a0, sscratch, a0
        
        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        sret
```

完成如下工作：

1. 切换至用户页表
2. 恢复用户寄存器
3. 设置syscall返回值至a0寄存器
4. `sret`指令，恢复user pc

至此，整个trap流程分析完毕。掌握如下流程后，本lab非常简单。


## 2. 内联汇编

在本lab中会有内敛汇编相关的知识，虽然不了解也能完成这个lab，不过还是建议学习下。
具体可参考： [这里](https://xr1s.me/2018/04/02/gnu-extension-extended-asm)

## 3. 题解

### 1. RISC-V assembly 

Which registers contain arguments to functions? For example, which register holds 13 in main's call to printf?

> a3
>

Where is the call to function f in the assembly code for main? Where is the call to g? (Hint: the compiler may inline functions.)

> f: 0xe
> g: 0x0 --> 0x0


At what address is the function printf located?

> printf: 0x630
>

What value is in the register ra just after the jalr to printf in main?

> ra: 0x38
>

Run the following code.

```
unsigned int i = 0x00646c72;
printf("H%x Wo%s", 57616, &i);
```


What is the output? Here's an ASCII table that maps bytes to characters.

> output:t He110 World
> 个人觉得无需修改。 大小端编码应该由编译器帮我们做。


 In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?

```
printf("x=%d y=%d", 3);
```

>  a3 寄存器的值。 

### 2. Backtrace 

实现类似gdb中的bt命令，打印调用stack frame。

具体要求如下：

> Implement a `backtrace()` function in `kernel/printf.c`. Insert a call to this function in `sys_sleep`, and then run bttest, which calls `sys_sleep`. Your output should be as follows:
>
> ```
> backtrace:
> 0x0000000080002cda
> 0x0000000080002bb6
> 0x0000000080002898
>   
> ```
>
> After `bttest` exit qemu. In your terminal: the addresses may be slightly different but if you run `addr2line -e kernel/kernel` (or `riscv64-unknown-elf-addr2line -e kernel/kernel`) and cut-and-paste the above addresses as follows:
>
> ```
>     $ addr2line -e kernel/kernel
>     0x0000000080002de2
>     0x0000000080002f4a
>     0x0000000080002bfc
>     Ctrl-D
>   
> ```
>
> You should see something like this:
>
> ```
>     kernel/sysproc.c:74
>     kernel/syscall.c:224
>     kernel/trap.c:85
>   
> ```

要完成这个task，需要了解调用stack frame的过程，*这部分其实csapp课程说得非常仔细*。函数调用stack frame如下图：

![函数调用栈-详细](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/函数调用栈-详细.svg)

1. 每个函数调用栈开始都会保存一个 `return address`, 表示退出该stack frame时，pc应该设置到哪儿，本task实际上就是打印每个stack frame的return address。
2. 接着会保存一个frame pointer(fp), 表明上一个stack frame bottom address。 我们利用fp 寄存器在不同stack frame之间切换。
3. 在往上追溯的时候，追溯到哪儿结束？有kernel stack只有一个page，所以可以求出kernel stack的page地址作为边界判定。

在printf.c文件中添加 backtrace()函数：

```c
// Backtrace call stack
void
backtrace()
{
  // get current fp register value
  uint64 fp            = r_fp();
  uint64 ret           = 0;
  uint64 kstack_bottom = PGROUNDDOWN(fp);
  while (1) {
    ret = *(uint64*)(fp - 8);
    fp  = *(uint64*)(fp - 16);
    if (fp < kstack_bottom) {
      break;
    }
    printf("%p\n", (uint64*)ret);
  }
}
```

在 `sys_sleep` 添加backtrace：

<img src="https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220513113844145.png" alt="image-20220513113844145" style="zoom:67%;" />

启动qemu，执行 `bttest`，或者三行地址，退出qemu，执行 `addr2line -e kernel/kernel`，然后粘贴刚才的三行地址，输出

![image-20220513114010492](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220513114010492.png)

说明通过。

### 3. Alarm 

本task要求实现 ` sigalarm(interval, handler)`系统调用，使得用户进程可向内核注册一个handler，让内核每隔interval ticks就回调一次handler。

实现具体要求为：

> 
> In this exercise you'll add a feature to xv6 that periodically alerts a process as it uses CPU time. This might be useful for compute-bound processes that want to limit how much CPU time they chew up, or for processes that want to compute but also want to take some periodic action. More generally, you'll be implementing a primitive form of user-level interrupt/fault handlers; you could use something similar to handle page faults in the application, for example. Your solution is correct if it passes alarmtest and usertests.
>
> You should add a new `sigalarm(interval, handler)` system call. If an application calls `sigalarm(n, fn)`, then after every `n` "ticks" of CPU time that the program consumes, the kernel should cause application function `fn` to be called. When `fn` returns, the application should resume where it left off. A tick is a fairly arbitrary unit of time in xv6, determined by how often a hardware timer generates interrupts. If an application calls `sigalarm(0, 0)`, the kernel should stop generating periodic alarm calls.
>
> You'll find a file `user/alarmtest.c` in your xv6 repository. Add it to the Makefile. It won't compile correctly until you've added `sigalarm` and `sigreturn` system calls (see below).
>
> `alarmtest` calls `sigalarm(2, periodic)` in `test0` to ask the kernel to force a call to `periodic()` every 2 ticks, and then spins for a while. You can see the assembly code for alarmtest in user/alarmtest.asm, which may be handy for debugging. Your solution is correct when `alarmtest` produces output like this and usertests also runs correctly:
>
> ```
> $ alarmtest
> test0 start
> ........alarm!
> test0 passed
> test1 start
> ...alarm!
> ..alarm!
> ...alarm!
> ..alarm!
> ...alarm!
> ..alarm!
> ...alarm!
> ..alarm!
> ...alarm!
> ..alarm!
> test1 passed
> test2 start
> ................alarm!
> test2 passed
> $ usertests
> ...
> ALL TESTS PASSED
> $
> ```
>
> When you're done, your solution will be only a few lines of code, but it may be tricky to get it right. We'll test your code with the version of alarmtest.c in the original repository. You can modify alarmtest.c to help you debug, but make sure the original alarmtest says that all the tests pass.

这个task分为两个子task

#### 1. test0： invoke handler

先让内核每隔interval ticks可以调用handler：

首先添加`sys_alarm`和`sys_return`两个系统调用，具体添加方法已在 `syscall` lab做过，这里不在赘述。

接着实现 `sys_return` 直接返回一个0. 将在下一个子task中实现该函数。

然后实现 `sys_alarm`, alarm记录用户传入的interval和handler。为了记录该值，需要扩展proc结构：

```c
  // lab traps
  int alarm_ticks_interval;    // alarm ticks interval
  uint64 alarm_handler;        // handler to be called per `alarm_tics_interval`
  uint64 last_tick;            // the tick of alarm handler was called at last time
```

```c
uint64
sys_alarm(void)
{
  int alarm_ticks;
  uint64 alarm_handler = 0;
  acquire(&tickslock);
  p->last_tick = ticks;
  release(&tickslock);
  return 0;
}
```

值得注意的是记录 `ticks` 的值时，需要获取lock。否则可能出现data race的现象，比如另一个timer interrupt出现，修改了ticks的值。

为了应用handler，需要usertrap中进行修改：

```c
  // give up the CPU if this is a timer interrupt.
  if (which_dev == 2) {
    // alarm check
    if (p->alarm_ticks_interval != 0 && ticks - p->last_tick >= p->alarm_ticks_interval) {
      p->last_tick = ticks;
      p->trapframe->epc = p->alarm_handler;
    }
    yield();
  }
```

代码非常简单，这里最重要的时修改epc，使得其指向alarm_handler,已达到回调alarm_handler的目的。

一个问题：为什么修改epc变量就可达到回调的目的？

因为在usetrapret中：

```c
  // set S Exception Program Counter to the saved user pc.
  w_sepc(p->trapframe->epc);
```

而`sepc`将在`userret`函数中，通过`sret`指令设置pc寄存器等于`sepc`。所以当回到用户态时，用户进程将执行 `alarm_handler`.

ok，现在运行 `alarmtest` 会出现alarm！字样，不过会报错，这是正常的。为什么会报错？可以注意到，前面我们覆盖了 `sepc`寄存器为 `alarm_handler`。那原本的 `sepc`寄存器值被覆盖掉后，如何使得`user process`执行原先的指令？也就是说 `alarm_handler`执行完后，`user process`如何恢复继续执行主逻辑？这就是 下一个子task要求我们做的。

#### 2. test1/test2(): resume interrupted code

要求如下：

> Chances are that alarmtest crashes in test0 or test1 after it prints "alarm!", or that alarmtest (eventually) prints "test1 failed", or that alarmtest exits without printing "test1 passed". To fix this, you must ensure that, when the alarm handler is done, control returns to the instruction at which the user program was originally interrupted by the timer interrupt. You must ensure that the register contents are restored to the values they held at the time of the interrupt, so that the user program can continue undisturbed after the alarm. Finally, you should "re-arm" the alarm counter after each time it goes off, so that the handler is called periodically.
>
> As a starting point, we've made a design decision for you: user alarm handlers are required to call the `sigreturn` system call when they have finished. Have a look at `periodic` in `alarmtest.c` for an example. This means that you can add code to `usertrap` and `sys_sigreturn` that cooperate to cause the user process to resume properly after it has handled the alarm

这里的设计是，每个handler的最后将通过 `sys_sigreturn`告诉内核，之后恢复至 interrupted code 之前的指令地址继续执行。

所以需要修改 `sys_sigreturn` 函数，用于恢复interrupted之前的状态，思考下需要恢复哪些状态：

1. sepc寄存器需要恢复
2. 由于handler可能修改任何寄存器（实际上如果编译器在实现handler时，如果遵从了 riscv calling convention的话，那么safe register，也就是s开头的寄存器(s0-s11)是肯定不会被修改的)，所以我们需要保存整个trapframe里面的寄存器。

修改prco结构体：

```c
  struct trapframe interrupt_trapframe_save;  // save trapframe for alarm, which will be used to restore `*trapmeframe`
```

在 `usertrap`函数中，保存 trapframe：

```c
  // give up the CPU if this is a timer interrupt.
  if (which_dev == 2) {
    // alarm check
    if (p->alarm_ticks_interval != 0 && ticks - p->last_tick >= p->alarm_ticks_interval &&  p->interrupt_trapframe_save.epc == -1) {	// 最后一个判断，用于避免重复执行handler调用
      p->last_tick = ticks;
      p->interrupt_trapframe_save =  *p->trapframe;
      p->trapframe->epc = p->alarm_handler;
    }
    yield();
  }
```

在 `sys_return` 函数中，执行恢复：

```c
uint64
sys_return(void)
{
  struct proc *p = myproc();
  *p->trapframe = p->interrupt_trapframe_save;
  p->interrupt_trapframe_save.epc = -1;
  return 0;
}
```

在allocproc中添加初始化代码：

```c
  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  // Init other fields
  p->alarm_ticks_interval = 0;    // no alarm
  p->alarm_handler = 0;
  p->last_tick = 0;
  memset(&p->interrupt_trapframe_save, 0, sizeof(struct trapframe));
  p->interrupt_trapframe_save.epc = -1;           // set interrpet pc at the top of vm
```

这里唯一需要说的是我将 `p->interrupt_trapframe_save.epc`初始化为-1，由于`epc`为`uint64`，所以-1其实代表无符号最大值。 在user process的地址空间中，最顶上内存属于 `trampoline` 的最后一条指令的最后一个字节地址，因为ricsv的指令为4字节，所以这个地址相当于是个垃圾地址。用这个垃圾地址表示当前没有调用handler。

至此完成本次lab。执行 `alarmtest` 验证是否成功， 执行`usertest`验证是否有影响其他功能。

## 4. 总结

本次lab和trap相关，学习了操作系统中trap的概念，并以write系统调用为例学习了整个trap流程，包括如何从用户空间陷入内核，如何从内核空间回到用户空间。掌握了函数调用stack frame的概念，学习了riscv汇编。

