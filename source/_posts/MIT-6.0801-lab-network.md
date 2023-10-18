---
title: MIT6.S081-lab-network
categories: MIT-6.S081
abbrlink: '15618765'
date: 2022-06-16 22:10:31
tags:
---

## 0. 前言

本实验为MIT6.s8081的第十一个实验，也是最后一个实验，主题为 **网卡驱动** 相关。实验要求实现网卡驱动的收发过程核心逻辑。

本实验的前置知识较多，上来就是E1000网卡手册，但实际上需要阅读和理解的部分很少，实现的代码也只是几十行。

<!--more-->

## 1. 实验要求

> Your job is to complete `e1000_transmit()` and `e1000_recv()`, both in `kernel/e1000.c`, so that the driver can transmit and receive packets. You are done when `make grade` says your solution passes all the tests.
>
> While writing your code, you'll find yourself referring to the E1000 [Software Developer's Manual](https://pdos.csail.mit.edu/6.828/2020/readings/8254x_GBe_SDM.pdf). Of particular help may be the following sections:
>
> - Section 2 is essential and gives an overview of the entire device.
> - Section 3.2 gives an overview of packet receiving.
> - Section 3.3 gives an overview of packet transmission, alongside section 3.4.
> - Section 13 gives an overview of the registers used by the E1000.
> - Section 14 may help you understand the init code that we've provided.

Browse the E1000 [Software Developer's Manual](https://pdos.csail.mit.edu/6.828/2020/readings/8254x_GBe_SDM.pdf). This manual covers several closely related Ethernet controllers. QEMU emulates the 82540EM. Skim Chapter 2 now to get a feel for the device. To write your driver, you'll need to be familiar with Chapters 3 and 14, as well as 4.1 (though not 4.1's subsections). You'll also need to use Chapter 13 as a reference. The other chapters mostly cover components of the E1000 that your driver won't have to interact with. Don't worry about the details at first; just get a feel for how the document is structured so you can find things later. The E1000 has many advanced features, most of which you can ignore. Only a small set of basic features is needed to complete this lab.

The `e1000_init()` function we provide you in `e1000.c` configures the E1000 to read packets to be transmitted from RAM, and to write received packets to RAM. This technique is called DMA, for direct memory access, referring to the fact that the E1000 hardware directly writes and reads packets to/from RAM.

Because bursts of packets might arrive faster than the driver can process them, `e1000_init()` provides the E1000 with multiple buffers into which the E1000 can write packets. The E1000 requires these buffers to be described by an array of "descriptors" in RAM; each descriptor contains an address in RAM where the E1000 can write a received packet. `struct rx_desc` describes the descriptor format. The array of descriptors is called the receive ring, or receive queue. It's a circular ring in the sense that when the card or driver reaches the end of the array, it wraps back to the beginning. `e1000_init()` allocates `mbuf` packet buffers for the E1000 to DMA into, using `mbufalloc()`. There is also a transmit ring into which the driver places packets it wants the E1000 to send. `e1000_init()` configures the two rings to have size `RX_RING_SIZE` and `TX_RING_SIZE`.

When the network stack in `net.c` needs to send a packet, it calls `e1000_transmit()` with an mbuf that holds the packet to be sent. Your transmit code must place a pointer to the packet data in a descriptor in the TX (transmit) ring. `struct tx_desc` describes the descriptor format. You will need to ensure that each mbuf is eventually freed, but only after the E1000 has finished transmitting the packet (the E1000 sets the `E1000_TXD_STAT_DD` bit in the descriptor to indicate this).

When the E1000 receives each packet from the ethernet, it first DMAs the packet to the mbuf pointed to by the next RX (receive) ring descriptor, and then generates an interrupt. Your `e1000_recv()` code must scan the RX ring and deliver each new packet's mbuf to the network stack (in `net.c`) by calling `net_rx()`. You will then need to allocate a new mbuf and place it into the descriptor, so that when the E1000 reaches that point in the RX ring again it finds a fresh buffer into which to DMA a new packet.

In addition to reading and writing the descriptor rings in RAM, your driver will need to interact with the E1000 through its memory-mapped control registers, to detect when received packets are available and to inform the E1000 that the driver has filled in some TX descriptors with packets to send. The global variable `regs` holds a pointer to the E1000's first control register; your driver can get at the other registers by indexing `regs` as an array. You'll need to use indices `E1000_RDT` and `E1000_TDT` in particular.

To test your driver, run `make server` in one window, and in another window run `make qemu` and then run `nettests` in xv6. The first test in `nettests` tries to send a UDP packet to the host operating system, addressed to the program that `make server` runs. If you haven't completed the lab, the E1000 driver won't actually send the packet, and nothing much will happen.

After you've completed the lab, the E1000 driver will send the packet, qemu will deliver it to your host computer, `make server` will see it, it will send a response packet, and the E1000 driver and then `nettests` will see the response packet. Before the host sends the reply, however, it sends an "ARP" request packet to xv6 to find out its 48-bit Ethernet address, and expects xv6 to respond with an ARP reply. `kernel/net.c` will take care of this once you have finished your work on the E1000 driver. If all goes well, `nettests` will print `testing ping: OK`, and `make server` will print `a message from xv6!`.

tcpdump -XXnr packets.pcap should produce output that starts like this:

```
reading from file packets.pcap, link-type EN10MB (Ethernet)
15:27:40.861988 IP 10.0.2.15.2000 > 10.0.2.2.25603: UDP, length 19
        0x0000:  ffff ffff ffff 5254 0012 3456 0800 4500  ......RT..4V..E.
        0x0010:  002f 0000 0000 6411 3eae 0a00 020f 0a00  ./....d.>.......
        0x0020:  0202 07d0 6403 001b 0000 6120 6d65 7373  ....d.....a.mess
        0x0030:  6167 6520 6672 6f6d 2078 7636 21         age.from.xv6!
15:27:40.862370 ARP, Request who-has 10.0.2.15 tell 10.0.2.2, length 28
        0x0000:  ffff ffff ffff 5255 0a00 0202 0806 0001  ......RU........
        0x0010:  0800 0604 0001 5255 0a00 0202 0a00 0202  ......RU........
        0x0020:  0000 0000 0000 0a00 020f                 ..........
15:27:40.862844 ARP, Reply 10.0.2.15 is-at 52:54:00:12:34:56, length 28
        0x0000:  ffff ffff ffff 5254 0012 3456 0806 0001  ......RT..4V....
        0x0010:  0800 0604 0002 5254 0012 3456 0a00 020f  ......RT..4V....
        0x0020:  5255 0a00 0202 0a00 0202                 RU........
15:27:40.863036 IP 10.0.2.2.25603 > 10.0.2.15.2000: UDP, length 17
        0x0000:  5254 0012 3456 5255 0a00 0202 0800 4500  RT..4VRU......E.
        0x0010:  002d 0000 0000 4011 62b0 0a00 0202 0a00  .-....@.b.......
        0x0020:  020f 6403 07d0 0019 3406 7468 6973 2069  ..d.....4.this.i
        0x0030:  7320 7468 6520 686f 7374 21              s.the.host!
```

Your output will look somewhat different, but it should contain the strings "ARP, Request", "ARP, Reply", "UDP", "a.message.from.xv6" and "this.is.the.host".

`nettests` performs some other tests, culminating in a DNS request sent over the (real) Internet to one of Google's name server. You should ensure that your code passes all these tests, after which you should see this output:

```
$ nettests
nettests running on port 25603
testing ping: OK
testing single-process pings: OK
testing multi-process pings: OK
testing DNS
DNS arecord for pdos.csail.mit.edu. is 128.52.129.126
DNS OK
all tests passed.
```

You should ensure that `make grade` agrees that your solution passes.

**一些Hints:**

Start by adding print statements to `e1000_transmit()` and `e1000_recv()`, and running `make server` and (in xv6) `nettests`. You should see from your print statements that `nettests` generates a call to `e1000_transmit`.

Some hints for implementing `e1000_transmit`:

- First ask the E1000 for the TX ring index at which it's expecting the next packet, by reading the `E1000_TDT` control register.
- Then check if the the ring is overflowing. If `E1000_TXD_STAT_DD` is not set in the descriptor indexed by `E1000_TDT`, the E1000 hasn't finished the corresponding previous transmission request, so return an error.
- Otherwise, use `mbuffree()` to free the last mbuf that was transmitted from that descriptor (if there was one).
- Then fill in the descriptor. `m->head` points to the packet's content in memory, and `m->len` is the packet length. Set the necessary cmd flags (look at Section 3.3 in the E1000 manual) and stash away a pointer to the mbuf for later freeing.
- Finally, update the ring position by adding one to `E1000_TDT` modulo `TX_RING_SIZE`.
- If `e1000_transmit()` added the mbuf successfully to the ring, return 0. On failure (e.g., there is no descriptor available to transmit the mbuf), return -1 so that the caller knows to free the mbuf.

Some hints for implementing `e1000_recv`:

- First ask the E1000 for the ring index at which the next waiting received packet (if any) is located, by fetching the `E1000_RDT` control register and adding one modulo `RX_RING_SIZE`.
- Then check if a new packet is available by checking for the `E1000_RXD_STAT_DD` bit in the `status` portion of the descriptor. If not, stop.
- Otherwise, update the mbuf's `m->len` to the length reported in the descriptor. Deliver the mbuf to the network stack using `net_rx()`.
- Then allocate a new mbuf using `mbufalloc()` to replace the one just given to `net_rx()`. Program its data pointer (`m->head`) into the descriptor. Clear the descriptor's status bits to zero.
- Finally, update the `E1000_RDT` register to be the index of the last ring descriptor processed.
- `e1000_init()` initializes the RX ring with mbufs, and you'll want to look at how it does that and perhaps borrow code.
- At some point the total number of packets that have ever arrived will exceed the ring size (16); make sure your code can handle that.

You'll need locks to cope with the possibility that xv6 might use the E1000 from more than one process, or might be using the E1000 in a kernel thread when an interrupt arrives.

## 2. 解析

> 下文中使用到的手册为: [点这里](https://pdos.csail.mit.edu/6.828/2020/readings/8254x_GBe_SDM.pdf)

要完成本实验，主要是要理解E1000网卡驱动的packet发送与接收过程，涉及到的数据结构。

### 1. 一些概念

首先说说一些概念：

**网络栈:** 所谓网络栈，可以理解为我们在计网课上学过的TCP/IP网络协议，在xv6中，实现了各层中的部分协议，比如数据链路层中以太网协议，网络层的IP协议，ARP协议，传输层UDP协议，应用层DNS协议。如下图是网络栈的结构图：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616145246522.png" alt="image-20220616145246522" style="zoom: 50%;" />

**NIC：**网卡硬件，在本实验中，特指E1000网卡

**LAN：** local area network, 局域网。

现在简述下packet的收发过程：

当应用程序想要发送一个网络packet时，该packet经过各层协议逐层封装，到达host的NIC driver，NIC driver和NIC硬件通过通信协议，将packet发送出去。反过来，当NIC接收到packet时，和NIC driver通过协议将接收到的数据转换为packet, 通过DMA的方式存放在host的内存中。

在本实验中，`应用程序数据<->sockets<->udp<->ip<->arp<->NIC driver`这些逻辑处理已经提前实现，我们唯一需要完成的是NIC driver与NIC之间的通信。而这个通信过程中，最为重要的部分是，**driver如何存放和处理来自应用程序和NIC硬件的packet数据。**

### 2. tx_ring 和 rx_ring

这其中的核心结构叫做 **tx_ring**(发送时使用)以及**rx_ring**(接收时使用)。

#### 1. tx_ring 相关字段含义

**以tx_ring为例**，tx_ring本身为一个环形队列数组，如下图(摘抄自手册P66页)：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616210438091.png" alt="image-20220616210438091" style="zoom:50%;" />

这里使用的是典型的生产者和消费者模型，队列的写者为上层应用程序，读者为E1000硬件（硬件将读取到的数据转为电信号发送出去）。该队列有**HEAD**和**TAIL**两个指针，上层应用写时，写入至TAIL指针所指位置（写完后TAIL+1），E1000硬件读时，从HEAD开始读（读完HEAD+1），当HEAD=TAIL时，硬件认为无可读数据。

每个队列item称为Transmit Descriptor, Descriptor的结构为如下（先了解即可，不用完全理解其含义，可在看完后续的逻辑处理后，再回到此部分查看各字段含义会更清晰）：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616211808315.png)

对应xv6代码的tx_desc结构体：

```c
// [E1000 3.3.3]
struct tx_desc
{
  uint64 addr;
  uint16 length;
  uint8 cso;
  uint8 cmd;
  uint8 status;
  uint8 css;
  uint16 special;
};
```

每个字段的解释如下：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616211822795.png" alt="image-20220616211822795" style="zoom:50%;" />

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616211831624.png" alt="image-20220616211831624" style="zoom:50%;" />

特别关注CMD和STA字段含义：

**CMD字段：**

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616212006382.png)

对于完成本实验，只用关注如上红框所选字段，对应代码：

```c
/* Transmit Descriptor command definitions [E1000 3.3.3.1] */
#define E1000_TXD_CMD_EOP    0x01 /* End of Packet */
#define E1000_TXD_CMD_RS     0x08 /* Report Status */
```

两字段含义如下：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616212047291.png)

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616212053220.png)

