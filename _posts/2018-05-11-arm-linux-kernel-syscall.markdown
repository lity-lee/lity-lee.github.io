---
layout: post
category: linux
title:  "arm平台linux系统调用"
tags: [kernel,arm,syscall]
---


系统调用就是应用程序和内核交互的“桥梁”。系统调用的左边是应用程序，运行在用户态；右边是内核，运行在内核态。linux在arm平台上实现系统调用是通过软中断的方式，即：swi指令。linux提供多种系统调用完成不同的功能，相当于“桥梁”之上有不同的“车道”，那么这些“车道”是怎么联通的呢？有没有方法快速地找到用户态代码对应调用的内核态代码呢？

<!-- more -->


### 写在前面

文章参考的Android源码版本：4.4。

文章参考的内核源码是goldfish平台的，版本：android-3.18。我是从这里下载的

```
git clone https://android.googlesource.com/kernel/goldfish
git checkout android-3.18
```

### linux系统调用

什么是系统调用，前面我已经按照自己的理解介绍了一遍，如果还不是很理解，可以自行搜索。

linux 平台提供了哪些系统调用

* 方法1 可以查看这个头文件 /bionic/libc/kernel/arch-arm/asm/unistd.h
* 方法2 访问这个网站[***http://syscalls.kernelgrok.com/***](http://syscalls.kernelgrok.com/)

虽然方法2好像不是针对arm平台的(应该是x86), 但它在系统调用整体上给你一个大概的认识。


### 一个系统调用从用户态到内核态的对应

* 从用户态api查起
* 确定系统调用的名称及参数个数
* 查找内核态代码以 “SYSCALL_DEFINE{参数个数}(系统调用名称”为关键字查找

软中断swi可以认为它是“异步”调用，从用户态突然到内核态，再返回到用户态。

参数个数和系统调用名称的确定，将在下面以举例说明。


#### 例子，用户态sleep函数

通过关键字查找sleep, 找到sleep的定义

![sleep_define](../assets/2018-05-10_sleep_define.png)

在45行调用了nanosleep，查找nanosleep的定义

![nanosleep_define](../assets/2018-05-10_nanosleep_define.png)


由此可以确实系统调用的名称是： nanosleep， 参数个数是： 2

在内核代码中查找的关键字应该是： SYSCALL_DEFINE2(nanosleep

![nanosleep_search](../assets/2018-05-10_nanosleep_search.png)

#### 例子，用户态exit函数


通过关键字查找exit, 找到exit的定义

![exit_define](../assets/2018-05-10_exit_define.png)

在58行调用了_exit，查找_exit的定义

![exit_group_define](../assets/2018-05-10_exit_group_define.png)


由此可以确实系统调用的名称是： exit_group， 参数个数是： 1

在内核代码中查找的关键字应该是： SYSCALL_DEFINE1(exit_group

![exit_group_search](../assets/2018-05-10_exit_group_search.png)

#### 例子， 用户态open函数

通过关键字查找open, 找到open的定义

![open_define](../assets/2018-05-10_open_define.png)

在51行调用了 __open，查找 __open的定义

![__open_define](../assets/2018-05-10__open_define.png)


由此可以确实系统调用的名称是： open， 参数个数是： 3

在内核代码中查找的关键字应该是： SYSCALL_DEFINE3(open

![exit_group_search](../assets/2018-05-10__open_search.png)


### 用户态和内核态代码的联通

上面我提供了一个方法，可以快速地找到一个系统调用在内核态的实现代码。但却没有言明为什么是这样的，
它们的“联通”代码在哪里，下面将会讲述。

从分析中断系统和中断向量表，我们可以知道软中断swi的中断处理处理程序开始于： vector_swi。然而这个结论的分析过程并不在本文中，以后我可能会写，暂时没有链接。

vector_swi的定义在entry-common.S文件中，此文件include calls.S文件。所以“联通”的代码主要分布在entry-common.S和calls.S两个文件中。

#### vector_swi调用sys_*

我们可以理解成vector_swi是软中断swi内核代码执行的源头，且Linux在arm平台上的系统调用全部是用软中断swi实现的，就可是认为vector_swi是所有系统调用的源头。源头再依据系统调用号来“分发”到某个具体系统调用的实现方法， 系统调用的实现方法即是：sys_系统调用名称（统称为sys_*）。

1、分析vector_swi标签函数(截出主要部分)

![vector_swi](../assets/2018-05-11_vector_swi.png)

变量scno是系统调用号， 关键行413， 439。分析一下得出： 从变量sys_call_table开始，依据系统调用号scno的值向后做偏移。然后跳转到对应的地址。所以sys_call_table很重要。

2、sys_call_table的定义

![sys_call_table_define](../assets/2018-05-11_sys_call_table_define.png)

![calls_content](../assets/2018-05-11_calls_content.png)

![call_defile](../assets/2018-05-11_call_defile.png)

第一张图和第三张图取自entry-common.S文件， 第二张图取自calls.S， 所以我们看到entry-common.S文件两次include calls.S文件， 第一次是定义NR_syscalls, 用于统计系统调用的总数； 第二次是定义sys_call_table。

综述我们可以得出，vector_swi call sys_* 在内核态实现某一个系统调用的功能。

#### sys_*在哪里

如果直接搜索内核态系统调用的实现函数，比如sys_open, 会什么也搜索不到。

![sys_open_search](../assets/2018-05-11_sys_open_search.png)

原因是代码在编写的时候使用了宏定义“魔术”，导致直接搜索关键字找不到。在syscalls.h文件中，我们可以找到很多类似SYSCALL_DEFINE的宏。

![syscall_define](../assets/2018-05-11_syscall_define.png)

通过宏展开，我们可以得出源代码中“SYSCALL_DEFINE{参数个数}(系统调用名称”在预编译后生成的函数名就是“sys_系统调用名称”。至此，也验证了上述提供的查找方法是正确的。







