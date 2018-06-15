---
layout: post
category: android
title:  "追根原子操作、锁、同步实现原理"
tags: [java,kernel,asm]
---

在多线程环境下，i++并不安全。使用两个线程同时操作i++ 1w万次，可能得出i的值并不是2w。想要安全的使用i++, 在java上可以使用synchronized关键字， c/c++语言中可以使用pthread_mutex_lock。c的sleep函数也同样让我着迷，它可以让线程“暂停”，想不通它是怎么做到的。但感觉 它们应该是有共性的，抢占一个被占用的锁时是要出现“等待”的。

<!-- more -->

### 1. 原子操作i++

以前维护过一个c/c++项目，它实现了一套的引用计数对象来管理内存。跟踪它在增加引用计数时调用了android_atomic_inc，android_atomic_inc又调用了android_atomic_add，android_atomic_add函数是内嵌汇编实现的：

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

这很好理解，关键是怎样达到原子操作（线程安全）呢，就是ldrex和strex指令的功能了，两条指令的官方文档：

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

### 3. 线程锁

c/c++线程锁pthread_mutex_lock，可以达到同步的效果。而锁是软件层抽象出来的概念，它是如何实现的。在使用者的角度可以分为两段，第一段是锁状态的更改;第二段是等待锁释放的过程。

先说第一段，锁状态的更改。下面是调用栈：

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

第二段，等待锁的释放。这部分是使用系统调用实现的，代码有用户态部分和内核态部分。用户态部分的调用栈：

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

是不是从消息5开始就很熟悉啦，和上面sleep内核态代码相同，最终都调用到了context_switch, 引起进程调度。

### synchronized 关键字

既然我们了解了c的锁是如何实现的，那么java的锁呢？这里主要探讨java关键字synchronized。

先看一段java代码（摘自MobLink):

```
    void updateIntent(Activity activity, Intent intent) {
        /*
         * 1. 检查intent是否存在场景/是否需要从服务器恢复场景
         * 2. 检查缓存intent是否相同
         * 3. 检查之前的msg_id是否存在
         */
        if ((!isNeedToUpdateIntent(intent)) && !isNeedRestoreSceneFromServer) {
            return;
        }

        IntentRecorder ir;
        synchronized (cacheIntent) {
            if (cacheIntent.isSame(activity, intent)) {
                return;
            }

            // 先缓存一下，防止多次调用，导致多个场景
            cacheIntent.activity = activity;
            if (null == intent) {
                cacheIntent.intent = new Intent();
            } else {
                cacheIntent.intent = intent;
            }
            ir = new IntentRecorder(cacheIntent);
        }

        // 如果存在id, 需要修改参数
        if (thisHander.hasMessages(MSG_PRE_RESTORE_SCENE)) {
            thisHander.removeMessages(MSG_PRE_RESTORE_SCENE);
        }

        Message msg = thisHander.obtainMessage(MSG_PRE_RESTORE_SCENE, ir);
        thisHander.sendMessage(msg);
    }
```

这个函数可以被多个线程调用，调用之后把传过来的activity、intent对象缓存在cacheIntent对象里，然后在另外一个地方使用缓存在cacheIntent对象里的activity和intent，activity和intent是“成对”的，为了防止读取时又有写入导致的activity和intent不成对，所以加了一个synchronized关键字。

编译成class后，使用d2j-jar2dex.bat和d2j-dex2smali.bat反编译生成smali代码，如下：

