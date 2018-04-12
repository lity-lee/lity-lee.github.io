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



