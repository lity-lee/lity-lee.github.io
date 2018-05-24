---
layout: post
category: linux
title:  "Linux内核实现arm平台的硬中断"
tags: [arm,linux,kernel]
---

硬中断的概念很容易理解，在学习操作系统原理时会介绍的很清楚, 当然网上也有很多相关的文章。但我想知道在arm硬件平台上，linux内核是如何实现中断的呢？实现代码在linux内核代码的什么位置呢？带着这些问题，一步一步追踪，慢慢地对arm平台硬中断实现有一个较全面的认识，遂写下这篇文章。

<!-- more -->

### arm平台中断向量表

arm处理器在执行指令时，突然产生了中断，它将会跳转到指定位置开始执行， 这个指定位置就叫中断向量。
中断向量存放的是跳转到中断处理程序入口的跳转指令。中断向量表就是所有中断向量的集合，在程序上可以认为是首地址，中断向量表也叫异常向量表，下文中将不做区分。

在ARM7,ARM9/10等处理器，中断异常向量表可以存放在以 0x00000000或0xffff00000其始的地址。默认是以零地址开始存放的；cortex-A系列的处理器可以将异常向量表放在任何位置，那处理器收到异常后，它怎么知道应该将pc的值修改为多少呢? 这就需要我们通过协处理指令告诉它了。