当设置descriptor的cmd的RS时，每当网卡完成数据传输，将设置Descriptor Done bit（该bit在Status字段中）。

**下面来看Status字段：**

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616212307771.png" alt="image-20220616212307771" style="zoom:50%;" />

只用关注DD字段：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616212324993.png)

#### 2. 发送过程

到此，介绍了发送过程需要的各字段含义，但是我们仍不知道具体该如何发送。比如上层应用要将发送的数据包存放到哪儿？transmit descriptor并不能存放数据。

关注 transmit descriptor的Buffer Address字段，该字段指向要传输的数据的内存地址。

**在xv6中，数据包最终将封装在 `mbuf`  结构中。并且每一个transmit descriptor对应一个mbuf结构， 如下：**

```c
static struct tx_desc tx_ring[TX_RING_SIZE] __attribute__((aligned(16)));
static struct mbuf *tx_mbufs[TX_RING_SIZE];
```

至此，所有会用到的数据结构均已介绍，下面说下处理逻辑：

1. 应用程序将要发送的packet封装成mbuf，调用 e1000_transmit 进行传输。 （软件处理）
2. 在e1000_transmit中，首先获取下一个用于传输的tx desc, 即环形队列中TAIL指针对应的desc。（软件处理）
3. 检测该desc原先的packet（如果有的话）是否已经完成发送，具体通过判定desc.status的 DD字段是否被set。（软件处理）
4. 如果desc原先的packet已经发送完成，则重置desc的addr字段为mbuf，更新cmd字段为 `RS | EOP` (RS要求硬件在发送完本packet后，设置desc.status的DD字段)。（软件处理）
5. E1000网卡通过HEAD指针获取desc，通过desc找到要发送的数据，执行发送，当发送完成后，设置desc的status DD字段为1. (硬件处理)

