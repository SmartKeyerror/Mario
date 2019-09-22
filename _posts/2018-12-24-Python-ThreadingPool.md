---
layout: post
cover: false
navigation: true
title: Java并发编程(05)--Python线程池源码剖析
date: 2018-12-24 09:49:09
tags: ['并发编程']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---


本来是一个对`Java`并发编程的一个学习和总结专题， 虽然`Python`有`GIL`的存在， 但不能否认其线程池的实现非常的简洁而优雅， 此外温故而知新， 通过理解其它语言的线程池也能够加深我们对`Java`线程池的理解。

<!---more--->

#### 1. 简单使用
```java
from concurrent.futures import ThreadPoolExecutor


def do_something(*args, **kwargs):
    print("*****")

def start(*args, **kwargs):
    with ThreadPoolExecutor(10) as executor:
        for i in range(20):
            executor.submit(do_something, *args, **kwargs)

if __name__ == "__main__":
    start()
```

在`Python`中使用线程池非常的简单， `ThreadPoolExecutor`只接收`max_workers`和`thread_name_prefix`这两个`__init__`参数， 其余的内容`Python`都为我们准备好了， 直接使用即可。

并且由于`ThreadPoolExecutor`实现了`__enter__`和`__exit__`方法， 使得我们可以使用上下文管理器来自动管理线程池的关闭， 而不必担心资源没有被释放的问题。

#### 2. `__init__`方法剖析
```python
def __init__(self, max_workers=None, thread_name_prefix=''):
    if max_workers is None:
        max_workers = (os.cpu_count() or 1) * 5
    if max_workers <= 0:
        raise ValueError("max_workers must be greater than 0")

    self._max_workers = max_workers
    # 工作队列， 默认是无界的
    self._work_queue = queue.Queue()
    # 保存线程的集合
    self._threads = set()
    self._shutdown = False
    self._shutdown_lock = threading.Lock()
    self._thread_name_prefix = (thread_name_prefix or
                                ("ThreadPoolExecutor-%d" % self._counter()))
```

首先是对`max_workers`参数进行判断以及赋值， 当不传该参数时， 默认会给5倍的CPU数量或者5个工作线程。 这里的工作队列使用了`queue.Queue`， 一个无界且线程安全的队列， 然后定义了成员锁变量以及其它参数。


#### 3. `submit`方法剖析
```python
def submit(self, fn, *args, **kwargs):
    with self._shutdown_lock:
        if self._shutdown:
            raise RuntimeError('cannot schedule new futures after shutdown')

        f = _base.Future()
        w = _WorkItem(f, fn, args, kwargs)

        self._work_queue.put(w)
        self._adjust_thread_count()
        return f

def _adjust_thread_count(self):
    def weakref_cb(_, q=self._work_queue):
        q.put(None)
    num_threads = len(self._threads)
    if num_threads < self._max_workers:
        thread_name = '%s_%d' % (self._thread_name_prefix or self,
                                 num_threads)
        t = threading.Thread(name=thread_name, target=_worker,
                             args=(weakref.ref(self, weakref_cb),
                                   self._work_queue))
        t.daemon = True  # 守护线程
        t.start()
        self._threads.add(t)  # 将该线程置于集合中
        _threads_queues[t] = self._work_queue
```

`submit`方法可以认为是`ThreadPoolExecutor`的核心方法， 该方法接收一个函数对象以及该函数的参数， 将该函数包装成一个`_WorkItem`对象。 该对象做的事情其实也很简单， 执行任务函数， 并将结果赋予到`Future`对象的`_result`私有属性中。 将包装对象放置于任务队列中， 然后判断是否需要开启新的线程来执行刚才所提交的任务。 当当前的线程数量小于最大线程数时， 开启一个新的守护线程去执行`_worker`函数。

那么`_worker`应该就是一个循环， 不断的从工作队列中取任务并执行， 源码也证实了这一点：
```python
def _worker(executor_reference, work_queue):
    try:
        while True:
            # 以阻塞方式不断的取任务
            work_item = work_queue.get(block=True)
            if work_item is not None:
                # 执行该任务， 并将结果置于Future对象中
                work_item.run()
                # Delete references to object. See issue16284
                del work_item
                continue
            executor = executor_reference()  # executor是一个弱引用对象
            if _shutdown or executor is None or executor._shutdown:
                # Notice other workers
                # 以一种链式的方式通知其它工作线程结束
                work_queue.put(None)
                return
            del executor
    except BaseException:
        _base.LOGGER.critical('Exception in worker', exc_info=True)
```

