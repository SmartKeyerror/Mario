---
layout: post
cover: false
navigation: true
title: Java并发编程(04)--线程池
date: 2018-12-18 09:49:09
tags: ['并发编程']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

抛开`Java`自己封装的`newFixedThreadPool`, `newCachedThreadPool`等工厂线程池方法， 最核心的就是`ThreadPoolExecutor`的配置， 包括线程池的大小， 工作队列， 空闲线程存活时间以及饱和策略。

<!---more--->

#### 1. ThreadPoolExecutor参数
首先来看`ThreadPoolExecutor`的通用构造函数：
```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```
参数具体的含义在注释中也有非常清晰的讲到：

1）corePoolSize： 线程池的基本大小
2）maximumPoolSize：线程池中允许存在的线程数量大小
3）keepAliveTime： 空闲线程的存活时间， 当池子里面有50个线程， 40个线程在执行任务， 那么空闲10个线程。 这空闲的10个线程将会在达到keepAliveTime时被回收
4）unit: 时间单位， 提供给keepAliveTime参数
5）workQueue： 工作队列
6）threadFactory： 创建线程的工厂
7）handler： 当线程池中线程用尽并且工作队列达到最大长度时的处理器

线程池的基本大小(corePoolSize)， 最大大小(maximumPoolSize)以及存活时间等因素共同负责线程的创建与销毁。

基本大小表示在没有任务执行时线程池的大小， 也就是当第一个任务提交时池中的线程数量， 并且只有在**工作队列**满了的情况下才会创建超出这个数量的线程。 也就是说如果我们将基本大小设置为4， 并且使用无界队列的话， 那么池中永远就只有4个线程工作。 `newFixedThreadPool`就是这么干的。

线程池的最大大小表示可同时运行的线程数量的上限， 如果说我们使用基本大小为5、无界队列的线程池的话， 这个值永远不会产生作用。

空闲线程的存活时间表示当一个线程空闲的时间超过了我们设置的存活时间， 就会被标记为可回收， 并且当线程池的当前大小超过了基本大小时， 该线程将被终止。


#### 2. 选择任务队列
基本的任务排队方法有3种：无界队列， 有界队列以及同步移交(`Synchronus Handoff`)， 队列的选择与线程池的基本大小和最大大小有着直接的关联。

`newFixedThreadPool`和`newSingleThreadPool`在默认情况下将使用一个无界的阻塞队列。 如果所有的工作队列处于忙碌状态， 那么任务将会在队列中等待。 如果任务持续快速的到达， 并且超过了线程池处理的速度， 那么队列将会无限制地增加，直到内存溢出。

更稳妥的方式是使用有界队列， 比如`ArrayBlockingQueue`， 有界的链表阻塞队列以及优先队列。 那么当队列填满之后该如何处理？ 这个时候就需要使用饱和策略。

#### 3. 饱和策略
当有界队列被填满之后， 饱和策略开始发挥作用。 `ThreadPoolExecutor`的饱和策略可以通过调用`setRejectedExecutionHandler`来修改。 JDK提供了几种不同的`RejectedExecutionHandler`实现：
```java
AbortPolicy, CallerRunsPolicy, DiscardPolicy, DiscardOldestPolicy
```

1）中止策略(AbortPolicy)是默认的饱和策略， 直接拒绝任务的提交， 并向你抛出一个`RejectedExecutionException`异常， 然后自己处理这个异常。

2）抛弃策略(DiscardPolicy)将会悄咪咪的把你提交的任务干掉， 也不会抛出异常， 仿佛什么都没有发生。

3）抛弃最旧的策略(DiscardOldestPolicy)将会抛弃下一个将被执行的任务， 然后尝试重新提交任务。 需要注意的是如果队列是优先队列的话， 将会抛弃优先级别最高的任务， 所以这种策略最好不要和优先队列一起使用。