下面为具体的代码实现：

```c
int
e1000_transmit(struct mbuf *m)
{
  //
  // Your code here.
  //
  // the mbuf contains an ethernet frame; program it into
  // the TX descriptor ring so that the e1000 sends it. Stash
  // a pointer so that it can be freed after sending.
  //
  acquire(&e1000_lock);
  // First ask the E1000 for the TX ring index at which it's expecting the next packet, by reading the E1000_TDT control register.
  uint32 index = regs[E1000_TDT];
  // Then check if the the ring is overflowing. If E10E1000_TXD_STAT_DD00_TXD_STAT_DD is not set in the descriptor indexed by E1000_TDT, 
  // the E1000 hasn't finished the corresponding previous transmission request, so return an error.
  if( (tx_ring[index].status & E1000_TXD_STAT_DD) == 0) {
    release(&e1000_lock);
    return -1;
  }
  // Otherwise, use mbuffree() to free the last mbuf that was transmitted from that descriptor (if there was one).
  if(tx_mbufs[index]) {
    mbuffree(tx_mbufs[index]);
  }
  // Then fill in the descriptor. m->head points to the packet's content in memory, and m->len is the packet length. Set the necessary cmd flags (look at Section 3.3 in the E1000 manual) and stash away a pointer to the mbuf for later freeing.
  // NOTE: 可能需要将buffer拆分为多个descriptor来对应
  tx_ring[index].addr = (uint64)m->head;
  tx_ring[index].length = m->len;
  tx_ring[index].cmd = E1000_TXD_CMD_RS   // report status has to to be set so that hardware will set the  E1000_TXD_STAT_DD flag after processing this packet
                      | E1000_TXD_CMD_EOP;
  tx_mbufs[index] = m;
  // Finally, update the ring position by adding one to E1000_TDT modulo TX_RING_SIZE.
  regs[E1000_TDT] = (index + 1) % TX_RING_SIZE;
  release(&e1000_lock);
  return 0;
}
```

