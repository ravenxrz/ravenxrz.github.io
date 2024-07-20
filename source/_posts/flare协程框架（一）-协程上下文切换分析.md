---
title: flare协程框架（一）-协程上下文切换分析
categories: Linux
tags: gdb
date: 2024-07-20 16:24:32
---

## 背景

公司内项目为了提升性能，做的其中一个优化是上了协程+io uring。 我们选择的协程库为 [腾讯开源的flare](https://github.com/Tencent/flare/blob/master/flare/doc/fiber.md)。 本文分析flare的fiber是如何做上下文切换的，即用户态的context switch。

在flare中，一个用户态线程命名为fiber，所以本文也即分析fiber是如何做context  switch的。

<!--more-->

## 上下文切换 -- jump_context

在flare的源码中，上下文切换的函数为jump_context， 该逻辑由汇编实现：

一般来说，切换context的通用流程为：

- 保存原调用者现场
- 恢复当前要切换的context现场, 同时切stack
- 跳转到当前要切换的context来执行

这里的问题是，要保存/恢复什么现场？

参考如下图片，x86-64体系下各个通用寄存器的作用以及[calling-conventions](https://en.wikipedia.org/wiki/X86_calling_conventions#Register_preservation), 特别说下caller-owned regs，如果要在callee中使用caller-owned的寄存器，需要把这些寄存器先保存再使用， 对应本文分析的Jump context, 需要保存现场为**caller-owned寄存器以及RIP**（只不过RIP自动保存,无需我们关心）。

![image-20240720141534394](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720141534394.png)

> 什么是caller，什么是callee。有如下函数:
>
> ```C
> void foo() {}
> 
> void bar() {
>     foo()
> }
> ```
>
> bar为caller
>
> foo为callee
>
> 要在foo中使用 rbx, rbp, r12-r15 寄存器，需要先保存再使用，并且在回到bar的frame前，需要恢复这些寄存器的值。
>
> 通常我们在gdb中做反汇编时，进入一个函数时，前几行指令总会看到push xxx的影子，就是在保存这些寄存器。

有了如上背景知识，可以来看汇编了：

```asm
/*
    A slightly modified version of boost.context 1.69.

    // `self` is updated prior to `to` starts executing.
    //
    // RDI: self
    // RSI: to
    // RDX: context
    void jump_context(void** self, void* to, void* context);
*/

/*
            Copyright Oliver Kowalke 2009.
   Distributed under the Boost Software License, Version 1.0.
      (See accompanying file LICENSE_1_0.txt or copy at
            http://www.boost.org/LICENSE_1_0.txt)
*/

/****************************************************************************************
 *                                                                                      *
 *  ----------------------------------------------------------------------------------  *
 *  |    0    |    1    |    2    |    3    |    4     |    5    |    6    |    7    |  *
 *  ----------------------------------------------------------------------------------  *
 *  |   0x0   |   0x4   |   0x8   |   0xc   |   0x10   |   0x14  |   0x18  |   0x1c  |  *
 *  ----------------------------------------------------------------------------------  *
 *  | fc_mxcsr|fc_x87_cw|        R12        |         R13        |        R14        |  *
 *  ----------------------------------------------------------------------------------  *
 *  ----------------------------------------------------------------------------------  *
 *  |    8    |    9    |   10    |   11    |    12    |    13   |    14   |    15   |  *
 *  ----------------------------------------------------------------------------------  *
 *  |   0x20  |   0x24  |   0x28  |  0x2c   |   0x30   |   0x34  |   0x38  |   0x3c  |  *
 *  ----------------------------------------------------------------------------------  *
 *  |        R15        |        RBX        |         RBP        |        RIP        |  *
 *  ----------------------------------------------------------------------------------  *
 *                                                                                      *
 ****************************************************************************************/

.file "jump_context.S"
.text
.globl jump_context
.type jump_context,@function
.align 16
jump_context:
    leaq  -0x38(%rsp), %rsp /* prepare stack */

    movq  %r12, 0x8(%rsp)  /* save R12 */
    movq  %r13, 0x10(%rsp)  /* save R13 */
    movq  %r14, 0x18(%rsp)  /* save R14 */
    movq  %r15, 0x20(%rsp)  /* save R15 */
    movq  %rbx, 0x28(%rsp)  /* save RBX */
    movq  %rbp, 0x30(%rsp)  /* save RBP */

    /* store RSP (pointing to context-data) to variable pointed by RDI */
    movq  %rsp, (%rdi)

    /* restore RSP (pointing to context-data) from RSI */
    movq  %rsi, %rsp

    movq  0x38(%rsp), %r8  /* restore return-address */


    movq  0x8(%rsp), %r12  /* restore R12 */
    movq  0x10(%rsp), %r13  /* restore R13 */
    movq  0x18(%rsp), %r14  /* restore R14 */
    movq  0x20(%rsp), %r15  /* restore R15 */
    movq  0x28(%rsp), %rbx  /* restore RBX */
    movq  0x30(%rsp), %rbp  /* restore RBP */

    leaq  0x40(%rsp), %rsp /* prepare stack */

    /* pass `context` as first arg to fiber being run */
    /* RDX: third argument to `jump_context`, */
    /* RDI: first argument to `start_proc`. */
    movq  %rdx, %rdi

    /* indirect jump to context */
    jmp  *%r8
.size jump_context,.-jump_context

/* Mark that we don't need executable stack.  */
.section .note.GNU-stack,"",%progbits
```

在`FiberEntity::Resume`函数处，会调用 `jump_context`:

```cpp
inline void FiberEntity::Resume() noexcept {
  // Note that there are some inconsistencies. The stack we're running on is not
  // our stack. This should be easy to see, since we're actually running in
  // caller's context (including its stack).
  auto caller = GetCurrentFiberEntity();
  FLARE_DCHECK_NE(caller, this, "Calling `Resume()` on self is undefined.");

    
  // Argument `context` (i.e., `this`) is only used the first time the context
  // is jumped to (in `FiberProc`).
  jump_context(&caller->state_save_area, state_save_area, this);  // 换fiber
  
        // ...
  SetCurrentFiberEntity(caller);  // The caller has back.

  // Check for pending `ResumeOn`.
  DestructiveRunCallbackOpt(&caller->resume_proc); 
}
```

在执行 `jump_context`时，当前fiber的stack内容应该如下：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720141823501.png" alt="image-20240720141823501" style="zoom:50%;" />

回到汇编处：

一旦进入jump_context的frame，寄存器的现场为:

• RDI: &caller->state_save_area

• RSI: 要切换的fiber的state_save_area

• RDX: callee的fiber entity的this指针

```asm
jump_context:
    leaq  -0x38(%rsp), %rsp /* prepare stack */

    movq  %r12, 0x8(%rsp)  /* save R12 */
    movq  %r13, 0x10(%rsp)  /* save R13 */
    movq  %r14, 0x18(%rsp)  /* save R14 */
    movq  %r15, 0x20(%rsp)  /* save R15 */
    movq  %rbx, 0x28(%rsp)  /* save RBX */
    movq  %rbp, 0x30(%rsp)  /* save RBP */

    /* store RSP (pointing to context-data) to variable pointed by RDI */
    movq  %rsp, (%rdi)

```

> 额外说一下：rsp 为栈顶指针，栈push时，rsp是向下生长(高地址向低地址）。

第一步，将rsp向下推进0x38个字节。

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720141835528.png" alt="image-20240720141835528" style="zoom:50%;" />

第二步：保存caller-owned registers:

```asm
    movq  %r12, 0x8(%rsp)  /* save R12 */
    movq  %r13, 0x10(%rsp)  /* save R13 */
    movq  %r14, 0x18(%rsp)  /* save R14 */
    movq  %r15, 0x20(%rsp)  /* save R15 */
    movq  %rbx, 0x28(%rsp)  /* save RBX */
    movq  %rbp, 0x30(%rsp)  /* save RBP */
```

执行完如上代码后，stack内容为：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720141918881.png" alt="image-20240720141918881" style="zoom: 50%;" />

接着执行:

```

    /* store RSP (pointing to context-data) to variable pointed by RDI */
    movq  %rsp, (%rdi)

```

RDI为&caller->state_save_area，也就是让caller的state_save_area指向刚才保存的这片内存头。即:

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720141957104.png" alt="image-20240720141957104" style="zoom:50%;" />

到此步，原fiber context保存完成，下面开始切换到目标的fiber。

第三步：恢复要执行的context的现场, **首先将 RSI(要切换的fiber的state_save_area） 赋值给rsp, 这里完成切栈**

```asm
 /* restore RSP (pointing to context-data) from RSI */
    movq  %rsi, %rsp
```

接着：

```asm

    movq  0x38(%rsp), %r8  /* restore return-address */


    movq  0x8(%rsp), %r12  /* restore R12 */
    movq  0x10(%rsp), %r13  /* restore R13 */
    movq  0x18(%rsp), %r14  /* restore R14 */
    movq  0x20(%rsp), %r15  /* restore R15 */
    movq  0x28(%rsp), %rbx  /* restore RBX */
    movq  0x30(%rsp), %rbp  /* restore RBP */
```

这个过程很明显就是第二步的逆过程。

然后修正rsp位置：

```asm
leaq  0x40(%rsp), %rsp /* prepare stack */
```

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720142108148.png" alt="image-20240720142108148" style="zoom:50%;" />

第四步：执行切换后的入口函数

```cpp
    /* pass `context` as first arg to fiber being run */
    /* RDX: third argument to `jump_context`, */
    /* RDI: first argument to `start_proc`. */
    movq  %rdx, %rdi

    /* indirect jump to context */
    jmp  *%r8
```

准备调用入口函数， 入口函数需要一个参数，由rdi指定， rdi的值来自rdx(rdx 为最开始的callee的fiber entity的this指针）。

在flare的实现中，这个入口函数为:

```cpp
static void FiberProc(void* context) 
```

**实际上入口点不在这里，具体要看完make_context的分析。不过可以这样去理解。**

## 制作上下文context -  make_context

`make_context`在 `InstantiateFiberEntity` 时会调用:

```cpp
FiberEntity* InstantiateFiberEntity(SchedulingGroup* scheduling_group,
                                    FiberDesc* desc) noexcept {
  ScopedDeferred _{[&] { DestroyFiberDesc(desc); }};  // Don't leak.
  auto stack = desc->system_fiber ? CreateSystemStack() : CreateUserStack();
  auto stack_size =
      desc->system_fiber ? kSystemStackSize : FLAGS_flare_fiber_stack_size;
  auto bottom = reinterpret_cast<char*>(stack) + stack_size;
  // `FiberEntity` (and magic) is stored at the stack bottom.
  auto ptr = bottom - kFiberStackReservedSize;
  FLARE_DCHECK(reinterpret_cast<std::uintptr_t>(ptr) % alignof(FiberEntity) ==
               0);
  // NOT value-initialized intentionally, to save precious CPU cycles.
  auto fiber = new (ptr) FiberEntity;  // A new life has born.
  fiber->worker = desc->worker;
  fiber->debugging_fiber_id = id_alloc::Next<FiberIdTraits>();
  fiber->stack_size = stack_size - kFiberStackReservedSize;
  fiber->state_save_area =
      make_context(fiber->GetStackTop(), fiber->GetStackLimit(), FiberProc);  // make_context in here!!!
  fiber->scheduling_group = scheduling_group;
  fiber->state = FiberState::Ready;

  // Now move fields from `desc` into `fiber`.
  fiber->start_proc = std::move(desc->start_proc);
  fiber->exit_barrier = std::move(desc->exit_barrier);
  fiber->last_ready_tsc = desc->last_ready_tsc;
  fiber->scheduling_group_local = desc->scheduling_group_local;
  fiber->system_fiber = desc->system_fiber;
  return fiber;
}
```

分析`make_context`前，先看下这段逻辑。

分配一个stack（默认为128K），并在高地址的最后512B处，用placement_new 一个FiberEntity:

```cpp
  auto fiber = new (ptr) FiberEntity;  // A new life has born.
```

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720142332776.png" alt="image-20240720142332776" style="zoom:50%;" />

给make_context传的参数为:fiber->GetStackTop(), fiber->GetStackLimit()

```cpp
  // Get top (highest address) of the runtime stack (after skipping this
  // control structure).
  //
  // Calling this method on main fiber is undefined.
  void* GetStackTop() const noexcept {
    // The runtime stack is placed right below us.
    return reinterpret_cast<char*>(const_cast<FiberEntity*>(this));
  }
```

也就是:

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720142404346.png" alt="image-20240720142404346" style="zoom:50%;" />

fiber->GetStackLimit()

```cpp
  //   fiber->stack_size = stack_size - kFiberStackReservedSize;
  
  // Get stack size.
  std::size_t GetStackLimit() const noexcept { return stack_size; }
```

make_context的函数签名为:

```cpp
void* make_context(void* sp, std::size_t size, void (*start_proc)(void*));
```

所以:

- sp(RDI): 上图中的this指针处，即stack有效地址的的高地址处
- size(RSI): stack的limit size，即128K-512B
- start_proc(RDX): 入口函数地址

现在看汇编：

```cpp
/*
    A slightly modified version of boost.context 1.69.

    // DO NOT RETURN FROM `start_proc`, THIS LEADS TO CRASH (IN AN UNFRIENDLY
    // WAY.).
    //
    // Returns: Jump target (same as `sp`) for **first** call to `jump_context`.
    //
    // RDI: sp
    // RSI: size (not used)
    // RDX: start_proc (jump target is passed as parameter)
    void* make_context(void* sp, std::size_t size, void (*start_proc)(void*));
*/

/*
            Copyright Oliver Kowalke 2009.
   Distributed under the Boost Software License, Version 1.0.
      (See accompanying file LICENSE_1_0.txt or copy at
            http://www.boost.org/LICENSE_1_0.txt)
*/

/****************************************************************************************
 *                                                                                      *
 *  ----------------------------------------------------------------------------------  *
 *  |    0    |    1    |    2    |    3    |    4     |    5    |    6    |    7    |  *
 *  ----------------------------------------------------------------------------------  *
 *  |   0x0   |   0x4   |   0x8   |   0xc   |   0x10   |   0x14  |   0x18  |   0x1c  |  *
 *  ----------------------------------------------------------------------------------  *
 *  | fc_mxcsr|fc_x87_cw|        R12        |         R13        |        R14        |  *
 *  ----------------------------------------------------------------------------------  *
 *  ----------------------------------------------------------------------------------  *
 *  |    8    |    9    |   10    |   11    |    12    |    13   |    14   |    15   |  *
 *  ----------------------------------------------------------------------------------  *
 *  |   0x20  |   0x24  |   0x28  |  0x2c   |   0x30   |   0x34  |   0x38  |   0x3c  |  *
 *  ----------------------------------------------------------------------------------  *
 *  |        R15        |        RBX        |         RBP        |        RIP        |  *
 *  ----------------------------------------------------------------------------------  *
 *                                                                                      *
 ****************************************************************************************/

.file "make_context.S"
.text
.globl make_context
.type make_context,@function
.align 16
make_context:
    /* first arg of make_context() == top of context-stack */
    movq  %rdi, %rax

    /* shift address in RAX to lower 16 byte boundary */
    andq  $-16, %rax  // 低4bit置0， 这里应该是为了stack高地址limit处是16B对齐的， 为什么是16B对齐，不是8B对齐？没做过硬件，不理解

    /* reserve space for context-data on context-stack */
    /* on context-function entry: (RSP -0x8) % 16 == 0 */
    leaq  -0x40(%rax), %rax

    /* third arg of make_context() == address of context-function */
    /* stored in RBX */
    movq  %rdx, 0x28(%rax)

    /* save MMX control- and status-word */
    stmxcsr  (%rax)        
    /* save x87 control-word */
    fnstcw   0x4(%rax)     // 保存和浮点运算相关的状态

    /* compute abs address of label trampoline */
    leaq  trampoline(%rip), %rcx  // 相对rip的offset寻址，结果是rcx 中的地址为trampoline label下的第一条指令的地址
    /* save address of trampoline as return-address for context-function */
    /* will be entered after calling jump_fcontext() first time */
    movq  %rcx, 0x38(%rax)

    ret /* return pointer to context-data */

trampoline:
    /* Crash on return. */
    push $0
    /* jump to context-function */
    jmp *%rbx

.size make_context,.-make_context

/* Mark that we don't need executable stack. */
.section .note.GNU-stack,"",%progbits
```

第一步，把stack有效高地址给rax，并把rax按16B对齐。

```asm
    /* first arg of make_context() == top of context-stack */
    movq  %rdi, %rax

    /* shift address in RAX to lower 16 byte boundary */
    andq  $-16, %rax
```

第二步，准备context需要的内存, 给这片区域0x40B的内存

```
    /* reserve space for context-data on context-stack */
    /* on context-function entry: (RSP -0x8) % 16 == 0 */
    leaq  -0x40(%rax), %rax
```

第三步，记录入口地址，注意在第一部分的JumpContext中说的入口地址（FiberProc)就是这里，但是真正执行的入口不在这里，见下文

```cpp
    /* third arg of make_context() == address of context-function */
    /* stored in RBX */
    movq  %rdx, 0x28(%rax)
```

第四步，保存浮点数相关的状态

```asm
    /* save MMX control- and status-word */
    stmxcsr  (%rax)         
    /* save x87 control-word */
    fnstcw   0x4(%rax)     // 保存和浮点运算相关的状态
```

第五步，保存trampoline地址到 0x38偏移量

结合jump context的分析， 0x38偏移量处的地址才是切换到本fiber context时的rip地址，**所以真正的入口地址是trampolin**

```cpp
    /* compute abs address of label trampoline */
    leaq  trampoline(%rip), %rcx  // 相对offset寻址，rcx 中的地址为trmpoline label下的第一条指令的地址
    /* save address of trampoline as return-address for context-function */
    /* will be entered after calling jump_fcontext() first time */
    movq  %rcx, 0x38(%rax)
```

第六步，ret，返回rax寄存器, 即 fiber->state_save_area =$rax

```asm
ret
```

结合jump context, 切换本fiber context后，执行的第一条指令为 trampoline 的 push $0

```asm
trampoline:
    /* Crash on return. */
    push $0
    /* jump to context-function */
    jmp *%rbx 
```

这里的rbx就是FiberProc

trampoline做的工作是push一个0x0地址到stack上作为ret时的rip，避免从FiberProc中返回（因为返回后，rip将restore成0x0，这是不可执行的，程序会core），继续分析 `FiberProc`, 看它是否会返回：

函数内有一行:

```cpp
    // No one is waiting for us, this is easy.
    GetMasterFiberEntity()->ResumeOn([self] { FreeFiberEntity(self); });
```

这里的意思是，切换到master fiber , 并在master fiber 的context上，释放本FiberEntity。 所以FiberProc并不会返回，符合预期。

### gdb单步改make_context

```cpp
  /* first arg of make_context() == top of context-stack */
    movq  %rdi, %rax
```

![image-20240720142747423](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720142747423.png)

rax 寄存器为 0x7fffe4e2be00

```asm
 andq  $-16, %rax 
```

rax结果为 0x7fffe4e2be00 已经16B对齐，所以没有效果

```asm
  /* reserve space for context-data on context-stack */
    /* on context-function entry: (RSP -0x8) % 16 == 0 */
    leaq  -0x40(%rax), %rax
```

rax结果为 0x7fffe4e2bdc0 。 此时内存分布为：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720142816180.png" alt="image-20240720142816180" style="zoom:50%;" />

```asm
    /* third arg of make_context() == address of context-function */
    /* stored in RBX */
    movq  %rdx, 0x28(%rax)
```

将rdx给到 0x28 偏移处

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720142848692.png" alt="image-20240720142848692" style="zoom:50%;" />

> 可以通过 x /10i rdx 查看是否是FiberProc地址：
>
> ![image-20240720142929995](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720142929995.png)

```cpp
    /* save MMX control- and status-word */
    stmxcsr  (%rax)         // 保存MXCSR寄存器 
    /* save x87 control-word */
    fnstcw   0x4(%rax)     // 保存和浮点运算相关的状态
```

保存多媒体和浮点运算相关的状态。

```asm
 /* compute abs address of label trampoline */
    leaq  trampoline(%rip), %rcx  // 相对offset寻址，rcx 中的地址为trmpoline label下的第一条指令的地址
```

这条指令让rcx保存 trampoline 的第一条指令的地址，实际上经过编译后的指令为:

![image-20240720143011764](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720143011764.png)

此时 rip = 0x0x00007ffff4d895a5, rip +5 也不等于0x7ffff4d895b1. 

具体的运行应该是， 5 + (rip 之后一条指令的地址），想象为在具体执行lea 指令时，rip已经指向了 0x00007ffff4d895ac。所以结果是 0x00007ffff4d895ac + 5 = 0x00007ffff4d895b1 

```asm
    movq  %rcx, 0x38(%rax)
```

保存在 0x38($rax)处：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720143048092.png" alt="image-20240720143048092" style="zoom:50%;" />

最后:

```
ret
```

即：
```
 fiber->state_save_area = $rax
```

所以 state_save_area 指向了这个stack的一部分，图示的红色部分。

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240720143131372.png" alt="image-20240720143131372" style="zoom:50%;" />
