---
title: MIT6.S081-中断-uart驱动阅读笔记
categories: MIT-6.S081
abbrlink: 6f033b7d
date: 2022-05-22 22:10:31
tags:
---

听完MIT6.S081的中断课，感觉仍不是很理解，本文摘抄自[这篇](https://zhuanlan.zhihu.com/p/352432393)文章，做些整理写给自己，部分内容可能有错，看原文会更好。

<!--more-->

## 1. 驱动(Driver)

在内核开发中，驱动代码是最多的，超过了内核核心代码。驱动管理着特定硬件，告诉硬件该执行什么操作，反馈给上层设备硬件完成了工作等等。常见驱动包括磁盘驱动，网卡驱动等等。

操作系统通过中断来关注一些特定事件，比如磁盘会通过中断告知内核完成了指定I/O，由内核再将数据拷贝进用户进程空间。中断是trap的一种，在xv6中，当外部设备发起中断时，通过 `usertrap`，再到 `devintr`中做特定处理。

通常情况下，大多数的设备驱动程序，都可以看成一个分上下部分的结构：顶部**top half**运行在内核空间中，通常由某一个进程的内核线程来运行，而底部**bottom half**则在中断产生时执行，大体上就是Interrupt handler。

当内核希望与设备进行一些交互时，请求read、write等系统调用，驱动程序的top half就会被调用，top half会根据相应请求，让设备开始执行一些具体的操作（例如从磁盘上读一块）；在相关操作完成后，设备就会产生中断，因此驱动程序的bottom half开始执行，它会查看设备完成的是什么工作，在适当的时候唤醒等待该工作的进程，同时让设备开始做新的工作。

一个设备驱动程序的top half和bottom half，可以**并发**地运行在不同的CPU上。

## 2. UART驱动

下面以UART驱动简单介绍其外部设备驱动工作原理。

### 1. 读写外部设备原理

第一个要知道的，外部设备是如何和内核间交互（或者说读写）的。

**先看硬件层面：**

**控制台Console**是与用户进行交互的硬件设备，它接受用户的输入（如键盘输入），将其传递给内核和用户程序，进行相应的处理，然后再输出结果给用户（如输出到屏幕上）。

首先，简单地看总体流程：用户将会通过**键盘**键入一连串字符，通过连接到RISC-V上的**UART串行端口**（**UART Serial-port**）传输，**控制台**驱动程序将会顺利地接收这些输入。接着，控制台驱动程序处理其中的一些特殊字符（如BackSpace和Ctrl等），并不断累积这些输入字符，直到达到完整的一行（一般用户键入Enter表示一行的结束）。最后，**用户进程**，例如**shell**，就会使用read从控制台中读取这些一行行的输入，然后由shell来具体处理它们。

QEMU仿真的UART是16550系列的芯片。在真实的计算机上，UART可能还会负责管理经典的RS232串行连接。你的键盘输入实际上就是由QEMU仿真的UART硬件传输到xv6内核的。

**再看软件层面：**

在页表的lab中，提到过内核的内存空间布局如下：

<img src="https://pic.imgdb.cn/item/6277ce800947543129504076.jpg" alt="image-20220508131437202" style="zoom:50%;" />

`KERNBASSE`之下的地址空间并不映射到RAM中，而是一些外部设备，RISCV通过将外部设备映射到指定地址（如UART0映射到0X10000000），内核通过直接读写该段地址，达到读写指定设备的目的。

### 2. UART控制寄存器

UART芯片中含有多个控制寄存器，**每个寄存器大小为1B**，这些寄存器被映射到指定位置（即上图中提到的UART0段），通过读写这些寄存器，即达到和硬件设备交互的目的。xv6中，这些寄存器定义为：

```c
// the UART control registers are memory-mapped
// at address UART0. this macro returns the
// address of one of the registers.
#define Reg(reg) ((volatile unsigned char *)(UART0 + reg))

// the UART control registers.
// some have different meanings for
// read vs write.
// see http://byterunner.com/16550.html
#define RHR 0                 // receive holding register (for input bytes)
#define THR 0                 // transmit holding register (for output bytes)
#define IER 1                 // interrupt enable register
#define IER_RX_ENABLE (1<<0)
#define IER_TX_ENABLE (1<<1)
#define FCR 2                 // FIFO control register
#define FCR_FIFO_ENABLE (1<<0)
#define FCR_FIFO_CLEAR (3<<1) // clear the content of the two FIFOs
#define ISR 2                 // interrupt status register
#define LCR 3                 // line control register
#define LCR_EIGHT_BITS (3<<0)
#define LCR_BAUD_LATCH (1<<7) // special mode to set baud rate
#define LSR 5                 // line status register
#define LSR_RX_READY (1<<0)   // input is waiting to be read from RHR
#define LSR_TX_IDLE (1<<5)    // THR can accept another character to send

#define ReadReg(reg) (*(Reg(reg)))
#define WriteReg(reg, v) (*(Reg(reg)) = (v))
```

解释一些重要的寄存器含义：

1. RHR，接收持有寄存器，保存着UART芯片收到的输入，比如键盘通过串口输入的字符，将保存在该寄存器中。
2. THR，发送持有寄存器，保存着UART芯片收到的输出，比如用户进程通过write调用，准备向外部设备输出字符。
3. IER, 中断使能寄存器，含义如下图， 所以后面的 `IER_RX_ENABLE`和`IER_TX_ENABLE`分别代表接收到和发送完成的中断使能。

![image-20220522112533331](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220522112533331.png)

> **T**he Interrupt Enable Register (IER) masks the incoming interrupts from receiver ready, transmitter empty, line status and modem status registers to the INT output pin.
>
> **IER BIT-0:**
> 0=disable the receiver ready interrupt.
> 1=enable the receiver ready interrupt.
>
> **IER BIT-1:**
> 0=disable the transmitter empty interrupt.
> 1=enable the transmitter empty interrupt.
>
> **IER BIT-2:**
> 0=disable the receiver line status interrupt.
> 1=enable the receiver line status interrupt.
>
> **IER BIT-3:**
> 0=disable the modem status register interrupt.
> 1=enable the modem status register interrupt.
>
> **IER BIT 7-4:**
> All these bits are set to logic zero.

4. LSR， Line Status Register(不知道如何翻译)，提供了数据传输到CPU的状态。所以`LSR_RX_READY`代表是否有数据接收并保存到 `RHR` 寄存器中，即表示是否接收到了数据。`LSR_TX_IDLE`代表`THR`是否为空，即表示是否需要发送数据。

> **T**his register provides the status of data transfer to CPU.
>
> **LSR BIT 0:**
> 0 = no data in receive holding register or FIFO.
> 1 = data has been receive and saved in the receive holding register or FIFO.
>
> **LSR BIT 1:**
> 0 = no overrun erro (normal)
> 1 = overrun error. A character arived before receive holding register was emptied or if FIFOs are enabled, an overrun error will occur only after the FIFO is full and the next character has been completely received in the shift register. Note that character in the shift register is overwritten, but it is not transferred to the FIFO.
>
> **LSR BIT 2:**
> 0 = no parity error (normal)
> 1 = parity error. Receive data does not have correct parity information.
>
> **LSR BIT 3:**
> 0 = no framing error (normal)
> 1 = framing error received. Received data did not have a valid stop bit.
>
> **LSR BIT 4:**
> 0 = no break condition (normal)
> 1 = receiver received a break signal (RX was low for one character time frame).
>
> **LSR BIT 5:**
> 0 = transmit holding register is full. 16550 will not accept any data for transmission.
> 1 = transmitter hold register (or FIFO) is empty. CPU can load the next character.
>
> **LSR BIT 6:**
> 0 = transmitter holding and shift registers are full.
> 1 = transmit holding register is empty. In FIFO mode this bit is set to one whenever the the transmitter FIFO and transmit shift register are empty.
>
> **LST BIT 7:**
> 0 = normal
> 1 = At least one parity error, framing error or break indicator is in the FIFO. Cleared when LSR is read.

### 3. UART驱动初始化

xv6内核开机后，会执行到`main`函数中, `main`函数首先调用 `consoleinit`函数：

```c
void
main()
{
  if(cpuid() == 0){
    consoleinit();
    ...
}
```

`consoleinit`为：

```c
void
consoleinit(void)
{
  initlock(&cons.lock, "cons");

  uartinit();

  // connect read and write system calls
  // to consoleread and consolewrite.
  devsw[CONSOLE].read = consoleread;
  devsw[CONSOLE].write = consolewrite;
}

```

看`uartinit`：

```c
void
uartinit(void)
{
  // disable interrupts.
  WriteReg(IER, 0x00);

  // special mode to set baud rate.
  WriteReg(LCR, LCR_BAUD_LATCH);

  // LSB for baud rate of 38.4K.
  WriteReg(0, 0x03);

  // MSB for baud rate of 38.4K.
  WriteReg(1, 0x00);

  // leave set-baud mode,
  // and set word length to 8 bits, no parity.
  WriteReg(LCR, LCR_EIGHT_BITS);

  // reset and enable FIFOs.
  WriteReg(FCR, FCR_FIFO_ENABLE | FCR_FIFO_CLEAR);

  // enable transmit and receive interrupts.
  WriteReg(IER, IER_TX_ENABLE | IER_RX_ENABLE);

  initlock(&uart_tx_lock, "uart");
}
```

这里完成了波特率的设定，FIFO初始化，**和 发送/接收 中断的使能初始化**，即当UART芯片收到字符，或者发送完一个字符时，会发起中断。

### 4. 键盘输入的整体流程分析

**本节的目标是分析，用户从键盘数据，到shell读取该输入是如何进行的。**

> 硬件的理解可能有问题

首先为输入的**硬件处理**，用户通过键盘输入，输入经过串口传输到 UART芯片，字符保存至RHR寄存器中（或许是先到FIFO中，不过最终是到RHR中的），UART芯片发起中断，中断首先来到PLIC(platform level interrupt controller), PLIC将中断请求路由至指定CPU（看哪些CPU对该中断感兴趣, 同时PLIC是可编程的），最后中断来到CPU。 之后CPU硬件再做一些简单工作，比如保存当前PC到 sepc 寄存器，设置`pc=stvec`寄存器，进入`uservec`。

现在来到 **软件处理部分**，`uservec`将进入 `usertrap`：

```c
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
```

`usertrap`中的`devintr`完成对外设的中断响应。

```c
  if((scause & 0x8000000000000000L) &&
     (scause & 0xff) == 9){
    // this is a supervisor external interrupt, via PLIC.

    // irq indicates which device interrupted.
    int irq = plic_claim();

    if(irq == UART0_IRQ){
      uartintr();
    } else if(irq == VIRTIO0_IRQ){
      virtio_disk_intr();
    } else if(irq){
      printf("unexpected interrupt irq=%d\n", irq);
    }

    // the PLIC allows each device to raise at most one
    // interrupt at a time; tell the PLIC the device is
    // now allowed to interrupt again.
    if(irq)
      plic_complete(irq);

    return 1;
   }
```

通过 `plic_claim` 查询当前中断请求号，如果是 UART 发起的中断，则进入 `uartintr`

```c
// handle a uart interrupt, raised because input has
// arrived, or the uart is ready for more output, or
// both. called from trap.c.
void
uartintr(void)
{
  // read and process incoming characters.
  while(1){
    int c = uartgetc();
    if(c == -1)
      break;
    consoleintr(c);
  }

  // send buffered characters.
  acquire(&uart_tx_lock);
  uartstart();
  release(&uart_tx_lock);
}
```

对于键盘输入逻辑来说，只用关心 `while`循环即可。

通过 `uartgetc` 读取 `RHR`寄存器中的值：

```c
// read one input character from the UART.
// return -1 if none is waiting.
int
uartgetc(void)
{
  if(ReadReg(LSR) & 0x01){
    // input data is ready.
    return ReadReg(RHR);
  } else {
    return -1;
  }
}
```

然后调用 `consoleintr` 执行进一步处理， `consoleintr`中将对输入字符判定处理，如果输入的是 `ctrl+p`则打印当前运行的进程表，如果输入的是 `ctrl+u`则清屏，关注switch中最后的default处理：

```c
  default:
    if(c != 0 && cons.e-cons.r < INPUT_BUF){
      c = (c == '\r') ? '\n' : c;

      // echo back to the user.
      consputc(c);

      // store for consumption by consoleread().
      cons.buf[cons.e++ % INPUT_BUF] = c;

      if(c == '\n' || c == C('D') || cons.e == cons.r+INPUT_BUF){
        // wake up consoleread() if a whole line (or end-of-file)
        // has arrived.
        cons.w = cons.e;
        wakeup(&cons.r);
      }
    }
    break;
```

首先通过 `consputc`做字符回显（到这一步，终端上就会出现输入字符了，但是这里目前还不是关注的重点）。接着将输入字符写入至 `cons.buf`中，这个buf是一个128B的循环输入队列，定义如下：

```c
  // input
#define INPUT_BUF 128
  char buf[INPUT_BUF];
  uint r;  // Read index
  uint w;  // Write index
  uint e;  // Edit index
} cons;
```

然后，看当前输入字符是否为换行符或者`ctrl+d`，如果是，**则唤醒一个等待读取输入的进程**(为什么要做唤醒处理，可看后文的解释），唤醒的进程将进行数据读取。至此，键盘输入到内核已经完成，再来看看进程是如何进行数据读取的。

假设当前要读取的进程为`shell`, shell将有如下调用链：

```
 getcmd -> gets -> read -> sys_read -> fileread
```

filread有如下代码：

```c
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].read)
      return -1;
    r = devsw[f->major].read(1, addr, n);
  } 
```

这里的 `devsw[f->major].read(1, addr, n);` 是个函数指针，再`connsoleinit`中做的初始化，最终将调用 `console_read`

```c
//
// user read()s from the console go here.
// copy (up to) a whole input line to dst.
// user_dist indicates whether dst is a user
// or kernel address.
//
int
consoleread(int user_dst, uint64 dst, int n)
{
  uint target;
  int c;
  char cbuf;

  target = n;
  acquire(&cons.lock);
  while(n > 0){
    // wait until interrupt handler has put some
    // input into cons.buffer.
    while(cons.r == cons.w){
      if(myproc()->killed){
        release(&cons.lock);
        return -1;
      }
      sleep(&cons.r, &cons.lock);
    }

    c = cons.buf[cons.r++ % INPUT_BUF];

    if(c == C('D')){  // end-of-file
      if(n < target){
        // Save ^D for next time, to make sure
        // caller gets a 0-byte result.
        cons.r--;
      }
      break;
    }

    // copy the input byte to the user-space buffer.
    cbuf = c;
    if(either_copyout(user_dst, dst, &cbuf, 1) == -1)
      break;

    dst++;
    --n;

    if(c == '\n'){
      // a whole line has arrived, return to
      // the user-level read().
      break;
    }
  }
  release(&cons.lock);

  return target - n;
}
```

这里的工作如下：

1. **如果当前 cons.buf 为空，则让当前进程等待至 `cons.r` chan上。**  这里对应了上面提到的 `wakeup`函数。
2. 如果cons.buf不为空，读取一个字符，并通过`either_copyout`将该字符拷贝到用户进程空间中。

可以发现**键盘输入和进程读取两部分是解耦的**，通过缓冲区的设计，键盘的输入可直接放入到缓冲区就返回，而用户进程读取的过程也只用查看当前缓冲区是否有数据，如果有即可读取。

至此，键盘输入到用户进程读取到该输入的流程分析完毕。总结整个如下图：

![中断-键盘输入流程](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/中断-键盘输入流程.svg)

### 5. shell写入的整体流程分析

**本节的目标是分析，shell写入一个'$'符号是如何进行的。**

shell的 `getcmd`有：

```c
int
getcmd(char *buf, int nbuf)
{
  fprintf(2, "$ ");
  ...
}
```

printf一个`$`. 调用链如下：

```shell
getcmd -> fprintf -> vprintf -> putc -> write -> sys_write -> filewrite
```

filewrite有如下代码：

```c
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].write)
      return -1;
    ret = devsw[f->major].write(1, addr, n);
  } else if(f->type == FD_INODE){
```

之后进入 `consolewrite`

```c
//
// user write()s to the console go here.
//
int
consolewrite(int user_src, uint64 src, int n)
{
  int i;

  for(i = 0; i < n; i++){
    char c;
    if(either_copyin(&c, user_src, src+i, 1) == -1) // 从进程空间拷贝进内核空间
      break;
    uartputc(c);
  }

  return i;
}
```

重点为 `uartputc`

```c

// add a character to the output buffer and tell the
// UART to start sending if it isn't already.
// blocks if the output buffer is full.
// because it may block, it can't be called
// from interrupts; it's only suitable for use
// by write().
void
uartputc(int c)
{
  acquire(&uart_tx_lock);

  if(panicked){
    for(;;)
      ;
  }

  while(1){
    if(uart_tx_w == uart_tx_r + UART_TX_BUF_SIZE){
      // buffer is full.
      // wait for uartstart() to open up space in the buffer.
      sleep(&uart_tx_r, &uart_tx_lock);
    } else {
      uart_tx_buf[uart_tx_w % UART_TX_BUF_SIZE] = c;
      uart_tx_w += 1;
      uartstart();
      release(&uart_tx_lock);
      return;
    }
  }
}
```

对于output过程来说，也存在一个输出buffer:

```c
#define UART_TX_BUF_SIZE 32
char uart_tx_buf[UART_TX_BUF_SIZE];
uint64 uart_tx_w; // write next to uart_tx_buf[uart_tx_w % UART_TX_BUF_SIZE]
uint64 uart_tx_r; // read next from uart_tx_buf[uart_tx_r % UART_TX_BUF_SIZE]
```

while循环内的工作为：

1. 如果当前buffer满，则睡眠当前进程
2. 否则写入当前字符到buffer，调用 `uartstart`，准备发送当前字符

```c
// if the UART is idle, and a character is waiting
// in the transmit buffer, send it.
// caller must hold uart_tx_lock.
// called from both the top- and bottom-half.
void
uartstart()
{
  while(1){
    if(uart_tx_w == uart_tx_r){
      // transmit buffer is empty.
      return;
    }
    
    if((ReadReg(LSR) & LSR_TX_IDLE) == 0){
      // the UART transmit holding register is full,
      // so we cannot give it another byte.
      // it will interrupt when it's ready for a new byte.
      return;
    }
    
    int c = uart_tx_buf[uart_tx_r % UART_TX_BUF_SIZE];
    uart_tx_r += 1;
    
    // maybe uartputc() is waiting for space in the buffer.
    wakeup(&uart_tx_r);
    
    WriteReg(THR, c);
  }
}
```

首先检查输出buffer是否为空，如果为空，则直接返回，接着查看当前 THR状态是否为空闲，如果不是空闲，即前面的 THR还未发送完，也直接返回。最后获取要发送的字符，执行一次wakeup（因为之前的uartputc中可能有进程正在睡眠，等待buffer不为满），写入THR寄存器，等待处理。

除了通过`uartputc`调用 `uartstart`外， 每当uart成功发送一个字符，会触发一次uart中断，在uart中断中，也可能会继续发送字符，如下面代码：

```c

// handle a uart interrupt, raised because input has
// arrived, or the uart is ready for more output, or
// both. called from trap.c.
void
uartintr(void)
{
  ...
  // send buffered characters.
  acquire(&uart_tx_lock);
  uartstart();
  release(&uart_tx_lock);
}
```

所以对于进程一次写入多个字符来说，第一次uart发送时通过进程的系统调用，之后则是uart没发送一次字符会触发一次中断，由中断继续发送。

### 6. uartputc_sync

上面分析过shell通过系统调用的执行输出，可以看到uartputc内部调用uartstart，uartstart在处理“buf为空”和“LSR_TX_IDLE ！= 0”的情况下，直接返回，并不需要等待这些情况为false，然后执行发送，换句话说**uartstart对发送字符的处理是异步的，后续交由中断来继续发送字符，即shell发起系统调用输出，和uart实际发送字符之间不要同步，只要uart在未来某个时间将输出请求执行即可。**

> 关于同步、异步、阻塞、非阻塞，可参考：[这里](https://stackoverflow.com/questions/2625493/asynchronous-and-non-blocking-calls-also-between-blocking-and-synchronous)

对于某些要求同步输出的场景，在xv6中实现了 `uartputc_sync`函数：

```c
// alternate version of uartputc() that doesn't 
// use interrupts, for use by kernel printf() and
// to echo characters. it spins waiting for the uart's
// output register to be empty.
void
uartputc_sync(int c)
{
  push_off();

  if(panicked){
    for(;;)
      ;
  }

  // wait for Transmit Holding Empty to be set in LSR.
  while((ReadReg(LSR) & LSR_TX_IDLE) == 0)
    ;
  WriteReg(THR, c);

  pop_off();
}
```

**可以看到在这里通过while循环，让CPU空转，一直检测，直到`LSR_TX_IDLE`状态为1，然后发送字符。** 而在上文的 `uartputc`中，这里是直接做了返回。

## 3. 中断并发

在consoleread和consoleintr中，我们可能已经注意到，对于驱动程序的某些数据结构，多个进程会**并发地访问**它们，因此我们需要**锁**来保护这些数据结构，并通过acquire来获取锁。

如果不使用锁的话，可能会有以下的并发问题：

- 两个不同CPU上的进程同时调用consoleread。
- 即使CPU已经在执行consoleread，但硬件要求该CPU响应一个控制台（UART）的中断。
- 当consoleread执行时，硬件可能会在不同的CPU上响应一个控制台（UART）中断。

还有一种需要注意的情况是，一个进程可能正在等待设备工作的完成，但是当该设备的工作完成并产生中断时，正在运行的是另一个进程。因为这种原因，中断处理程序不应该认为当前运行的进程就是它所要交付工作的进程。例如，中断处理程序简单地使用当前进程的页表来调用copyout是不安全的。因此，更好的方式是，中断处理程序只做很小一部分工作，例如将数据拷贝到缓冲区中，然后在top half中，唤醒特定的进程来完成剩下的工作。

## 4. 总结

本文是对MIT6.S801中断课程的补充笔记，解释了驱动的工作，内核如何和外设交互，详细分析了UART的输入输出流程。不过个人仍有一些疑问：

中断的top half和bottom half到底如何区分？