```
.method updateIntent(Landroid/app/Activity;Landroid/content/Intent;)V
    .catchall { :L2 .. :L4 } :L3
    .catchall { :L5 .. :L7 } :L3
    .catchall { :L9 .. :L10 } :L3
    .registers 9
    .prologue
    const/16 v5, 1001
    .line 209
    invoke-direct { p0, p2 }, Lcom/mob/moblink/utils/MobLinkImpl;->isNeedToUpdateIntent(Landroid/content/Intent;)Z
    move-result v2
    if-nez v2, :L1
    iget-boolean v2, p0, Lcom/mob/moblink/utils/MobLinkImpl;->isNeedRestoreSceneFromServer:Z
    if-nez v2, :L1
    :L0
    .line 236
    return-void
    :L1
    .line 214
    iget-object v3, p0, Lcom/mob/moblink/utils/MobLinkImpl;->cacheIntent:Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;
    monitor-enter v3
    :L2
    .line 215
    iget-object v2, p0, Lcom/mob/moblink/utils/MobLinkImpl;->cacheIntent:Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;
    invoke-virtual { v2, p1, p2 }, Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;->isSame(Landroid/app/Activity;Landroid/content/Intent;)Z
    move-result v2
    if-eqz v2, :L5
    .line 216
    monitor-exit v3
    goto :L0
    :L3
    .line 227
    move-exception v2
    monitor-exit v3
    :L4
    throw v2
    :L5
    .line 220
    iget-object v2, p0, Lcom/mob/moblink/utils/MobLinkImpl;->cacheIntent:Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;
    invoke-static { v2, p1 }, Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;->access$202(Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;Landroid/app/Activity;)Landroid/app/Activity;
    .line 221
    if-nez p2, :L9
    .line 222
    iget-object v2, p0, Lcom/mob/moblink/utils/MobLinkImpl;->cacheIntent:Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;
    new-instance v4, Landroid/content/Intent;
    invoke-direct { v4 }, Landroid/content/Intent;-><init>()V
    invoke-static { v2, v4 }, Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;->access$302(Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;Landroid/content/Intent;)Landroid/content/Intent;
    :L6
    .line 226
    new-instance v0, Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;
    iget-object v2, p0, Lcom/mob/moblink/utils/MobLinkImpl;->cacheIntent:Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;
    invoke-direct { v0, p0, v2 }, Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;-><init>(Lcom/mob/moblink/utils/MobLinkImpl;Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;)V
    .line 227
    .local v0, ir:Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;
    monitor-exit v3
    :L7
    .line 230
    iget-object v2, p0, Lcom/mob/moblink/utils/MobLinkImpl;->thisHander:Landroid/os/Handler;
    invoke-virtual { v2, v5 }, Landroid/os/Handler;->hasMessages(I)Z
    move-result v2
    if-eqz v2, :L8
    .line 231
    iget-object v2, p0, Lcom/mob/moblink/utils/MobLinkImpl;->thisHander:Landroid/os/Handler;
    invoke-virtual { v2, v5 }, Landroid/os/Handler;->removeMessages(I)V
    :L8
    .line 234
    iget-object v2, p0, Lcom/mob/moblink/utils/MobLinkImpl;->thisHander:Landroid/os/Handler;
    invoke-virtual { v2, v5, v0 }, Landroid/os/Handler;->obtainMessage(ILjava/lang/Object;)Landroid/os/Message;
    move-result-object v1
    .line 235
    .local v1, msg:Landroid/os/Message;
    iget-object v2, p0, Lcom/mob/moblink/utils/MobLinkImpl;->thisHander:Landroid/os/Handler;
    invoke-virtual { v2, v1 }, Landroid/os/Handler;->sendMessage(Landroid/os/Message;)Z
    goto :L0
    :L9
    .line 224
    .end local v0
    .end local v1
    iget-object v2, p0, Lcom/mob/moblink/utils/MobLinkImpl;->cacheIntent:Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;
    invoke-static { v2, p2 }, Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;->access$302(Lcom/mob/moblink/utils/MobLinkImpl$IntentRecorder;Landroid/content/Intent;)Landroid/content/Intent;
    :L10
    goto :L6
.end method
```

查看smali代码发现synchronized代码段前后插入monitor-enter, monitor-exit指令，在.line 214、.line 216、.line 227处。

之后严谨的思路应该是dalvik怎样解释monitor-enter和monitor-exit指令，但这会有很多东西要说，且不是本文描述的重点。我们直接从delvik解释字节码时，遇到monitor-enter和monitor-exit指令做了什么事情开始说起，其实每条字节码指令都会对应一个c/cpp函数，delvik遇到这个字节码时就会执行这个函数。这些指令每条分别对应一个cpp文件， 这些文件存放在“/dalvik/vm/mterp/c/”目录下。查看一下monitor-enter指令对应的实现函数：

```
HANDLE_OPCODE(OP_MONITOR_ENTER /*vAA*/)
    {
        Object* obj;

        vsrc1 = INST_AA(inst);
        ILOGV("|monitor-enter v%d %s(0x%08x)",
            vsrc1, kSpacing+6, GET_REGISTER(vsrc1));
        obj = (Object*)GET_REGISTER(vsrc1);
        if (!checkForNullExportPC(obj, fp, pc))
            GOTO_exceptionThrown();
        ILOGV("+ locking %p %s", obj, obj->clazz->descriptor);
        EXPORT_PC();    /* need for precise GC */
        dvmLockObject(self, obj);
    }
    FINISH(1);
OP_END
```

追踪一下dvmLockObject函数。这里同样像c线程锁一样分为两段，第一是锁状态的更改，调用栈：

```
dvmLockObject
android_atomic_acquire_cas
android_atomic_cas
```

android_atomic_cas函数的实现代码：

```
extern ANDROID_ATOMIC_INLINE
int android_atomic_cas(int32_t old_value, int32_t new_value,
                       volatile int32_t *ptr)
{
    int32_t prev, status;
    do {
        __asm__ __volatile__ ("ldrex %0, [%3]\n"
                              "mov %1, #0\n"
                              "teq %0, %4\n"
#ifdef __thumb2__
                              "it eq\n"
#endif
                              "strexeq %1, %5, [%3]"
                              : "=&r" (prev), "=&r" (status), "+m"(*ptr)
                              : "r" (ptr), "Ir" (old_value), "r" (new_value)
                              : "cc");
    } while (__builtin_expect(status != 0, 0));
    return prev != old_value;
}
```

看到这里，是不是很熟悉。和上面原子操作及c线程锁一样，他们都是使用ldrex和strex来完成的，独占式读写arm指令。

第二段是锁被其它线程占用了，当然线程会等待锁的释放，追踪代码调用栈：

```
dvmLockObject
nanosleep
```

dvmLockObject函数代码太长，分析起来也很费劲，这里摘出主要实现代码截图，其实也包括第一段的代码：

![sync_dvm_lock](../assets/2018-06-15_sync_dvm_lock.png)

这里是不是也很熟悉，因为它调用了nanosleep系统调用，和sleep代码的实现是一样的。

以前很不理解原了操作、线程锁、等待，不同的语言里更是千差万别，但追根究底之后发现原来他们是“一样的”，万变不离其宗吧。