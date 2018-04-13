---
layout: post
category: linux
title:  "追根原子操作、锁、同步实现原理"
tags: [android,unity3d,c#/java/native]
---

在Android平台上开发unity3d游戏，有三种不同的编程语言java, c/c++, c#，有时免不了三种语言之前的互相调用。比如现在有一个已经完成某一功能的SDK, 游戏开发者需要调用此SDK，SDK被调用后完成功能后传递结果到C#，表面上看只涉及c#调用java, java调用c#, 并没有c/c++语言什么事儿，然而好像游戏开发人员需要调用java语言写好的SDK, java
很久以前我就想学习一下汇编语言，或者写个hello world什么的，但发现很难“找齐”一个完整的开发流程，后来也是在无意中（自己多年坚持下）找到的。在这个环境下，你可以尽情地“玩耍”，像学习c/c++语言一样来快速地学习汇编，c/c++语言这样的学习环境是很容易找到的，而汇编语言的就不那么容易了。所以这篇仅于记录，共勉有同等需求的人。

<!-- more -->

* 操作系统：linux(ubuntu)
* 编辑器：sublime (nasm 插件)
* 汇编器：nasm
* 链接器：ld
* 调试工具：gdb

#### 1. 原子操作

源处引用计数的思考。

实现i++, 调用了android_atomic_inc函数


ldrex 独占取值
strex 独占存值

不能独占指令不产生影响。





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

系统调用





sublime及sublime nasm 插件安装完毕后，可创建汇编文件进行编辑，且有“智能提示”功能，如下图所示

![sublime_edit](../assets/2018-02-18_sublime_edit.png)


写好汇编程序，源代码存放在一个叫hello.asm的文件中，它是一个32位的程序，那么怎样汇编、链接、运行、调试呢？

```
nasm -f elf32 hello.asm -g
ld -m elf_i386 hello.o
./a.out
gdb a.out
```

上面的4条命令分别对应汇编、链接、运行、调试。使用gdb调试如下

![nasm_gdb](../assets/2018-02-18_nasm_gdb.png)

反汇编之后与源代码一致。

#### 3. cxx调用c# 



#### 4. cxx调用java

#### 5. java调用c# 

#### 6. java调用c# 

程序码代码

```nasm
BITS 32
section .data
		msg db "Hello, world!", 0xA
		len equ $ - msg
section .text
global _start
_start:
        mov edx, len;
		mov ecx, msg;
		mov ebx, 1;
		mov eax, 4;
		int 0x80;

		mov ebx, 0;
		mov eax, 1;
		int 0x80;
```

这里是两个系统调用，分别是向1文件中(标准输出流)写内容和调用exit来退出程序。
看这个网站就会明白为什么这样写或者还可以调用更多的linux系统调用，[***http://syscalls.kernelgrok.com/***](http://syscalls.kernelgrok.com/)。