上面这段话的正确性如何验证呢，当然最权威的莫过于官方文档了。参考[***arm处理器官方文档***](http://infocenter.arm.com/help/index.jsp), 下面截图也来自官方文档。

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

从start_kernel开始查找与异常相关的代码，其调用栈图如下：

![kernel_exception_stack](../assets/2018-05-22_kernel_exception_stack.png)

early_trap_init函数的实现

![early_trap_init](../assets/2018-05-22_early_trap_init.png)

从885,886行，可以看到是内存复制, “__vectors_start”就是硬中断向量表，在entry-armv.S中定义:

![_vectors_start](../assets/2018-05-22_vectors_start.png)

所以以后，查看硬中断处理程序，都可以从这里开始，“__vectors_start”。另外886行复制了“__stubs_start”，　“__stubs_start”也定义在entry-armv.S中：

![kernel_exception_swi](../assets/2018-05-22_kernel_exception_swi.png)

结合"____vectors_start"的定义，可以得出"vector_swi"是软中断swi指令的处理程序。arm平台linux系统调用都是通过软中断指令实现的，那么“vector_swi”也就是系统调用内核态代码的入口了。

### arm平台(at91)时钟中断

#### 1.irq中断处理过程

从上面可以知道可以知道irq中断的中断处理程序入口是: vector_irq。展开宏vector_stub，可以看出内核态中断处理程序下一步将执行: __irq_svc。之后一步步分发到中断处理程序，不一一赘述，看下图：

![irq_exception_callback](../assets/2018-05-24_irq_exception_callback.png)

第8个消息“generic_handle_irq”的实现代码：

```
int generic_handle_irq(unsigned int irq)
{
    struct irq_desc *desc = irq_to_desc(irq);

    if (!desc)
        return -EINVAL;
    generic_handle_irq_desc(irq, desc);
    return 0;
}
```

第9个消息“generic_handle_irq_desc”

```
static inline void generic_handle_irq_desc(unsigned int irq, struct irq_desc *desc)
{
    desc->handle_irq(irq, desc);
}
```

代码大概含义是：通过中断号找到中断描述信息irq_desc类型的指针，然后调用其handle_irq函数。结构体irq_desc里有一个action字段，它是一个链表，记录着中断回调列表，注释上写“IRQ action list”。那么我们基本上可以猜到，注册中断处理程序的过程也就是向action字段中增加节点了。

#### 2.注册时钟中断

从start_kernel找起，调用图如下：

![register_irq](../assets/2018-05-24_register_irq.png)

第4个消息“init_time”，函数代码：

```
void __init time_init(void)
{
    if (machine_desc->init_time) {
        machine_desc->init_time();
    } else {
#ifdef CONFIG_COMMON_CLK
        of_clk_init(NULL);
#endif
        clocksource_of_init();
    }
}
```

可以知道它最终是调用machine_desc->init_time()来注册时钟中断的。那么应该先找到machine_desc在哪里初化的，依然在上面的调用图中，第3个消息返回的值就是machine_desc。setup_machine_tags函数的部分代码：

```
const struct machine_desc * __init
setup_machine_tags(phys_addr_t __atags_pointer, unsigned int machine_nr)
{
    struct tag *tags = (struct tag *)&default_tags;
    const struct machine_desc *mdesc = NULL, *p;
    char *from = default_command_line;

    default_tags.mem.start = PHYS_OFFSET;

    /*
     * locate machine in the list of supported machines.
     */
    for_each_machine_desc(p)
        if (machine_nr == p->nr) {
            printk("Machine: %s\n", p->name);
            mdesc = p;
            break;
        }

    // do something...

    return mdesc;
}
```

看到代码中的宏for_each_machine_desc， 找出其定义：

```
/*
 * Machine type table - also only accessible during boot
 */
extern const struct machine_desc __arch_info_begin[], __arch_info_end[];
#define for_each_machine_desc(p)            \
    for (p = __arch_info_begin; p < __arch_info_end; p++)

/*
 * Set of macros to define architecture features.  This is built into
 * a table by the linker.
 */
#define MACHINE_START(_type,_name)          \
static const struct machine_desc __mach_desc_##_type    \
 __used                         \
 __attribute__((__section__(".arch.info.init"))) = {    \
    .nr     = MACH_TYPE_##_type,        \
    .name       = _name,

#define MACHINE_END             \
};
```

可以明白，machine_desc就是通过参数machine_nr在__arch_info_begin到__arch_info_end遍历中查找，那么__arch_info_begin在哪里呢？ 在vmlinux.lds.S文件里:

![arch_info_begin](../assets/2018-05-24_arch_info_begin.png)

可以看出在编译时把标识为.arch.info.init的section定义全放在__arch_info_begin和__arch_info_end之间，这里就包括mach-at91/board-eb01.c文件中的定义：

```
MACHINE_START(AT91EB01, "Atmel AT91 EB01")
    /* Maintainer: Greg Ungerer <gerg@snapgear.com> */
    .init_time  = at91x40_timer_init,
    .handle_irq = at91_aic_handle_irq,
    .init_early = at91eb01_init_early,
    .init_irq   = at91eb01_init_irq,
MACHINE_END
```

MACHINE_START宏在上面已经看到了，展开它就明了， 在arm at91处理器上machine_desc就是这时定义的结构体。串联起来，可以得出注册时钟中断的代码machine_desc->init_time()的执行就是执行at91x40_timer_init函数。下面再把调用图完整绘制一遍：

![register_irq_all](../assets/2018-05-24_register_irq_all.png)

第7个消息“__setup_irq”的函数实现代码：
```
/*
 * Internal function to register an irqaction - typically used to
 * allocate special interrupts that are part of the architecture.
 */
static int
__setup_irq(unsigned int irq, struct irq_desc *desc, struct irqaction *new)
{
    struct irqaction *old, **old_ptr;
    unsigned long flags, thread_mask = 0;
    int ret, nested, shared = 0;
    cpumask_var_t mask;

    // do something...
    /*
     * The following block of code has to be executed atomically
     */
    raw_spin_lock_irqsave(&desc->lock, flags);
    old_ptr = &desc->action;
    old = *old_ptr;
    if (old) {
        /*
         * Can't share interrupts unless both agree to and are
         * the same type (level, edge, polarity). So both flag
         * fields must have IRQF_SHARED set and the bits which
         * set the trigger type must match. Also all must
         * agree on ONESHOT.
         */
        if (!((old->flags & new->flags) & IRQF_SHARED) ||
            ((old->flags ^ new->flags) & IRQF_TRIGGER_MASK) ||
            ((old->flags ^ new->flags) & IRQF_ONESHOT))
            goto mismatch;

        /* All handlers must agree on per-cpuness */
        if ((old->flags & IRQF_PERCPU) !=
            (new->flags & IRQF_PERCPU))
            goto mismatch;

        /* add new interrupt at end of irq queue */
        do {
            /*
             * Or all existing action->thread_mask bits,
             * so we can find the next zero bit for this
             * new action.
             */
            thread_mask |= old->thread_mask;
            old_ptr = &old->next;
            old = *old_ptr;
        } while (old);
        shared = 1;
    }

    // do something...
    return ret;
}
```

可以看出它就是向中断回调列表中追加回调。至此，第1部分中分析的中断回调过程，和第2部分时钟中断处理程序的注册过程相互关连、印证。时钟中断有一个重要的工作，就是进程调度，会在下面讲述。

#### 3.时钟中断引起进程 调度

看第2部分注册过程中第5个消息“at91x40_timer_init”， 看下图：

![at91x40_timer_init](../assets/2018-05-24_at91x40_timer_init.png)

从82、62行，可以知道时钟中断产生后，将调用“at91x40_timer_interrupt”函数，调用图：

![scheduler_tick](../assets/2018-05-24_scheduler_tick.png)

最终调到scheduler_tick函数， 从函数的注释和大概内容可以知道它会引起进程调度，此篇主要“分析”硬中断和时钟中断，对进程调度就不多说了。




