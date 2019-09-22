---
layout: post
cover: false
navigation: true
title: Java并发编程(06)--AbstractQueuedSynchronizer
date: 2018-12-26 09:49:09
tags: ['并发编程']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---


在整理下一篇文章， 有关锁的知识之前， 有一个无论如何都绕不开的话题：`AbstractQueuedSynchronizer`， 队列同步器， 通常简称AQS。

<!---more--->

#### 1. 产生由来
在`JDK 1.5`之前， `synchronized`几乎是使用最为广泛的线程同步机制， 然而该关键字的设计采用非常保守的策略， 灵活性较差。 所以必须有一种机制能够提供更加灵活， 效率更高的同步锁， AQS应运而生。


#### 2. AQS简介
AQS解决了实现同步器时所涉及的大量细节问题，例如获取同步状态、FIFO同步队列。基于AQS来构建同步器可以带来很多好处。它不仅能够极大地减少实现工作，而且也不必处理在多个位置上发生的竞争问题。 在开发中大量使用的`ReentrantLock`, `Semaphore`等均由AQS所构建， 这样以来， AQS的地位就显得非常之重要了。

更多的设计细节可以参考**Doug Lea**所写的论文， 也是理解AQS所必读的论文之一。

> http://gee.cs.oswego.edu/dl/papers/aqs.pdf


#### 3. AQS最基本原理
参照上面的论文， 可以看出同步器的设计思想其实是非常简单的。
```java
// 加锁
while (synchronization state does not allow acquire) {
    enqueue current thread if not already queued;
    possibly block current thread;
}
dequeue current thread if it was queued;

// 锁的释放
update synchronization state;
if (state may permit a blocked thread to acquire)
    unblock one or more queued threads;
```

1）加锁： 当同步状态不允许加锁时(此时锁被其它线程占有)， 将当前线程送入队列， 并可能使当前线程阻塞； 当可以进行加锁时从队列取出线程并给予相关的锁

2）解锁：首先更新同步状态， 当同步状态允许一个阻塞的线程加锁时， 使队列中的一个或者多个线程解除阻塞。

因此， 想要完成这样的一个同步器， 我们需要3个基本组件：

- 同步状态的原子性管理
- 线程的阻塞与解除阻塞
- 队列的管理

那么下面就对这3个组件进行学习和整理。

##### 3.1 同步状态
AQS类使用单个int(32位)来保存同步状态，并暴露出`getState`、`setState`以及`compareAndSet`操作来读取和更新这个状态。 这些方法都依赖于j.u.c.atomic包的支持， 即这些操作均为原子性的。 另外在JDK 1.8中， 该标志位也可以使用64位的长整形来表示。

基于AQS的具体实现类必须根据暴露出的状态相关的方法定义`tryAcquire`和`tryRelease`方法， 以控制加锁和释放锁的操作。 当同步状态满足时， `tryAcquire`方法必须返回true， 而当新的同步状态允许后续加锁时， `tryRelease`方法也必须返回true。 这些方法都接受一个int类型的参数用于传递想要的状态。 例如：可重入锁中， 当某个线程从条件等待中返回， 然后重新获取锁时， 为了重新建立循环计数的场景。 很多同步器并不需要这样一个参数， 因此忽略它即可。


##### 3.2 线程的阻塞
在java.util.concurrent.locks包中提供了`LockSupport`类来对线程进行阻塞和阻塞的解除


##### 3.3 队列
可以认为同步器最为核心且关键的组件就是管理阻塞的线程队列。 该队列是一个非常严格的FIFO队列， 所以框架不支持基于优先级的同步。

同步队列的最佳选择是自身没有使用底层锁来构造的非阻塞数据结构。 而其中主要有两个选择：一个是Mellor-Crummey和Scott锁(MCS锁)的变体， 另一个是Craig， Landin和Hagersten锁(CLH锁)的变体。 一直以来，CLH锁仅被用于自旋锁。 但是， 在这个框架中， CLH锁显然比MCS锁更合适。 因为CLH锁可以更容易地去实现取消和超时功能，  因此我们选择了CLH锁作为实现的基础。 但是最终的设计已经与原来的CLH锁有较大的出入， 因此下文将对此做出解释。

在继续阅读论文之前， 先来看下**什么是自旋锁。** 等待获取锁的方式有被动和主动两种。 被动模式指的是互相竞争的线程不关心锁是否已经被释放， 而是让自身进入阻塞状态， 等待某种调度机制在锁释放时唤醒其中某个线程； 主动模式通常使用自旋的方式实现：线程进入某种循环主动判断锁的状态并尝试获得锁， 一旦获得了锁会立即退出循环。

尽管自旋锁类似于一种死循环， 会消耗一些计算资源， 但是在**锁资源很快被释放的情境**下， 其开销要小于唤醒一个线程所带来的开销。 另外一点就是自旋锁通常是在用户空间层面的， 而`synchronized`属于重量级锁， 由内核进行托管， 那么内核空间与用户空间的切换也会造成开销。

一个最简单的自旋锁的实现是通过原子变量类进行实现：
```java
public class SpinLock {
	private AtomicReference<Thread> owner = new AtomicReference<>();
	public void lock(){
		Thread current = Thread.currentThread();
		while(!owner.compareAndSet(null, current)) {
		    // 空转
		}
	}
	public void unlock () {
		Thread current = Thread.currentThread();
		owner.compareAndSet(current, null);
	}
}
```
这种实现方式首先需要借助于原子类， 而原子类的CAS操作需要硬件的支持， 并且这里的公平性是无法进行保证的。 所以也就有了`MCS锁`和`CLH锁`。

这里就使用图示简单的描述一下CLH锁的原理： 这个锁本质上是一个双向链表， 并具有头结点和尾节点的引用， 每一个节点可以认为是一个线程。

1）初始化：head节点和tail节点均指向一个虚拟头结点(dummy节点)

2）第一个线程加锁：第一个线程进入并尝试加锁， 此时链表刚初始化完毕， 加锁操作能够直接成功。 创建一个节点， 保存当前获得锁的线程信息。

3）加锁：创建一个新节点， 挂到链表最后， 等待前驱节点的线程释放， 这里使用自旋的方式等待， 而非休眠。

4）释放锁：前驱节点线程释放锁资源， 那么此时该节点的后驱节点停止自旋， 获取资源锁， 并将head节点指向当前节点。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-14%2015-45-02%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

这里只是一个非常浅显的原理分析， 更详细的操作以及一些细节请参考`java-doc`。


#### 4. 使用AQS
`AbstractQueuedSynchronizer`的具体实现其实是很复杂且精妙的， 但是暴露给我们使用的核心API其实只有几个， 我们继承该类， 并重写其相关方法即可。

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    protected AbstractQueuedSynchronizer() { }

    // 尝试获取锁， 成功返回true, 失败返回false
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }

    // 尝试释放锁， 成功返回true, 失败返回false
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }

    // 因为共享锁的加锁和排它锁的加锁有一些差别， 所以这里的返回值有所区别。
    // 返回负值表示加锁失败， 正值表示成功
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }

    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }
```

具体的实现可以查看`ReentrantLock`中`Sync`这个静态内部类， 以及`NonfairSync`， `FairSync`这两个继承自`Sync`类的静态内部类。


#### 5. 小结
本篇文章没有对`AbstractQueuedSynchronizer`做特别深入的分析， 仅从其原理以及设计思想进行了相应的阐述。 在开发中如果能有更深层次的理解当然最好， 但是日常使用我认为到这里就足够了， 遇到更深入的问题再回来深入的分析。
