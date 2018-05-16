---
layout: post
category: linux
title:  "追根原子操作、锁、同步实现原理"
tags: [linux,kernel,asm]
---

以前在维护过一个c++项目，它实现了一套的引用计数对象来管理内存。跟踪它的代码发现它调用了android_atomic_inc， 而这个函数是用汇编实现的，由于当时对汇编一点不了解，就暂停了继续追根，现在又想

锁是一个软件层抽象出来的概念，想了解它的“基础”，是怎么实现的。

同步应该是java的概念

当年学习c语言时，一直不太理解i++是如何实现原子操作的

<!-- more -->



#### 1. 原子操作

源处引用计数的思考。

实现i++, 调用了android_atomic_inc函数


ldrex 独占取值
strex 独占存值

不能独占指令不产生影响。


#### 2. 线程锁分支-加锁


pthread_mutex_lock
pthread_mutex_lock_impl
_normal_lock
__bionic_cmpxchg

strexeq 独占工


#### 3. 线程的分支－等待

__futex_wait_ex
__futex_syscall4
__futex_syscall3(futex_arm.S)

#### synchronized 关键字

反编译后， 发现在 代码段前后插入 monitor-enter, monitor-exit

java代码在执行时是：解释执行， 可以找到对应代码（补充对应关系）


搜索HANDLE_OPCODE(OP_MONITOR_ENTER， 在OP_MONITOR_ENTER.cpp文件内
dvmLockObject





