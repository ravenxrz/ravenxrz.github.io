---
title: CS61C-lab4-RISCV汇编练习
categories: CS61C
abbrlink: a1e407f
date: 2022-05-11 13:58:06
tags:
---

## 前言

为了更好的完成MIT6.S081的后续实验，花了一点时间学习了RISCV汇编。选择的资源为 [伯克利CS61C课程](https://inst.eecs.berkeley.edu/~cs61c/sp20/)，RISCV的介绍为Lec6到Lec9。本lab为RISCV的练习实验。

有用的资源推荐：

1. [CS61C 哔哩哔哩搬运](https://www.bilibili.com/video/BV1fC4y147iZ?spm_id_from=333.1007.top_right_bar_window_custom_collection.content.click)
2. [CS61C-RISCV汇编参考card](https://inst.eecs.berkeley.edu/~cs61c/sp21/resources-pdfs/riscvcard.pdf)
3. [MIT-6.004-RISCV汇编参考card](https://6004.mit.edu/web/_static/silvina-test/resources/references/6004_isa_reference.pdf)

本lab的要求为: [点这里](https://inst.eecs.berkeley.edu/~cs61c/sp20/labs/lab04/#exercise-1-familiarizing-yourself-with-venus)

实验采用的模拟器为 [venus](https://venus.cs61c.org/)

<!--more-->

## Exercise 1

看如下代码，回答问题：

```asm
.data
.word 2, 4, 6, 8
n: .word 9

.text
main:   add     t0, x0, x0  # t0=0
        addi    t1, x0, 1   # t1=1
        la      t3, n       # t3 = n label address
        lw      t3, 0(t3)   # t3 = 9
fib:    beq     t3, x0, finish    # if t3 == 0 ? goto finish : next line
        add     t2, t1, t0  # t2 = t1 + t0 
        mv      t0, t1      # t0 = t1
mv      t1, t2              # t1 = t2
        addi    t3, t3, -1  # t3--
        j       fib   
finish: addi    a0, x0, 1   # a0 = 1
        addi    a1, t0, 0   # a1 = t0 = ?
        ecall # print integer ecall
        addi    a0, x0, 10  # call exit
        ecall # terminate ecall
```

1. What do the .data, .word, .text directives mean (i.e. what do you use them for)? Hint: think about the 4 sections of memory.

> - .data 数据段
> - .word 定义数据占用一个word（4字节）
> - .text 代码段

2. Run the program to completion. What number did the program output? What does this number represent?

> output: 32
求解斐波那契数列中的第9项是多少

3. At what address is n stored in memory? Hint: Look at the contents of the registers.

> 0x10000010
>
> 这里遗留问题：data段的起始位置是由谁决定的？

4. Without actually editing the code (i.e. without going into the “Editor” tab), have the program calculate the 13th fib number (0-indexed) by manually modifying the value of a register. You may find it helpful to first step through the code. If you prefer to look at decimal values, change the “Display Settings” option at the bottom.

> 377
直接修改t3寄存器即可。

## Exercise 2: Translating from C to RISC-V

将如下C代码翻译成RISCV汇编：

```c
int source[] = {3, 1, 4, 1, 5, 9, 0};
int dest[10];

int fun(int x) {
	return -x * (x + 1);
}

int main() {
    int k;
    int sum = 0;
    for (k = 0; source[k] != 0; k++) {
        dest[k] = fun(source[k]);
        sum += dest[k];
    }
    return sum;
}
```

汇编代码如下：

```asm
.data
source:
    .word   3
    .word   1
    .word   4
    .word   1
    .word   5
    .word   9
    .word   0
dest:
    .word   0
    .word   0
    .word   0
    .word   0
    .word   0
    .word   0
    .word   0
    .word   0
    .word   0
    .word   0

.text
main:
    addi t0, x0, 0  # t0 = 0
    addi s0, x0, 0  # s0 = 0
    la s1, source   # s1 = source address
    la s2, dest     # s2 = dest address
loop:
    slli s3, t0, 2  # s3 = t0 << 2  -> +4 字节
    add t1, s1, s3  # t1 = s1 + s3  : t1是source array element的指针
    lw t2, 0(t1)    # t2 = source[k]
    beq t2, x0, exit
    add a0, x0, t2  # a0 = t2
    addi sp, sp, -8
    sw t0, 0(sp)    # save t0
    sw t2, 4(sp)    # save t2, why we need to store t2?
    jal square
    lw t0, 0(sp)    # resotre t0
    lw t2, 4(sp)    # restore t2
    addi sp, sp, 8  # restore sp
    add t2, x0, a0  # t2 = a0 -> a0 <=> desk[k]
    add t3, s2, s3  # t3 指向当前引用的desk[k]地址
    sw t2, 0(t3)    # desk[k] = t2
    add s0, s0, t2  # s0 <==> sum
    addi t0, t0, 1  # k++
    jal x0, loop
square:
    add t0, a0, x0
    add t1, a0, x0
    addi t0, t0, 1
    addi t2, x0, -1
    mul t1, t1, t2
    mul a0, t0, t1
    jr ra
exit:
    add a0, x0, s0
    add a1, x0, x0
    ecall # Terminate ecall

```

1. The register representing the variable k.

> t0 寄存器

2. The register representing the variable sum.

> s0 寄存器

3. The registers acting as pointers to the source and dest arrays.

> - source array pointer: t1
> - dest array pointer: t3
>

4. How the pointers are manipulated in the assembly code.

> - 指针寻址，需要计算出实际的byte address。 而不像c中类似p++操作，自动会计算一个 uint 的大小，如p是int类型指针，p++ 其实将p移动了4个字节（32位系统）
> - 可通过移位操作来完成p++,p--
> - 先计算要访问的element的地址，然后通过load或者store指令，加载或保存相应value
>


## Exercise 3: Factorial

实现阶乘函数，即求解n!

```asm
.globl factorial

.data
n: .word 8

.text
main:
    la t0, n        
    lw a0, 0(t0)
    jal ra, factorial

    addi a1, a0, 0
    addi a0, x0, 1
    ecall # Print Result

    addi a1, x0, '\n'
    addi a0, x0, 11
    ecall # Print newline

    addi a0, x0, 10
    ecall # Exit

factorial:    #use recursion 
    add t0, a0, x0  # t0 = param n
    addi t1, x0, 1  # t1 == 1
    bne t0, t1, other
    ret             # a0 is 1 now, so just return it. return 1 if n == 1
other:
    addi sp, sp, -8
    sw ra, 4(sp)    # save ra address 
    sw t0, 0(sp)
    addi a0, a0,-1  # construct n-1
    jal factorial   
    lw t0, 0(sp)    # restore t0, which is n
    lw ra, 4(sp)
    mul a0, t0, a0  # calc the final ans
    addi sp, sp, 8
    ret
```

## Exercise 4: RISC-V function calling with map

假设有如下结构体定义：

```c
struct node {
    int value;
    struct node *next;
};
```

实现map函数，

```c
void map(struct node *head, int (*f)(int))
{
    if(!head) { return; }
    head->value = f(head->value);
    map(head->next,f);
}
```

即对每个链表节点都apply一次f函数。

注释给的非常清晰，很好实现。

```asm
.globl map

.text
main:
    jal ra, create_default_list
    add s0, a0, x0  # a0 = s0 is head of node list

    #print the list
    add a0, s0, x0
    jal ra, print_list

    # print a newline
    jal ra, print_newline

    # load your args
    add a0, s0, x0  # load the address of the first node into a0

    # load the address of the function in question into a1 (check out la on the green sheet)
    la a1, square

    # issue the call to map
    jal ra, map

    # print the list
    add a0, s0, x0
    jal ra, print_list

    # print another newline
    jal ra, print_newline

    addi a0, x0, 10
    ecall #Terminate the program

map:
    # Prologue: Make space on the stack and back-up registers
    # backup ra, s0
    ### YOUR CODE HERE ###
    addi sp, sp, -8
    sw ra, 4(sp) 
    sw s0, 0(sp)

    beq a0, x0, done    # If we were given a null pointer (address 0), we're done.

    add s0, a0, x0  # Save address of this node in s0
    add s1, a1, x0  # Save address of function in s1

    # Remember that each node is 8 bytes long: 4 for the value followed by 4 for the pointer to next.
    # What does this tell you about how you access the value and how you access the pointer to next?

    # load the value of the current node into a0
    # THINK: why a0?
    ### YOUR CODE HERE ###
    lw a0, 0(s0) 

    # Call the function in question on that value. DO NOT use a label (be prepared to answer why).
    # What function? Recall the parameters of "map"
    ### YOUR CODE HERE ###
    jalr s1   # jump to f(head->value)

    # store the returned value back into the node
    # Where can you assume the returned value is?
    ### YOUR CODE HERE ###
    sw a0, 0(s0)

    # Load the address of the next node into a0
    # The Address of the next node is an attribute of the current node.
    # Think about how structs are organized in memory.
    ### YOUR CODE HERE ###
    lw a0, 4(s0)

    # Put the address of the function back into a1 to prepare for the recursion
    # THINK: why a1? What about a0?
    ### YOUR CODE HERE ###
    mv a1, s1

    # recurse
    ### YOUR CODE HERE ###
    jal map

done:
    # Epilogue: Restore register values and free space from the stack
    ### YOUR CODE HERE ###
    lw s0, 0(sp)
    lw ra, 4(sp)
    addi sp, sp, 8
    jr ra # Return to caller

square:
    mul a0 ,a0, a0
    jr ra

create_default_list:
    addi sp, sp, -12
    sw  ra, 0(sp)
    sw  s0, 4(sp)
    sw  s1, 8(sp)
    li  s0, 0       # pointer to the last node we handled
    li  s1, 0       # number of nodes handled
loop:   #do...
    li  a0, 8
    jal ra, malloc      # get memory for the next node
    sw  s1, 0(a0)   # node->value = i
    sw  s0, 4(a0)   # node->next = last
    add s0, a0, x0  # last = node
    addi    s1, s1, 1   # i++
    addi t0, x0, 10
    bne s1, t0, loop    # ... while i!= 10
    lw  ra, 0(sp)
    lw  s0, 4(sp)
    lw  s1, 8(sp)
    addi sp, sp, 12
    jr ra

print_list:
    bne a0, x0, printMeAndRecurse
    jr ra       # nothing to print
printMeAndRecurse:
    add t0, a0, x0  # t0 gets current node address
    lw  a1, 0(t0)   # a1 gets value in current node
    addi a0, x0, 1      # prepare for print integer ecall
    ecall
    addi    a1, x0, ' '     # a0 gets address of string containing space
    addi    a0, x0, 11      # prepare for print string syscall
    ecall
    lw  a0, 4(t0)   # a0 gets address of next node
    jal x0, print_list  # recurse. We don't have to use jal because we already have where we want to return to in ra

print_newline:
    addi    a1, x0, '\n' # Load in ascii code for newline
    addi    a0, x0, 11
    ecall
    jr  ra

malloc:
    addi    a1, a0, 0
    addi    a0, x0 9
    ecall
    jr  ra
```

**这里的考点主要是考虑哪些寄存器需要保存，什么时候保存，是什么时候恢复。**参考lec07即可。

课堂上给出的一个基础结构为：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220511180603469.png)

我们知道riscv中一共有32个寄存器，如下：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220511180657392.png)

**其中的Saver列指明了我们在哪里需要保存些什么寄存器**，比如 `s开头一系列寄存器和sp`需要在callee中保存，而 `t开头的寄存器和ra`需要在caller中保存。**保持这种 calling convention是非常重要的**，能够得到更好的可读的汇编，而不是滥用各类寄存器，到头备受折磨。

## 总结

根据CS61C初学了部分汇编，了解了常用指令和伪指令，函数调用过程，call convention以及指令编码格式（虽然这个对本lab来说没什么用）。

之后就重新回归到MIT6.S081了。
