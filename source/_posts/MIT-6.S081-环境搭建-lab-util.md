---
title: MIT6.S081-环境搭建-lab-util
categories: MIT-6.S081
date: 2022-05-03 22:10:31
tags:
---



## 0. 前言

虽然MIT6824（分布式）还剩最难的一个大lab未做，不过我还是决定从本篇开始新的学习---操作系统6.S081，原计划为6.828，由于没有系统学过汇编，6828的第一个bootloader实验需要耗费的时间过多，故而改为6.S081，除了整体更简单外，采用risc-v也为后续C61C做点铺垫。

好，本篇为6.S081-2020的起始篇章。主要讲讲环境搭建和第一个lab。

<!--more-->

## 1. 环境搭建

环境搭建结论：如果是Ubuntu系统，一定用Ubuntu20.04. 否则可能踩不少坑。

最开始我用虚拟机ubuntu18.04，原因`qemu-system-misc=1:4.2-3ubuntu6` 这个包是找不到的。 而这个在ubuntu20.04中可以轻易下载。对于18.04可能可以从源码安装，不过qemu相对复杂，我选择放弃，毕竟我还有ubuntu20.04的wsl2. 

> 官网的搭建教程：https://pdos.csail.mit.edu/6.828/2020/tools.html

安装依赖包：

```shell
sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu 
sudo apt-get remove qemu-system-misc
sudo apt-get install qemu-system-misc=1:4.2-3ubuntu6
```

拉下官网仓库：

```
git clone git://g.csail.mit.edu/xv6-labs-2020
git checkout util
make qemu
```

如果出现以下字样，则搭建成功：

```
$ make qemu
riscv64-unknown-elf-gcc    -c -o kernel/entry.o kernel/entry.S
riscv64-unknown-elf-gcc -Wall -Werror -O -fno-omit-frame-pointer -ggdb -DSOL_UTIL -MD -mcmodel=medany -ffreestanding -fno-common -nostdlib -mno-relax -I. -fno-stack-protector -fno-pie -no-pie   -c -o kernel/start.o kernel/start.c
...  
riscv64-unknown-elf-ld -z max-page-size=4096 -N -e main -Ttext 0 -o user/_zombie user/zombie.o user/ulib.o user/usys.o user/printf.o user/umalloc.o
riscv64-unknown-elf-objdump -S user/_zombie > user/zombie.asm
riscv64-unknown-elf-objdump -t user/_zombie | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$/d' > user/zombie.sym
mkfs/mkfs fs.img README  user/xargstest.sh user/_cat user/_echo user/_forktest user/_grep user/_init user/_kill user/_ln user/_ls user/_mkdir user/_rm user/_sh user/_stressfs user/_usertests user/_grind user/_wc user/_zombie 
nmeta 46 (boot, super, log blocks 30 inode blocks 13, bitmap blocks 1) blocks 954 total 1000
balloc: first 591 blocks have been allocated
balloc: write bitmap block at sector 45
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$
```

如何退出qemu？

**ctrl+A+x.**

#### 如何调试？

开启两个窗口：

窗口1：

```
make CPUS=1 qemu-gdb		// 单核运行
```

窗口2:

```
gdb-multiarch			// 一定要用gdb-multiarch, 否则无法挂上去
```

## 2. lab util

第一个lab几乎就是用来熟悉环境的。包含如下几个tasks：

1. 实现一个user space的sleep program。 接收用户一个参数，表示要睡眠几个ticker。
2. pingpong，使用管道，在父子进程间来回传递一个byte。
3. primes。这个非常有趣，使用多个进程流水线式的计算素数。如下图：

![img](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/sieve.gif)

第一个进程排除所有可以被2整除的数。其他数传递给下一个进程，下一个进程排除所有可以被3整除的数，其余数传递给下一个进程，依次类推。

1. find。实现简化版本的unix find. 
2. xargs。实现简化版本的xargs。

