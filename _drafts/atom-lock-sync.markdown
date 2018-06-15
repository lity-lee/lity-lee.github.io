---
layout: post
category: android
title:  "追根原子操作、锁、同步实现原理"
tags: [java,kernel,asm]
---

在多线程环境下，i++并不安全。使用两个线程同时操作i++ 1w万次，可能得出i的值并不是2w。想要安全的使用i++, 在java上可以使用synchronized关键字， c/c++语言中可以使用pthread_mutex_lock。 。c/c++的sleep函数也同样让我着迷，它可以让线程“暂停”，想不通它是怎么做到的。但感觉 它们应该是有共性的，抢占一个被占用的锁时是要出现“等待”的。

<!-- more -->

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

```
SYSCALL_DEFINE2(nanosleep, struct timespec __user *, rqtp,
        struct timespec __user *, rmtp)
{
    struct timespec tu;

    if (copy_from_user(&tu, rqtp, sizeof(tu)))
        return -EFAULT;

    if (!timespec_valid(&tu))
        return -EINVAL;

    return hrtimer_nanosleep(&tu, rmtp, HRTIMER_MODE_REL, CLOCK_MONOTONIC);
}
```

完整的调用栈如下：

![sleep_kernel](../assets/2018-06-14_sleep_kernel.png)

最终调到context_switch, context_switch函数的调用会引起进程调度，就是cpu切换到其它进程去执行了，当前进程就“失去”cpu了，这里会涉及时间片的概念。所以sleep函数是不占用cpu资源的，对调用者来说它是“耗时”。我们知道sleep函数在“暂停”指定时间后会继续执行后面的代码，现在cpu切换到其它进程去执行了，那cpu什么时候再切换回来呢！

### 2. 线程锁

c/c++ 线程锁pthread_mutex_lock， 可以达到同步的效果。而锁是软件层抽象出来的概念，它是如何实现的。在使用者的角度可以分为两部分，一是锁状态的更改; 二是获取释放锁的过程。


先说第一部，锁状态的更改。下面是调用栈：

```
pthread_mutex_lock
pthread_mutex_lock_impl
__bionic_cmpxchg
```

__bionic_cmpxchg函数在bionic_atomic_arm.h文件中，实现代码：

```
/* Compare-and-swap, without any explicit barriers. Note that this functions
 * returns 0 on success, and 1 on failure. The opposite convention is typically
 * used on other platforms.
 */
__ATOMIC_INLINE__ int
__bionic_cmpxchg(int32_t old_value, int32_t new_value, volatile int32_t* ptr)
{
    int32_t prev, status;
    do {
        __asm__ __volatile__ (
            __ATOMIC_SWITCH_TO_ARM
            "ldrex %0, [%3]\n"
            "mov %1, #0\n"
            "teq %0, %4\n"
#ifdef __thumb2__
            "it eq\n"
#endif
            "strexeq %1, %5, [%3]"
            __ATOMIC_SWITCH_TO_THUMB
            : "=&r" (prev), "=&r" (status), "+m"(*ptr)
            : "r" (ptr), "Ir" (old_value), "r" (new_value)
            : __ATOMIC_CLOBBERS "cc");
    } while (__builtin_expect(status != 0, 0));
    return prev != old_value;
}
```

这段代码与上面原子操作i++部分相似，这里不多解释了，核心还是ldrex和strex, 独占式读写操作。

第二部分，锁释放的等待。这将分为两阶段，一是用户态态部分，二是内核态部分。用户态部分的调用栈：

```
pthread_mutex_lock
pthread_mutex_lock_impl
__futex_wait_ex
__futex_syscall4
__futex_syscall3
```

摘出主要代码：
```
// __futex_syscall3(*ftx, op, val)
ENTRY(__futex_syscall3)
    mov     ip, r7
    ldr     r7, =__NR_futex
    swi     #0
    mov     r7, ip
    bx      lr
END(__futex_syscall3)

// __futex_syscall4(*ftx, op, val, *timespec)
ENTRY(__futex_syscall4)
    b __futex_syscall3
END(__futex_syscall4)
```

可以知道最终调用的系统调用是futex, 但使用[arm平台linux系统调用]({{ site.baseurl }}{% post_url 2018-05-11-arm-linux-kernel-syscall %})提供的方法找内核代码，这里的参数个数不太明确，但模糊查找可以确定是SYSCALL_DEFINE6(futex。内核态栈调用图：

![futex_kernel](../assets/2018-06-15_futex_kernel.png)

是不是第消息5就很熟悉啦，和上面sleep内核态代码相同，最终都调用到了context_switch, 引用进程调度，不“占用”cpu。

### synchronized 关键字

既然我们了解了c/c++的锁是如何实现的，那么java的锁呢？这里主要探讨java关键字synchronized。

反编译后， 发现在 代码段前后插入 monitor-enter, monitor-exit

java代码在执行时是：解释执行， 可以找到对应代码（补充对应关系）


搜索HANDLE_OPCODE(OP_MONITOR_ENTER， 在OP_MONITOR_ENTER.cpp文件内
dvmLockObject