值得探究的是当`_shutdown`为真或者其它标志着线程池已经关闭的变量为真时， 代码向工作队列里面儿丢了一个`None`值， 为什么？ 假设我们现在有10个工作线程， 其中一部分在执行任务， 另一部分休眠等待任务获取， 此时我们关闭该线程池， 会发生什么？

下面的`shutdown`方法贴出了关闭线程池时所做的事情， 因为`work_queue.get()`是一种抢占式的， 谁先拿到锁谁获得到任务， 所以10个工作线程其中一个拿到了`None`, 知道了线程池被关闭， 在关闭自己之前得通知其它兄弟关闭自己， 所以往队列中丢一个`None`值。 那么这样一来， 每个线程结束自己之前都通知其它兄弟线程该溜了， 到最后所有的线程都能平滑的关闭。

#### 3. `shutdown`方法剖析
```python
def shutdown(self, wait=True):
    # 首先获取显示锁， 避免有新的任务继续添加
    with self._shutdown_lock:
        self._shutdown = True
        self._work_queue.put(None)
    # 如果wait为True， 则该方法等待所有的任务执行完毕之后才返回
    if wait:
        for t in self._threads:
            t.join()
```
`Python`的`shutdown`主要做了两件事： 重置`_shutdown`变量， 以及向工作队列中添加一个`None`值。 添加`None`值的意义在于使得`_worker`方法中的`work_queue.get(block=True)`从阻塞状态中被唤醒， 得以继续向下执行， 由于取出来的任务为None， 并且此时`_shutdown`为`True`， 那么该工作线程自然而然的就退出了。

到这里其实任务的创建与添加， 线程池内部原理以及线程池的关闭就完成了， 剩下的就是一些高级的内容， 例如线程池的`map`方法以及`Future`对象的内部组成， 接下来这两个内容。

#### 4. `Executor.map`方法剖析
`ThreadPoolExecutor`继承于`Executor`类， 重写了`submit`以及`shutdown`方法， 并添加了一些其余的核心函数， 而`map`方法则是定义在`Executor`对象中的:

```python
def map(self, fn, *iterables, **kwargs):
    timeout = kwargs.get('timeout')
    if timeout is not None:
        end_time = timeout + time.time()

    fs = [self.submit(fn, *args) for args in itertools.izip(*iterables)]

    def result_iterator():
        try:
            for future in fs:
                if timeout is None:
                    yield future.result()
                else:
                    yield future.result(end_time - time.time())
        finally:
            for future in fs:
                future.cancel()
    return result_iterator()
```
首先是获取`timeout`参数， 单位为秒， 用于设置该任务的最大执行时间。 当任务执行超过了该时间， 方法将会抛出`TimeoutError`。 `izip`方法有些类似于`zip`方法， 将两个或者多个迭代对象进行组合， 并返回一个迭代器：
```python
from itertools import izip

a = [1, 2, 3]
b = [4, 5, 6]

for i in izip(a, b):
    print(i)

[output]:
(1, 4)
(2, 5)
(3, 6)
```

总之， `submit`方法返回一个`Future`对象， 那么`fs`这个列表中也就保存着所有任务的`Future`对象。 `result_iterator`是个闭包， 本质上是实现的一个生成器， 该生成器中即为任务执行的结果。

`map`的调用方式其实比较简单， 用于执行任务列表：
```python
with ThreadPoolExecutor(10) as executor:
    res = executor.map(do_something, [1, 2, 3, 4, 5, 6])
```

需要注意的是， `executor.map`方法是阻塞的， 只有在所有的任务执行完毕之后才会返回最终的结果， 但是是以多线程的方式进行执行。