4）调用者运行策略(CallerRunsPolicy)实现了一种调节机制， 既不会抛出异常， 也不会抛弃最旧的任务， 而是将某些任务回退到调用者， 从而降低新任务的流量。 简单来讲就是在工作队列填满时， 下一个任务由主线程执行， 那么此时主线程不能提交任务， 并且也不接收新任务， 如果是`Web Server`的话， 此时任务会保存在TCP层的队列中， 如果TCP层队列满了的话， TCP将会抛弃请求。

##### 3.1 自定义饱和策略
如果我们想要自定义饱和策略的话， 那么首先就需要查看一下JDK的源代码， 看看JDK是如何实现的， 此后我们根据源码进行自定义的饱和策略编写即可。

上面儿所提到的4种饱和策略的源码均实现`RejectedExecutionHandler`接口， 那么我们自定义的饱和策略同样实现这个接口即可。

```java
/**
 * AbortPolicy和DiscardPolicy以及其它的饱和策略其实都是ThreadExecutorPool中的内部类,
 * 均实现了RejectedExecutionHandler接口，可以很明白的看到AbortPolicy直接抛出异常，
 * 而DiscardPolicy什么都没做， 任务的拒绝交给了有界队列进行丢弃。
 */

public static class AbortPolicy implements RejectedExecutionHandler {

    public AbortPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " +
                                             e.toString());
    }
}

public static class DiscardPolicy implements RejectedExecutionHandler {

    public DiscardPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}
```




#### 4. 线程工厂
在我们使用`newFixedThreadPool`或者是`newCachedThreadPool`时， 都是使用默认的线程工厂来创建线程：
```java
static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
                      poolNumber.getAndIncrement() +
                     "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

其实自定义一个线程工厂也比较简单， 实现`ThreadFactory`接口， 并在`newThread`方法中赋予线程一些其余的属性或者特性即可。 例如我们想要将线程池中的所有线程都设置为守护线程， 或者自定义我们的线程名称。 例如：
```java
public class MyThreadFactory implements ThreadFactory {

    private String prefix;

    private boolean threadIsDaemon = false;

    public MyThreadFactory(String prefix) { this.prefix = prefix; }

    public MyThreadFactory(String prefix, boolean threadIsDaemon) {
        this.prefix = prefix;
        this.threadIsDaemon = threadIsDaemon;
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(r);
        t.setName(prefix + t.getName());
        t.setDaemon(threadIsDaemon);
        return t;
    }
}
```

主要是对线程池中的线程做一些属性的配置， 没有什么很复杂的地方。


#### 5. 线程池的关闭
线程池的关闭主要有两个函数， `shutdown`以及`shutdownNow`， 前者在调用之后首先禁止添加新的任务， 然后执行完所有的任务之后关闭线程池。 `shutdownNow`在调用后同样禁止添加新的任务， 然后中断所有的工作线程， 并返回没有被执行任务列表(`List<Runnable>`)。
两种方式各有其应用场景， 视具体的业务来决定需要调用哪个关闭函数。


#### 6. 线程池的大小如何选择
如果任务是CPU密集型， 例如图片的拼接， 转换或者是其它需要计算的任务， 那么此时线程数量与CPU核心数相同即可。 因为CPU密集型的任务没有阻塞和等待， 所需要的只是CPU执行更长的时间， 而不是频繁的线程上下文切换。

如果任务是I/O密集型， 虽然有理论上的计算公式， 但是很难计算系统中的阻塞系数， 那么此时可能需要进行一些并发的测试， 例如200个任务开25， 50， 75， 100个线程分别进行测试， 找到最佳的平衡点。 并不是线程数量越多越好， 操作系统创建一个线程以及线程的上下文切换都可以认为是重量级的操作， 应避免过多的上下文切换。 这个时候我认为只能通过测试以及实验的方法来找到平衡点， 这种情况的数量是无法被计算的。


#### 7. 小结
线程池的内容基本上就只有这么多， 在理清的其运行机制以及参数所代表的含义之后， 更多的是如何正确的配置以及使用， 例如是以单例模式应用于整个系统， 还是分布于系统的各个角落， 都需要根据特定的业务场景以及架构方式来进行选择。 并没有哪种方式绝对的好， 也没有哪种方式绝对的差， 有的只是权衡利弊。