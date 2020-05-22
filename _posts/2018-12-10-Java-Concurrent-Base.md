---
layout: post
cover: false
navigation: true
title: Java并发编程(01)--基础学习
date: 2018-12-10 08:49:09
tags: ['并发编程']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---



打了2年多的`Python`代码， 大大小小的项目也做了一些， 代码规范和并发效率一直以来是比较头疼的问题。 因为`GIL`全局解释器锁的存在使得`Python`程序员永远只能使用单核， 并且在锁的保护下许多的效率问题都被掩盖。 在学习了`Java`之后， 对其并发模块的设计深感惊艳， 比如`ConcurrentHashMap`的分段锁实现， `volatile`关键字保证变量的可见性， 所以在这里对其进行整理并进一步加深理解。

<!---more--->

#### 1. 为什么要使用多线程
1) 提高资源的利用率： 当某些情况， 程序必须要等待外部的某个操作执行完成， 比如`socket`的连接与建立， 那么此时程序只能等待， 无法执行其它任务。 多线程可以在程序等待时做一些其它的事情， 提高CPU的利用效率。
2）提高公平性： 假设我们的PC只有单核， 并且以单线程的方式运行， 那么当一个程序运行时发生了长时间的阻塞时， 后续所有的任务均被阻塞。 而使用多线程后CPU会尽可能的执行每个线程同样的时间， 达到最大的公平性， 从而一个程序阻塞了也不会影响整个用户。

#### 2. 定义一个任务
在`Java`中有两种方式来定义一个可以使用多线程的方式所执行的任务： 实现`Runnable`接口， 实现`Callbale`接口。 前者用于任务无具体的返回值或者我们根本不关心返回值是什么的任务， 后者用于任务有具体的返回值并且我们需要返回值来进行处理。
两个接口都非常的简单， 只有一个方法需要被实现。
##### 2.1 Runnable任务
```java
public class Count implements Runnable {

    public void run() {
        System.out.println("This is runnable task");
    }
}
```

##### 2.2 Callable任务
```java
public class Count implements Callable<String> {

    public String call() throws Exception {
        return "This is callable test";
    }
}
```
因为`Callable`需要对返回值进行获取， 那么自然而然的需要使用到泛型， 并且在`call`方法中主动的抛出异常， 这一点的设计在线程池中将会得到体现。

#### 3. 使用多线程的方式执行任务
对于`Runnable`的任务而言， 我们处理起来就非常的简单， 将`Runnable`对象传递给`Thread`类并调用`Thread.start`即可。
```java
public class Count implements Runnable {

    public void run() {
        System.out.println("This is runnable task");
    }

    public static void main(String[] args) {
        Thread t = new Thread(new Count());
        t.start();
        System.out.println("This is main thread");
    }
}
```
代码看起来虽然非常的简单， 但是里面还是有相当多的细节值得我们去分析。
1) 当主线程打印完"This is main thread"之后程序会结束吗？
不会， 因为JVM会等到程序内没有线程(除守护线程)在运行时才关闭
2）在`main`线程中开启的线程和`t`线程之间有优先级关系吗？
没有， 线程作为资源调度的基本单位， 在CPU的时间片轮转中每个线程都会得到执行。 CPU喜欢谁就多执行一点儿时间， 所以在该代码下没有优先级一说。
3）能否预测`main`线程和`t`线程的执行顺序？
不能， 在不同的平台， 甚至同样的平台不同的环境下两个线程所执行的顺序和时机都不尽相同， CPU为乱序执行， 所以无法预测线程的执行顺序。

通过上面一些简单的分析， 可以看出多线程并没有我们想象中的那么简单， 其复杂性会与操作系统以及硬件CPU有直接的关系。

对于`Callable`的任务而言， 就要更加复杂一些。 因为我们需要拿到线程任务的返回值， 所以就必须使用`ExecutorService.submit()`进行调用。
```java
public class Count implements Callable<String> {
    private int count;

    Count(int i) {
        count = i;
    }

    public String call() throws Exception {
        return "callable: " + count;
    }

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        ArrayList<Future<String>> results = new ArrayList<Future<String>>();
        for (int i = 0; i < 10; i++) {
            results.add(executorService.submit(new Count(i)));
        }
        for (Future future : results) {
            try {
                System.out.println(future.get());
            } catch (Exception e) {
                System.out.println(e);
            } finally {
                executorService.shutdown();
            }
        }
    }
}
```
代码量明显的就上来了， 相比`Runnable`的任务而言。 `submit`方法会产生一个`Future`对象， 这里我们放到了一个数组中， 并且在遍历数组时尝试获取返回值， 当当前任务没有结束时， `future.get`方法会阻塞， 直到有返回值为止。 `newFixedThreadPool`为一个固定线程数量的线程池， 具体的用法在线程池章节中再整理。 另外需要注意的是， 如果我们不主动的关闭线程池， 那么`JVM`就不会停止运行， 内存也不会得到释放。