#### 3. rx_ring 相关字段含义

同tx_ring相同， rx_ring也是一个唤醒队列：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616213958659.png" alt="image-20220616213958659" style="zoom: 67%;" />

每当硬件收到一个packet后，将写入至HEAD指针所指向的desc，并产生中断，host os收到中断后，从TAIL指针处获取要接收的packet的相关信息，并将其封装为mbuf,上传给网络栈进行处理。

每个rx_ring内的desc结构如下：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616214154919.png" alt="image-20220616214154919" style="zoom:50%;" />

对应代码：

```c
struct rx_desc
{
  uint64 addr;       /* Address of the descriptor's data buffer */
  uint16 length;     /* Length of data DMAed into data buffer */
  uint16 csum;       /* Packet checksum */
  uint8 status;      /* Descriptor status */
  uint8 errors;      /* Descriptor Errors */
  uint16 special;
};
```

仅用关注 status 字段：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616214253654.png" alt="image-20220616214253654" style="zoom:50%;" />

本实验仅使用了EOP和DD字段，两字段含义如下：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_img/image-20220616214319098.png" alt="image-20220616214319098" style="zoom:50%;" />

#### 4. 接收过程

同tx_ring一样， 每个rx_desc也有一个对应mbuf用于存放packet:

```c
static struct rx_desc rx_ring[RX_RING_SIZE] __attribute__((aligned(16)));
static struct mbuf *rx_mbufs[RX_RING_SIZE];
```

