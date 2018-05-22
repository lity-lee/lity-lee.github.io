---
layout: post
category: linux
title:  "Linux内核实现arm平台的硬中断"
tags: [arm,linux,kernel]
---

硬中断的概念很容易理解，在学习操作系统原理时会介绍的很清楚, 当然网上也有很多相关的文章。但我想知道在arm硬件平台上，linux内核是如何实现中断的呢？实现代码在linux内核代码的什么位置呢？带着这些问题，一步一步追踪，慢慢地对arm平台硬中断实现有一个较全面的认识，遂写下这篇文章。

<!-- more -->

### arm平台中断向量表

arm处理器在执行指令时，突然产生中断，它将会跳转到指定位置开始执行， 这个指定位置就叫中断向量。
中断向量存放的是跳转到中断处理程序入口的跳转指令。中断向量表就是所有中断向量的集合，在程序上可以认为是首地址，中断向量表也叫异常向量表，下文中将不会区分。

在ARM7,ARM9/10等处理器，中断异常向量表可以存放在以 0x00000000或0xffff00000其始的地址。默认是以零地址开始存放的。cortex-A系列的处理器可以将异常向量表放在任何位置，那ARM处理收到异常后，它怎么知道应该将pc的值修改为多少呢? 这就需要我们通过协处理指令告诉它了。

上面这段话的正确性, 可参考[***arm处理器官方文档***](http://infocenter.arm.com/help/index.jsp), 下面截图也来自官方文档。

ARM7异常向量表如下：

![arm7_exception_vectors.png](../assets/2018-05-22_arm7_exception_vectors.png)

cortex-A文档异常处理相关篇幅，2.15.12和3.2.68， 下面是部分截图：


![arm-cortexa8_exception_vectors](../assets/2018-05-22_arm-cortexa8_exception_vectors.png)

![arm-cortexa8_exception_register](../assets/2018-05-22_arm-cortexa8_exception_register.png)


在arm7处理器上，把内存地址0x00000000或者0xFFFF0000处复制成中断处理程序的跳转入口指令即可； contex-A处理器上，需要使用mcr指令设置相应的协处理器寄存器。

### 内核代码起点

上面我们知道了，arm处理器在产生中断时，会跳转到指定位置，这个指定位置应该存放的是跳转到异常处理程序入口的指令， 那么我们应该搜索内核代码找到设置指令的代码就吻合了。下面摘自网络上的一篇帖子，提示内核入口点。

```
Does the kernel have a main() function?


In user space programs, main() is the entry point to the program that is called by the libc initialization code when the binary is executed. Kernel code does not have the luxury to rely on libc, as libc itself relies on the kernel syscall interface for memory allocation, I/O, process managements etc.

That said, the equivalent of main() in kernel code is start_kernel(), which is called by the bootloader after having loaded the kernel image, decompressed it into memory and setup essential hardware and memory paging. start_kernel() performs the majority of the system setup and eventually spawns the init process.

The entry point to Linux kernel modules is an init function that is registered with the kernel by calling the module_init() macro. The registered module init function is then called by kernel code through the do_initcalls() function during kernel startup.
```

### 内核内码




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