#### 4. 线程池的简单使用
在`Web Server`中我们通常的做法是使用多线程的方式来处理并发请求， 但是由于服务器资源有限， 所能够创建的线程数量是有限的， 并且如果创建了大量的线程， 那么这些线程的上下文切换将会带来大量的资源开销， 所以我们需要限制创建的线程数量。 此时就可以使用线程池来进行限制。
`Executors`中的静态工厂方法提供了4种线程池：
1) `newFixedThreadPool`： 固定长度的线程池， 每提交一个任务创建一个线程， 直到达到最大线程数量， 此时线程池的规模不再发生变化。 此时若再有新任务提交会等到池中有可用线程时才会被执行。
2）`newCachedThreadPool`：无固定长度， 可伸缩的线程池。 当任务数量小于线程数量时将回收空闲线程， 当需求增加时， 会增加线程的数量， 其规模仅受操作系统和硬件的限制。
3）`newSingleThreadPool`： 单线程线程池， 通常会作为优先级队列使用。
4）`newScheduledThreadPool`： 创建一个固定长度的线程池， 并且以延迟或定时的方式来执行任务。

如何创建一个线程池， 并向其中提交任务在上一小结已经介绍过了。 在一般情况下这些线程池就已经能够满足我们的需求了， 但是总会有特殊情况， 需要我们定制一个线程池。


#### 5. 配置ThreadPoolExecutor
可以简单的看下`newFixedThreadPool`这个工厂函数：
```java
public class Executors {
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
}
```
当我们使用这个工厂方式时， 其实会返回一个`ThreadPoolExecutor`对象回来， 也可以看到这里的任务队列是使用的`LinkedBlockingQueue`。 链表头部插入和获取效率非常快， 所以用在这里比较的合适。 需要注意一点的是， `LinkedBlockingQueue`虽然有最大长度， 为`0x7fffffff`,  即`int`型最大值(值为2147483647)， 但是这么大的数值在一般的服务器中内存中根本无法存储， 所以说可以认为该队列就是无界的。 也就是说`newFixedThreadPool`这个线程池对任务的数量是没有限制的， 除非达到了硬件的最大值。 所以才需要进行定制化。
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
4）unit: 时间单位， 为keepAliveTime参数提供的
5）workQueue： 工作队列
6）threadFactory： 创建线程的工厂
7）handler： 当线程池中线程用尽并且工作队列达到最大长度时的处理器

需要注意区分基本大小和线程池最大大小， 前者为没有任务执行时的线程池大小， 给0都可以。 后者为实际上我们的线程池具体能有多少个线程， 给0是不可以的， 通常来讲会根据操作系统以及任务情况来综合判断该值大小。
另外一点就是工作队列如何选取的问题： 工作队列常分为3种， 有界队列， 无界队列以及`SynchronousQueue`。 `SynchronousQueue`并不是一种真正的队列， 而是一种在线程之间移交的机制。 要将一个任务放入到`SynchronousQueue`中， 那么必须有一个线程正在等待处理。 如果没有线程等待， 并且当前的线程数量没有达到最大线程数量限制时， 将会开启一个新的线程进行处理。 如果线程数已经饱和， 那么此时会根据饱和策略对任务进行拒绝。 在`newCachedThreadPoll`中就使用了这种队列进行任务的移交。
这里贴`Java并发实战`第8章对于线程池的选择：
> 对于Executor， newCachedThreadPoll工厂方法是一种很好的默认选择， 他能提供比固定大小的线程池更好的排队性能。 当需要限制当前任务的数量以满足资源管理需求时， 那么可以选择固定大小的线程池， 例如Web Server中， 如果对此类任务不进行限制的话， 很容易发生内存溢出的问题。

只有当任务相互独立时， 为线程池或工作队列设置界限才是合理的。 如果任务之间存在依赖性， 那么有界的线程池或队列就可能导致线程"饥饿"死锁问题。 此时应该使用无界的线程池， 比如`newCachedThreadPoll`。

线程池的选择和配置其实是一件很复杂的事情， 也不打算在这里一次性的整理完毕， 所以我们只需要知道什么情况下选择什么样的线程池即可， 更多的定制内容开新的文章进行整理。


#### 6. 守护线程(后台线程)
守护线程作为一种非必需的线程使用， 或者为了管理线程的方便而使用。 守护线程的唯一特点就是当程序中没有任何的非守护线程工作时， JVM将会退出运行， 并将所有的守护线程杀死。 从另一个角度来讲， 只有存在任何的非守护线程在运行时， 程序就不会退出。