了解了如上数据结构，现在来说说发送过程：

1. 网卡硬件接收到packet电信号，转换为packet结构后，通过HEAD指针获取可以使用的desc，并将packet通过DMA的方式存放在desc所指向的addr mbuf中。（硬件处理）
2. 完成packet存放后，设置desc的相关元数据，如开启desc.status的DD bit，同时产生中断（硬件处理）
3. host os接收到中断后，调用`e1000_recv`函数处理。（软件处理）
4. 在一次中断过程中，可以处理多个连续的packes，所以中断处理函数内部为一个循环，循环内部首先通过TAIL指针，获取下一个可以处理的desc（软件处理）
5. 获取desc对应的mbuf，调用 net_rx 将其传递给网络栈处理。（软件处理）
6. 上述网络栈处理完mbuf后，会释放mbuf，所以 net_rx 函数后，还需申请新的 mbuf，已供后续的硬件存放。（软件处理）
7. 循环处理，直至没有网络包需要处理，通过检测desc.status 的 DD bit来判定。（软件处理）

转换为代码如下：

```c
static void
e1000_recv(void)
{
  //
  // Your code here.
  //
  // Check for packets that have arrived from the e1000
  // Create and deliver an mbuf for each packet (using net_rx()).
  //
  // First ask the E1000 for the ring index at which the next waiting received packet (if any) is located, by fetching the E1000_RDT control register and adding one modulo RX_RING_SIZE.
  uint32 index;
  for (index = (regs[E1000_RDT] + 1) % RX_RING_SIZE;; index = (index + 1) % RX_RING_SIZE) {
    // Then check if a new packet is available by checking for the E1000_RXD_STAT_DD bit in the status portion of the descriptor. If not, stop.
    if ((rx_ring[index].status & E1000_RXD_STAT_DD) == 0) { // the packet has not been processed
      break;
    }

    // Otherwise, update the mbuf's m->len to the length reported in the descriptor. Deliver the mbuf to the network stack using net_rx().
    struct mbuf* m = rx_mbufs[index];
    m->len         = rx_ring[index].length;
    net_rx(m); // m will be freed inside net_rx
    // Then allocate a new mbuf using mbufalloc() to replace the one just given to net_rx(). Program its data pointer (m->head) into the descriptor. Clear the descriptor's status bits to zero.
    rx_mbufs[index] = mbufalloc(0);
    if (!rx_mbufs[index])
      panic("e1000");
    rx_ring[index].addr   = (uint64)rx_mbufs[index]->head;
    rx_ring[index].length = 0;
    rx_ring[index].status = 0;
  }
  // Finally, update the E1000_RDT register to be the index of the last ring descriptor processed.
  regs[E1000_RDT] = index - 1;
}
```

