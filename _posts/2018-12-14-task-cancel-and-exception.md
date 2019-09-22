---
layout: post
cover: false
navigation: true
title: Java并发编程(03)--任务的取消与异常处理
date: 2018-12-14 09:49:09
tags: ['并发编程']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

在前面的一章整理并发编程的一些基础内容， 包括任务的创建， 任务的执行， 线程池的简单使用， 加入一个线程以及守护线程和线程同步。 基本上涵盖了绝大多数的基础内容， 在本章中学习任务的取消与异常处理。

<!---more--->

#### 1. 任务取消
如果外部代码能在某个操作正常完成之前将其置于"完成"状态， 那么这个操作就可以称为可取消的， 取消某个操作的原因有很多：

1）**用户请求取消**： 这个任务用户不想做了，点击请求按钮， 或者发送取消的http请求。
2）**有时间限制的操作**： 当一个任务的执行时间超过了我们规定的时间， 我们需要把它干掉。
3）**应用程序时间**： 比如一个多线程的搜索程序， 当一个线程找到了想要的东西， 那么其余的线程也就没有存在的必要了， 将其干掉以释放资源。
4）**错误**： 比如一个爬虫在爬取数据并保存于数据库时， 发生了错误， 如网络终端， 磁盘空间已满等， 这个时候所有的爬取任务都应取消，并且记录当前状态， 以便稍后启动。
5）**关闭**： 当一个程序或服务关闭时， 必须对正在处理和等待处理的工作执行某种操作。 在平缓的关闭过程中， 当前正在执行的任务将继续执行到完成为止， 而在立即关闭过程中， 当前的任务可能取消。

可以看到上面的5个原因在我们的应用中完全有可能出现， 并且出现的几率还很大， 所以想要构建出行为良好的应用， 那么任务和线程的取消与关闭是必须掌握的。

`Java`中并没有一种安全的抢占方式方法来停止线程， 只有一些协作式的机制来终止任务的执行。 比如我们对一个任务添加一个标志， 当标志为真时执行， 为假时停止， 这样就达到了外部控制的效果。

```java
public class PrimeGenerator implements Runnable {

    private final ArrayList<BigInteger> results = new ArrayList<BigInteger>();

    private volatile Boolean isCanceled = false;

    public void run() {
        BigInteger p = BigInteger.ONE;
        while (! isCanceled) {
            p = p.nextProbablePrime();
            synchronized (this) {
                results.add(p);
            }
        }
    }

    public void cancel() {
        isCanceled = true;
    }

    public synchronized ArrayList<BigInteger> getResults() {
        return new ArrayList<BigInteger>(results);
    }
}
```
这个例子我直接从`Java并发编程实战`的7-2程序清单粘过来， 例子很有代表性。 首先这个标志位必须是`volatile`类型， 使得我们能够在任何时刻都能获取标志位的最新状态。  此外我们还需要对写操作和读操作加锁， 写操作加锁通常很好理解， 读操作加锁是为了能够实时的获取最新数据， 也就是每一次的方法调用一定是当前的最新状态。

这样一来在任务运行之后， 在想要停止的地方调用相同实例的`cancel`方法即可终止任务。


#### 2. 中断
通过标志位的方式能够解决一部分的问题， 但是如果一个任务调用了某个阻塞方法， 比如像`BlockingQueue.put`， 建立一个阻塞式的`socket`， 那么此时任务会被阻塞， 根本无法查看标志位的变量变化， 这个时候就需要使用其它的方式来取消一个任务。

`Thread`类为我们提供了`interrupt`方法来中断一个线程， 并且提供了`isInterrupted`方法来返回目标线程的中断状况。

像阻塞库的方法， `Thread.sleep`和`Object.wait`等， 都会检查线程何时中断， 并且在发现中断时提前返回。 整体流程： 清除中断状态， 抛出`InterruptedException`， 表示阻塞操作由于中断而提前结束。 `JVM`并不能保证阻塞方法检测到中断的速度， 但在实际情况中还是很快的。
```java
class PrimeProducer extends Thread {

    private final BlockingQueue<BigInteger> blockingQueue;

    PrimeProducer(BlockingQueue<BigInteger> blockingQueue) {
        this.blockingQueue = blockingQueue;
    }

    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!Thread.currentThread().isInterrupted()) {
                blockingQueue.put(p.nextProbablePrime());
            }
        } catch (InterruptedException e) {
            // 这里线程退出即可， 并执行一些资源清理的工作
        }
    }

    public void cancel() {interrupt();}
}
```
之所以在`while`中进行一次中断检测是为了更快的响应中断， 如果可中断的阻塞方法`put`调用的频率不是很高的话， 那么该线程就对中断的处理就会很迟钝， 所以我们再加一个主动的判断， 以更快的响应。

通常来讲， 使用中断是比较好的停止线程工作的策略。


#### 3. 运行时异常抛出
任何代码都可能会抛出`RuntimeException`， 然而对于`Runnable`而言， 我们没有办法去捕捉这类异常， 约定俗成的是`Runnable`任务自行处理所有的异常。 当线程有异常抛出时， 线程就是终止运行， 并将异常信息输出至控制台， 这种方式其实不是很友好， 因为没人会看控制台， 只会检查日志。 所以这个时候我们需要一种手段来获取运行时抛出的异常。

`Thread.UncaughtExceptionHandler`是javaSE5中的新接口， 它允许我们在每一个Thread对象上添加一个异常处理器(`UncaughtExceptionHandler`)。 我们可以这样来定义：

```java
class TestDemo implements Runnable {
    @Override
    public void run() {
        throw new RuntimeException("运行时异常抛出");
    }
}

class ThreadExceptionHandler implements Thread.UncaughtExceptionHandler {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.println("处理器接收异常信息为: " + e);
    }
}

public class ExceptionThread {
    public static void main(String[] args) {
        for (int i = 0; i < 2; i++) {
            Thread t = new Thread(new TestDemo());
            t.setUncaughtExceptionHandler(new ThreadExceptionHandler());
            t.start();
        }
    }
}
```

那么这个时候我们就可以将一些意料之外的异常记录到日志当中了， 当然了， 这是一种比较绕的解决方案。 如果能够在线程内解决该异常就在线程内进行捕捉处理。

```java
class TestDemo implements Runnable {
    @Override
    public void run() {
        try {
            throw new RuntimeException("运行时异常抛出");
        } catch (Exception e) {
            System.out.println("我捕捉到了异常");
        } finally {
            System.out.println("清理工作");
        }
    }
}
```

这是一种更为彻底的解决方案， 但是这样的方式如果是有多个线程同时抛出异常， 日志记录会有很多的重复， 使用`UncaughtExceptionHandler`则可以避免这个问题。