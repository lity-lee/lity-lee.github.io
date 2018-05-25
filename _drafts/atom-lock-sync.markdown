---
layout: post
category: android
title:  "追根原子操作、锁、同步实现原理"
tags: [java,kernel,asm]
---

在多线程环境下，i++并不安全。使用两个线程同时操作i++ 1w万次，可能得出i的值并不是2w。想要安全的使用i++, 在java上可以使用synchronized关键字， c/c++语言中可以使用pthread_mutex_lock。 。c/c++的sleep函数也同样让我着迷，它可以让线程“暂停”，想不通它是怎么做到的。但感觉 它们应该是有共性的，抢占一个被占用的锁时是要出现“等待”的。

<!-- more -->



时候很神奇吧，我一直觉得
可一直对它们的实现很着迷。

站在使用的角度已经够了，可我很想知道它是怎么实现的嘛


以前学习编程时，想实现计数器count功能，了解到i++在多线程共享环境下是不安全的，



锁是一个软件层抽象出来的概念，想了解它的“基础”，是怎么实现的。

同步应该是java的概念

当年学习c语言时，一直不太理解i++是如何实现原子操作的





### 1. 原子操作i++

以前在维护过一个c/c++项目，它实现了一套的引用计数对象来管理内存。跟踪它在增加引用计数时调用了android_atomic_inc，然后又调用了android_atomic_add，android_atomic_add函数是内嵌汇编实现的：

```
extern ANDROID_ATOMIC_INLINE
int32_t android_atomic_add(int32_t increment, volatile int32_t *ptr)
{
    int32_t prev, tmp, status;
    android_memory_barrier();
    do {
        __asm__ __volatile__ ("ldrex %0, [%4]\n"
                              "add %1, %0, %5\n"
                              "strex %2, %1, [%4]"
                              : "=&r" (prev), "=&r" (tmp),
                                "=&r" (status), "+m" (*ptr)
                              : "r" (ptr), "Ir" (increment)
                              : "cc");
    } while (__builtin_expect(status != 0, 0));
    return prev;
}
```

第一眼看上去，肯定会怀疑代码是不是找错了，怎么会有一个循环呢？我们慢慢分析一下，依据c/c++调用汇编参数传递格式，可以列出下表：

|汇编参数|c/c++参数|
|%0|prev|
|%1|tmp|
|%2|status|
|%3|*ptr|
|%4|ptr|
|%5|increment|

依据这个表可以把上面汇编代码转换成伪代码：

```
    do {
        tmp = *ptr;
        tmp = tmp + increment;
        *ptr = tmp;
    } while (status != 0);
```

这是很好理解，关键是怎样达到原子操作（线程安全）呢， 就是ldrex和strex指令的功能了， 两条指令的官方文档：

![ldrex_doc](../assets/2018-05-25_ldrex_doc.png)

![ldrex_doc](../assets/2018-05-25_strex_doc.png)

主要看Operation栏。指令ldrex在取值时会在对应内存上做一个exclusive
access标记;指令strex存值时有这个标记就会存值成功，返回0，没有这个标记就会存值失败，返回1。这个标记就是独占式访问标记，是针对cpu的，多核环境下也没有问题。

### 2. sleep函数的暂停

sleep函数的实现会涉及到系统调用，这里我们只需要关注sleep用户态的代码和内核的代码即可，至于它们是怎样关联的，请参考[arm平台linux系统调用]({{ site.baseurl }}{% post_url 2018-05-11-arm-linux-kernel-syscall %})这篇文章。

sleep用户态代码相对简单，从sleep函数调到nanosleep标签：

```
unsigned int sleep(unsigned int seconds)
{
    struct timespec  t;

   /* seconds is unsigned, while t.tv_sec is signed
    * some people want to do sleep(UINT_MAX), so fake
    * support for it by only sleeping 2 billion seconds
    */
    if ((int)seconds < 0)
        seconds = 0x7fffffff;
        
    t.tv_sec  = seconds;
    t.tv_nsec = 0;

    if ( !nanosleep( &t, &t ) )
    return 0;

    if ( errno == EINTR )
        return t.tv_sec;

    return -1;
}
```

```
ENTRY(nanosleep)
    mov     ip, r7
    ldr     r7, =__NR_nanosleep
    swi     #0
    mov     r7, ip
    cmn     r0, #(MAX_ERRNO + 1)
    bxls    lr
    neg     r0, r0
    b       __set_errno
END(nanosleep)
```


sleep内核态代码：



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

#### synchronized 关键字

反编译后， 发现在 代码段前后插入 monitor-enter, monitor-exit

java代码在执行时是：解释执行， 可以找到对应代码（补充对应关系）


搜索HANDLE_OPCODE(OP_MONITOR_ENTER， 在OP_MONITOR_ENTER.cpp文件内
dvmLockObject