#### 5. `submit`与`map`方法的对比
在`map`方法中， 实际上是调用了`submit`方法， 并获取结果形成一个生成器返回， 此时生成器中为执行结果。 而`submit`方法立即返回一个`Future`对象， 但是结果的获取仍然阻塞， 并且我们需要自己维护一个任务结果队列， 然后进行结果的获取。

这样一来两者直接的区别就非常清晰了：粗略的来讲， `submit`方法适用于不需要返回结果的任务， `map`方法适用于需要返回结果的任务。 然而现实并没有那么简单, `submit`更加灵活， 能够对每一个任务进行状态查看和取消， 而`map`方法更像一个黑盒子， 无法查看任务执行的过程。

还是那句话， 选哪个， 看具体的业务场景和需求。 但是为了避免"手里是锤子， 看什么都是钉子"的现象发生， 两种方法都需要掌握。


#### 6. Future对象
在了解`Future`对象的代码之前， 首先需要理解`Condition`对象。

##### 6.1 Condition
`Condition`对象提供了对复杂线程同步问题的支持， 同时`Condition`又被称为"条件锁"或者"条件变量"， 除了提供像`Lock`一样的`acquire`以及`release`这些常规的锁方法之外， 还提供了`wait`以及`notify`方法。

当线程acquire一个条件变量， 当条件变量不满足条件的时候， 则进行wait操作； 当条件满足时， 进行处理并改变一些其它的条件， 使用notify方法通知其它的线程(兄dei， 你的条件满足了， 不要再等待了， Go!)。 当我们不断的重复这个过程时， 就能够达到复杂问题的同步化。

并且`Condition`对象的中的锁是可重入的， 即可以连续的调用`Condition.acquire`方法

##### 6.2 Future对象方法
纵观整个的`Future`对象代码， 方法比较简单， 主要就是`set_result`以及`set_exception_info`， 其余的就是对`Condition`的加锁， 解锁以及notify。 因为时间的关系， 所以没有对该对象进行`DEBUG`调试， 那么每个锁为什么这么加， 为什么需要调用`notify_all`方法， 目前仍然不是很清楚。

另外需要注意的是， 当我们调用`ThreadPoolExecutor.submit`方法并接收了一个`Future`对象时， 如果此时任务已经开始执行，那么该任务是不可被取消的。 只有任务在`PENDING`时， 任务才有可能被取消执行， 这一点需要特别注意。


#### 7. 使用线程池
在上面大致了解释了一下`ThreadPoolExecutor`的源码， 看源码是为了更好的使用， 那么将源码所分析的信息进行整理， 就能够得到很清晰的使用经验。

1）执行I/O密集型任务，并以异步的方式进行执行(任务添加完成之后直接去干别的事情)。
```python
executor = ThreadPoolExecutor(10)
for i in range(20):
    executor.submit(do_something, *args, **kwargs)
executor.shutdown(wait=False)
```

2）需要等待结果的产生， 并使用执行的结果去做下面的事情
```python
# 那么在这种情况下， 就可以使用with语句块， 并结合map方法直接获取结果

with ThreadPoolExecutor(10) as executor:
    # 阻塞直到所有结果返回
    results = list(executor.map(do_something, [i for i in range(20)]))
```

3）既希望使用`with`， 又想以异步的方式执行， 此时我们可以继承`ThreadPoolExecutor`， 改写其`__exit__`方法
```python
class SynchronousThreadPoolExecutor(ThreadPoolExecutor):
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.shutdown(wait=False)
        return False
```

4）线程池的创建和线程的创建需要花费时间， 所以希望以单例的方式在整个系统中运行。 此时可以使用类的单例模式， 也可以使用`import`的方式， `Python`中的单例实现方式有多种， 比较灵活， 熟悉哪个用哪个
```python
# utils.py
thread_pool = ThreadPoolExecutor(10)

# client.py
from utils import thread_pool

# 或者使用__new__实现单例
lock = Lock()

class SingletonThreadPool(object):
    pool = {}

    def __new__(cls, *args, **kwargs):

        if "pool" not in cls.pool:
            # 采用双重校验锁, 确保单一实例的初始化以及提高并发量
            with lock:
                if "pool" not in cls.pool:
                    thread_pool = ThreadPoolExecutor(10)
                    cls.pool["pool"] = thread_pool
        return cls.pool["pool"]
```