那么在这里就需要明确一个事实： 线程与线程之间没有依赖性， 当A线程中开出一个守护线程B， 两个线程同时运行， 某一段时间之后A线程退出， 只要此时系统中还有其余的非守护线程运行， B线程就不会退出。 进程作为资源分配的基本单位， 而线程则是资源调度的基本单位， 所以线程只会依赖进程， 而不会依赖线程。

下面的`Java`代码是为了证明上面所说的守护线程与非守护线程不存在依赖关系的一个小Demo， 在`main`线程中开出一个线程A， A线程在运行的初期开启一个守护线程B， 并使用`volatile`变量来进行A线程的取消， `main`线程在A、B两个线程运行一段时间之后取消A线程的运行， 并且执行`while`循环， 使`JVM`不会退出。

代码运行结果也能够证明普通线程与非守护线程之间是没有任何依赖关系的， 除非我们主动的使用变量或者其它通信手段来将两个线程进行连接。 这种非依赖关系和语言是无关的， 在`Python`语言中同样如此：

```python
# Python中并不需要volatile这种东西， 因为Python中存在GIL
is_canceled = False

def daemon_thread():
    while True:
        print("This is daemon_thread")
        time.sleep(1)

def do_something():
    t = threading.Thread(target=daemon_thread)
    t.setDaemon(True)
    t.start()

    while not is_canceled:
        print("This is do_something thread")
        time.sleep(1)
    print("do_something thread exit")

if __name__ == "__main__":
    t = threading.Thread(target=do_something)
    t.start()
    time.sleep(5)
    is_canceled = True
    while True:
        print("This is main thread")
        time.sleep(2)
```

在明确了这些内容之后， 我们就可以很方便的找出守护线程能够应用的地方了。 当系统中没有非守护线程时， JVM一定会退出并且清理守护线程， 那么守护线程就可以作为一种"守护者"存在于系统的生命周期中。 例如垃圾回收。

```java
public class DaemonDemo implements Runnable {
    private volatile boolean isCanceled = false;

    public void setCanceled(boolean canceled) {
        isCanceled = canceled;
    }

    public void run() {
        // 在该线程中开出一个"子"线程
        Thread t = new Thread(new Runnable() {
            public void run() {
                while (true) {
                    try {
                        // "子"线程打印语句并休眠
                        System.out.println("This is daemon thread");
                        TimeUnit.SECONDS.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                        break;
                    }
                }
            }
        });
        // 将该线程置为守护线程并开启
        t.setDaemon(true);
        t.start();

        while (!isCanceled) {
            try {
                System.out.println("This is father thread");
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws Exception{
        DaemonDemo daemonDemo = new DaemonDemo();
        Thread t = new Thread(daemonDemo);
        t.start();

        TimeUnit.SECONDS.sleep(5);

        daemonDemo.setCanceled(true);

        while (true) {
            System.out.println("This is main thread");
            TimeUnit.SECONDS.sleep(10);
        }
    }
}
```


#### 7. 加入一个线程
加入一个线程使用`join`方法， 与`Python`的作用调用方式是一样的。
```java
public static void main(String[] args) throws Exception{
    Thread t = new Thread(new Runnable (...));
    t.start();
    t.join()

    System.out.println("This is main thread");
}
```
"This is main thread"这条语句只有在`t`线程执行完毕之后才会被打印， 这个就是`join`的用法： 等待某个线程的任务完成才继续向下执行。


#### 8. 共享受限资源
`资源`在并发编程中拥有很多层含义， 比如变量， 某种数据结构或者一个类对象， 当两个线程同时访问同一个资源并进行操作时， 就有可能会出现数据混乱的问题。
```java
public class ConcurrentVariable implements Runnable{
    private int number = 0;

    public void run() {
        number += 1;
    }
}
```
如果开启多个线程同时对该任务进行执行， 那么最终的结果很有可能不等于开启的线程数量， 因为`number`自增操作不是原子性的。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-01%2016-34-26%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

当两个线程同时对一个资源进行非原子操作时， 就会出现上图所示的情况： 两个线程执行完`number += 1`之后其值应为2， 但是由于并发执行的原因， 最终的执行结果可能是1。

所以此时我们需要对资源加锁， 以保证对资源的操作是原子性的。 `Java`提供了`synchronized`互斥锁以及显示锁， `synchronized`使用频率比较多。

可以认为在同一个类中的所有`synchronized`关键字包含的代码所持有的锁都是同一个， 有些类似于：
```python
class Demo:
    self.__lock = Lock()

    def do_something():
        with self.__lock:
            ...

    def do_something2():
        with self.__lock:
            ...
```

只不过`synchronized`帮我们完成了锁定义和加锁， 释放锁的操作， 每个对象默认自动的含有单一的锁。