可以看到，整个处理过程没有像 `e1000_transmit`那样需要加锁处理，**这是因为 `e1000_recv`是在中断处理函数中使用，而xv6不支持中断嵌套的，所以同一时刻，当一个CPU收到该中断的时候，会关闭所有CPU的中断。**

### 3. e1000网卡初始化

最后，e1000是如何找到要发送或者要接收的包的？换句话说，环形队列的HEAD指针初始化，环形队列初始化在哪里？这部分逻辑在 e1000_init函数中，如下：

```c
void
e1000_init(uint32 *xregs)
{
  int i;

  initlock(&e1000_lock, "e1000");

  regs = xregs;

  // Reset the device
  regs[E1000_IMS] = 0; // disable interrupts
  regs[E1000_CTL] |= E1000_CTL_RST;
  regs[E1000_IMS] = 0; // redisable interrupts
  __sync_synchronize();

  // [E1000 14.5] Transmit initialization
  memset(tx_ring, 0, sizeof(tx_ring));
  for (i = 0; i < TX_RING_SIZE; i++) {
    tx_ring[i].status = E1000_TXD_STAT_DD;
    tx_mbufs[i] = 0;
  }
  regs[E1000_TDBAL] = (uint64) tx_ring;
  if(sizeof(tx_ring) % 128 != 0)
    panic("e1000");
  regs[E1000_TDLEN] = sizeof(tx_ring);
  regs[E1000_TDH] = regs[E1000_TDT] = 0;
  
  // [E1000 14.4] Receive initialization
  memset(rx_ring, 0, sizeof(rx_ring));
  for (i = 0; i < RX_RING_SIZE; i++) {
    rx_mbufs[i] = mbufalloc(0);
    if (!rx_mbufs[i])
      panic("e1000");
    rx_ring[i].addr = (uint64) rx_mbufs[i]->head;
  }
  regs[E1000_RDBAL] = (uint64) rx_ring;
  if(sizeof(rx_ring) % 128 != 0)
    panic("e1000");
  regs[E1000_RDH] = 0;
  regs[E1000_RDT] = RX_RING_SIZE - 1;
  regs[E1000_RDLEN] = sizeof(rx_ring);

  // filter by qemu's MAC address, 52:54:00:12:34:56
  regs[E1000_RA] = 0x12005452;
  regs[E1000_RA+1] = 0x5634 | (1<<31);
  // multicast table
  for (int i = 0; i < 4096/32; i++)
    regs[E1000_MTA + i] = 0;

  // transmitter control bits.
  regs[E1000_TCTL] = E1000_TCTL_EN |  // enable
    E1000_TCTL_PSP |                  // pad short packets
    (0x10 << E1000_TCTL_CT_SHIFT) |   // collision stuff
    (0x40 << E1000_TCTL_COLD_SHIFT);
  regs[E1000_TIPG] = 10 | (8<<10) | (6<<20); // inter-pkt gap

  // receiver control bits.
  regs[E1000_RCTL] = E1000_RCTL_EN | // enable receiver
    E1000_RCTL_BAM |                 // enable broadcast
    E1000_RCTL_SZ_2048 |             // 2048-byte rx buffers
    E1000_RCTL_SECRC;                // strip CRC
  
  // ask e1000 for receive interrupts.
  regs[E1000_RDTR] = 0; // interrupt after every received packet (no timer)
  regs[E1000_RADV] = 0; // interrupt after every packet (no timer)
  regs[E1000_IMS] = (1 << 7); // RXDW -- Receiver Descriptor Write Back
}
```

## 3. 总结

本lab虽说是网卡驱动相关的lab，不过只关心了网卡驱动的核心处理逻辑，这部分需要阅读手册，学习了现代网卡（其实也比较老了）的处理方式(通过rx_ring和tx_ring), DMA处理等。也算不错的体验。

至此，MIT 6.s081的所有lab均已完成。休息调整，即将毕业了，祝自己毕业快乐。

