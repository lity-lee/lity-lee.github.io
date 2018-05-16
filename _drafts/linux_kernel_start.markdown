---
layout: post
category: linux
title:  "Linux内核启动流程"
tags: [android,linux,kernel]
---

系统调用， swi指令， 内核什么时候完成了系统调用，内核开始的时候都做了什么事情，这些事情的源代码都存放在哪里呢


<!-- more -->

由一个问题引起“Does the kernel have a main() function?”


In user space programs, main() is the entry point to the program that is called by the libc initialization code when the binary is executed. Kernel code does not have the luxury to rely on libc, as libc itself relies on the kernel syscall interface for memory allocation, I/O, process managements etc.

That said, the equivalent of main() in kernel code is start_kernel(), which is called by the bootloader after having loaded the kernel image, decompressed it into memory and setup essential hardware and memory paging. start_kernel() performs the majority of the system setup and eventually spawns the init process.

The entry point to Linux kernel modules is an init function that is registered with the kernel by calling the module_init() macro. The registered module init function is then called by kernel code through the do_initcalls() function during kernel startup.



#### 1. 原子操作

源处引用计数的思考。

实现i++, 调用了android_atomic_inc函数


ldrex 独占取值
strex 独占存值

不能独占指令不产生影响。


#### 2. 异常向量表

https://blog.csdn.net/haoge921026/article/details/46686525


ARM异常处理：

只要正常的程序流被暂时中止，处理器就进入异常模式。例如响应一个来自外设的中断。在处理异常之前，ARM内核保存当前的处理器状态，这样当处理程序结束是可以恢复执行原来的程序。

注意:如果同时发生两个或更多异常，那么将按照固定的顺序来处理异常 。

ARM支持的异常种类:

一、异常的进入与退出

当一种异常发生时，硬件就会自动执行如下动作:

(1)将CPSR保存到相应异常模式下的SPSR中

(2)把PC寄存器保存到相应异常模式下的LR中

(3)将CPSR设置成相应的异常模式

(4)设置PC寄存器的值为相应处理程序的入口地址

可以总结如下图:

呵呵，在这里我们需要重点研究的是，异常产生后，最后PC会指到哪里去呢?这个事情实际不需要我们操心，ARM核在设计的时候就已经确定好了，也就是经常我们所说的异常向量表。异常向量表:

在ARM7,ARM9/10等处理器，异常向量表可以存放在以 0x00000000或0xffff00000其始的地址。默认是以零地址开始存放的。可能有些同学还是有些晕，我们来举个例子说明一下。

例如:ARM处理器正在执行指令，此时外部硬件产生了一个中断。此时将产生IRQ异常，然后ARM核就会自动完成我们上面说的4步。完成前3步后，ARM核会强制将pc的值修改为0x18。修改完成后，处理器就开始从0x18这个地址取指令执行。

从上图可以知道，不同的异常产生时，ARM核修改的PC值不一样。例如:如果是swi指令引起的异常，ARM核最后就会修改pc的值为0x08。

是不是异常向量表，一定要放在0x00000000或0xffff00000其始的地址呢？答案:不是，现在cortex-A系列的处理器可以将异常向量表放在任何位置，拿ARM核收到异常后，它怎么知道应该将pc的值修改为多少呢?这就需要我们通过协处理指令告诉它了。

例如:在cortex-A8上，我们可以操作如下协处理指令，来告诉ARM核异常向量表的位置。

cortex-A8官方手册:3.2.68节有详细说明

如将告诉ARM核，异常向量表存放在0x20008000

ldr r0,=0x20008000

mcr p15,0,r0,c12,c0,0


#### 内核初始化


#### 中断向量表

http://www.cnblogs.com/getyoulove/p/3850370.html


ARM要求中断向量表必须放置在从0地址开始，连续8×4字节的空间内(ARM720T和ARM9、ARM10也支持从0xFFFF0000开始的高地址向量表)，各异常和中断向量在向量表中的位置如下


#### kernel init


start_kernel
setup_arch
paging_init
devicemaps_init
early_trap_init



#### 2. 线程锁


pthread_mutex_lock
pthread_mutex_lock_impl
_normal_lock
__bionic_cmpxchg

strexeq 独占工


#### 3. 线程的分支－等待

__futex_wait_ex
__futex_syscall4
__futex_syscall3(futex_arm.S)

#### 4. 系统调用

#### 5. kernel init

#### 6.  do_timer_interrupt

