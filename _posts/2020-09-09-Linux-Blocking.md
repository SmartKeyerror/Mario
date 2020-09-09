---
layout: post
cover: false
navigation: true
title: Linux 阻塞与唤醒实现原理
date: 2020-09-09 07:50:25
tags: ['Linux']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

在前面的文件 I/O 文章中，我们有提到 Linux 文件 I/O 支持阻塞和非阻塞的数据读取方式，当采用阻塞方式进行 I/O 时，进程将会阻塞在`read()`或者`write()`系统调用上，直到文件可读或者是内核缓冲区可写。这些阻塞与唤醒的实现与内核调度紧密相关，Linux 内核使用等待队列和完成量来实现该功能。
> 注: 本篇文章所用Linux内核源码版本为v5.8

<!---more--->

### 1. 进程状态有限状态机

进程并不总是可以立即运行的，一方面是 CPU 资源有限，另一方面则是进程时常需要等待外部事件的发生，例如 I/O 事件、定时器事件等。

因此，对进程的状态进行分类就是一件非常有必要的事情，对于等待某事件发生的进程给予 CPU 资源是没有任何意义的，因为此时事件可能仍未发生。而对于正等待 CPU 资源的进程而言，在得到 CPU 之后即可立即执行。调度器为了尽可能最大地使用硬件资源，通常会将进程分为3个主要的状态: 运行、等待和睡眠。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Snorlax/schedule/process-state.png)

处于运行状态的进程正在使用 CPU 等资源，从上图中可以看到，运行态的进程在执行完任务后结束，进入到结束状态。当 CPU 时间片到期之后，调度器选择其它进程执行，此时将进入等待状态。同时，当运行时的进程发起 I/O 操作，或者等待其它事件的发生时，将进入睡眠状态。

处于等待状态的进程由于缺少 CPU 资源而被迫停止运行，只要调度器下次选中该进程即可立即执行，由等待状态转变为运行状态。

处于睡眠状态的进程在等待外部事件的发生，例如 I/O 操作的数据抵达，创建的定时器到期等等，**处于睡眠状态的进程永远不会被调度器进行选择并执行**。当期望的事件到达后，进程由睡眠状态更改为等待状态，等待调度器的下一次选择。

处于等待的进程将会被放置于就绪队列中（红黑树实现），而处于睡眠状态的进程则放置于等待队列（双链表实现）中。调度器的目光主要放在就绪队列上，从该队列中取出下一个将要执行的进程，而等待队列和就绪队列中的进程会因为事件的发生而进行相互转移。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Snorlax/schedule/read-and-wait-queue.png)

在实际的内核实现中，进程的运行状态表示要比上文所述更加详细一些，进程状态定义于`include/linux/sched.h`:

- `TASK_RUNNING`，可运行状态。此时进程并不一定正在运行，一旦得到调度器的调度即可立即运行。
- `TASK_INTERRUPTIBLE`，可中断睡眠状态。此时进程因为等待外部事件的发生而睡眠，此时可由信号或者是内核唤醒。
- `TASK_UNINTERRUPTIBLE`，不可中断睡眠状态。和`TASK_INTERRUPTIBLE`状态类似，等待外部事件发生的睡眠状态。不同的是改状态只能由内核亲自唤醒，不能由信号唤醒，通常用于进程必须等待某件工作完成，不能被 Kill。

除了这三个核心进程状态以外，还有`__TASK_STOPPED`、`__TASK_TRACED`等状态，由于这些状态在本文中并不重要，所以略去。

### 2. 等待队列

等待队列相关的源码位于`include/linux/wait.h`以及`kernel/sched/wait.c`文件中，头文件中定义了等待队列以及队列元素的基本数据结构，`wait.c`源文件则主要包含具体的方法实现。

首先来看等待队列的基本结构，分为队列头和队列项:

```cpp
/* 等待队列头 */
struct wait_queue_head {
	spinlock_t		lock;       /* 自旋锁 */
	struct list_head	head;   /* previous、next指针 */
};
typedef struct wait_queue_head wait_queue_head_t;

/* 等待队列元素 */
struct wait_queue_entry {
	unsigned int		flags;  /* 标识位 */
	void			*private;   /* 通常指向等待进程 */
	wait_queue_func_t	func;   /* 唤醒函数 */
	struct list_head	entry;  /* previous、next指针 */
};
```

在内核的链表实现中，绝大多数的链表均为循环双链表，等待队列也不例外。因为等待队列可能会在系统中断时进行修改，所以必须要添加互斥锁机制保护队列元素。

等待队列元素的设计也非常简洁，除了双链表必要的前后指针以外，仅包含一个指向等待进程`task_struct`实例的指针，一个唤醒函数和一个标识位。

唤醒函数通常由调度器实现，如`kernel/sched/core.c`中定义的`try_to_wake_up`方法，可以简单的认为唤醒函数就是将进程的状态由`TASK_INTERRUPTIBLE`或`TASK_UNINTERRUPTIBLE`修改为`TASK_RUNNING`，并将其加入至就绪队列中。

`wait.h`中提供了一系列与等待队列相关的宏定义供外部使用，例如`wait_event`，本质上是对等待队列的进一步封装:

```cpp
#define wait_event(wq_head, condition)						\
do {										\
	might_sleep();								\
	if (condition)								\
		break;								\
    /* 这里将原有的__wait_event宏展开，使用___wait_event代替 */     \
	___wait_event(wq_head, condition, TASK_UNINTERRUPTIBLE, 0, 0, schedule())	\
} while (0)
```

其中`wq_head`即`wait_queue_head`，`condition`则是一个C语言表达式，表示一个等待条件。宏定义的`wait_event`使得使用标准C表达式指定条件成为可能，如果使用函数实现的话，无法做到如宏实现的灵活性。注意到在调用`___wait_event`之前会首先检查一遍条件是否满足，避免进行无效的睡眠。

在`___wait_event`宏定义中传入的进程状态为`TASK_UNINTERRUPTIBLE`，也就是说，`wait_event`实现的事件等待是不可中断的。当然，`wait.h`中同样提供了其它时间等待实现:

- `wait_event_timeout`: 带有超时时间的不可中断事件等待
- `wait_event_interruptible`: 可中断的事件等待
- `wait_event_interruptible_timeout`: 带有超时时间的可中断事件等待

最后再来看`___wait_event`实现，该方法将会把当前进程包装成`wait_queue_entry`对象，并发安全地放置于等待队列中，并且在实际的让出CPU资源、引发调度器重新调度之前会再一次的检查等待事件是否发生，避免无效睡眠。由于源代码中该方法宏定义实现符号较多，所以将原实现抽象成伪代码:

```cpp
___wait_event(wq_head, condition, state, exclusive, ret, cmd) {
    /* 初始化队列元素 */
    init_wait_entry(...);
    
    for (;;) {
        /* 将队列元素插入至等待队列中(线程安全) */
        long __int = prepare_to_wait_event(&wq_head, &__wq_entry, state);
        
        /* 检查事件条件是否满足 */
        if (condition)
            break;
            
        /* 触发调度器重新调度 */
        schedule();
}
```

对于唤醒一个进程在前文中已经描述过了，通用方法为`wake_up()`，本质上会调用内核调度模块中的`try_to_wake_up()`来唤醒某个进程，唤醒的实质是将进程状态修改为`TASK_RUNNING`，从等待队列中移出并加入至就绪队列中。