```java
public void run() {
    synchronized (this) {
        number += 1;
    }
}
```

`synchronized`可以加在函数上， 也可以只包含某一段需要控制并发的代码。 需要注意的是， 我们需要尽量的控制锁的粒度， 能够在少部分代码上添加， 就不在函数上添加， 否则会带来比较大的并发效率问题。

虽然`synchronized`能够控制并发访问， 但是越简单的东西就会带来更大的约束性：
1）我们无法为`synchronized`添加一个等待锁的过期时间， 这样一来某个线程可能无限的等待锁的释放
2）我们将并发访问的控制权完全的交给了`Java`， 而不能自己控制， 无法进行定制化操作。


##### 8.1 显示锁
`synchronized`非常方便， 但是灵活性比较低； 而显示锁用起来比较麻烦， 但是胜在灵活。 `Java`中所实现的显示锁也很多， 在本篇"基础学习"中只介绍`ReentrantLock`可重入互斥锁。

```java
private Lock lock = new ReentrantLock();

public void run() {
    lock.lock();
    try {
        number += 1;
    } finally {
        lock.unlock();
    }
}
```
流程其实就是为每一个对象定义同一把显示锁->加锁->执行代码->释放锁， 将锁的释放写在`finally`中是一个很好的习惯， 因为不管有没有异常抛出， 锁都能够正常的释放掉。

此外， `ReentrantLock`还提供了`tryLock(long time, TimeUnit unit)`方法， 使得我们可以对等待加锁的时间进行控制。


#### 9. 原子性
**什么是原子性？**若某一个操作为原子性操作， 那么线程就不会在该操作执行时进行上下文切换， 即该操作一定能够在线程切换之前执行完毕。 在`Java`中除了`long`和`double`之外的所有基本类型的操作均为原子性操作， 例如：
```java
int i = 10;
boolean isDelete = false;
```
对于读取和写入这些原子变量时， 可以保证其操作不可再分。 但是对于64位变量， 如`long`和`double`， 其读取和写入是分为2个32位操作完成的， 那么在写这2个32位的数据时， 完全有可能发生线程切换， 导致数据异常。 这种现象有时会被称为`字撕裂`。 所以在并发的场景下使用这些非原子变量时， 可以加锁， 也可以使用`volatile`来保证其原子性。

##### 9.1 volatile
`volatile`可以认为是比`synchronized`更加轻量的锁， 保证了变量的原子性以及内存可见性。 可见性是指当某一个线程修改了一个变量时， 另一个线程一定能够读到最新的数据。 常常用于线程间的变量共享以及线程取消的标志位。

```java
private volatile boolean isCanceled = false;
```

更加具体的实现原理以单独的文章进行讨论， 在这里我们只需要知道`volatile`能够保证变量的原子性操作以及可见性即可。

##### 9.2 原子类
`java`额外的提供了一些原子类来保证变量的原子性操作， 包括`AtomicInteger`, `AtomicLong`以及`AtomicReference`， 以`AtomicInteger`为例：
```java
public class ConcurrentVariable implements Runnable{
    private AtomicInteger atomicInteger = new AtomicInteger(0);

    public int getValue() {
        return atomicInteger.get();
    }

    public void addValue(int value) {
        atomicInteger.addAndGet(value);
    }

    public void run() {
        addValue(1);
    }

    public static void main(String[] args) throws Exception{
        ConcurrentVariable concurrentVariable = new ConcurrentVariable();
        for (int i = 0; i < 10000; i++) {
            Thread t = new Thread(concurrentVariable);
            t.start();
        }

        TimeUnit.SECONDS.sleep(4);

        System.out.println(concurrentVariable.getValue());
    }
}
```


#### 10. 线程隔离
线程隔离并不是一种同步或者锁技术， 而是一种将变量隔离在当前线程的机制。 变量的作用域分为局部变量和全局变量， 通常来讲定义在类中的变量为全局变量， 定义在函数中的变量为局部变量， 而线程变量则是定义在一个线程中的。 可以理解为一个变量在不同的线程中有不同的值或者引用。

在`Flask`中就使用了`ThreadLocal`机制来保证在并发访问的情形下， 当前请求的`request`对象一定是最初的`request`对象， 而不会变成其它线程的`request`对象。 这种机制使得代码更加灵活， 耦合性更低， 因为我们可以在任意地方通过隔离栈来获取当前线程的隔离对象， 而不必使用函数传参的方式将变量传来传去。 代码更加优雅和整洁。


#### 11. 小结
到这里， 关于`java`并发的基础内容就结束了， 此时我们已经可以写一些简单或者稍微复杂一些的并发代码， 但是离强壮的并发代码还有很远的距离。 例如线程的取消与关闭， 对容器的并发使用， 在Web框架下使用并发的手段来提高资源利用率等等。

