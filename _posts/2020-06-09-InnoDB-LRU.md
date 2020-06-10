---
layout: post
cover: false
navigation: true
title: InnoDB独特的LRU
date: 2020-06-09 18:06:25
tags: ['MySQL']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

由于硬盘和内存的造价差异，一台主机实例的硬盘容量通常会远超于内存容量。对于数据库等应用而言，为了保证更快的查询效率，通常会将使用过的数据放在内存中进行加速读取。LRU算法经常用于数据的置换，但InnoDB的LRU却更加独特。

<!---more--->


### 1. 总览

InnoDB存储引擎是基于硬盘存储的，并且以页(page)的方式对数据记录进行管理。由于硬盘和CPU之间数据处理速度存在巨大差异，所以必须要使用内存来弥补两者之间的速度鸿沟。

正如同操作系统在读取硬盘文件时会将其纳入内核缓冲区一样，InnoDB存储引擎也会为硬盘中的数据和索引建立位于用户空间的内存缓冲池。当数据库从硬盘读取数据时，首先将其放置于位于内存的缓冲池中，下一次读取相同的数据时，首先判断是否位于缓冲池中。若在，则直接返回，若不在，再从硬盘中进行读取。

当修改数据页时，首先修改位于缓冲池中的数据页，InnoDB会寻找合适的时机将此修改持久化至硬盘中，该合适的时机通常由Checkpoint技术决定。

在Linux操作系统中，位于内核的内核缓冲区就是为了提高系统的I/O效率而产生的，那么为什么InnoDB还要建立自己的缓冲区? 内核缓冲区的终端为硬盘，需要处理诸如页对齐、数据边界等和硬盘硬件相关的事宜，而InnoDB缓冲区则主要服务于应用，并不关心硬件细节，并且可由应用程序控制。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/MySQL/InnoDB-Buffer/InnoDB-Buffer-Pool.png)

如上图所示，InnoDB缓冲池主要由数据页和索引页构成，两者占据了缓冲区的绝大部分空间。除此之外，undo页、插入缓冲以及自适应哈希索引等内容也位于缓冲池中。

### 2. 数据页与索引页的LRU

数据页和索引页的目的在于缓存一部分的表数据和索引数据，其数据总量通常会超过缓冲池大小，所以缓冲池中应只缓冲那些经常使用的热点数据。InnoDB内存管理使用的是最近最少使用(Least Recently Used, LRU)算法。来淘汰最久未使用的数据。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/MySQL/InnoDB-Buffer/normal-LRU.png)

在一般的LRU算法中，当链表中的某一个数据被读取时，将会将其放置于队首。当新增数据且链表已达最大数量时，将链表尾部的数据移除，并将新增的数据置于链表首部。

但是，InnoDB并没有采用传统的LRU算法，而是对其进行了一些更能够适应自身行为的改进: 最近访问到的数据并不直接放到LRU列表的首部，而是放到LRU列表的midpoiont位置。在默认配置下，midpoint位于LRU列表的5/8处。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/MySQL/InnoDB-Buffer/midpoint.png)

如上图所示，在midpoint之前的列表称之为new列表，或者使用Java中的GC术语: 新生代，在midpoint之后的列表称之为old列表，或者说，老年代。

当new列表中的数据被访问时，直接将其放置于LRU列表的首部。当出现新的页面进入LRU时，将其放置于midpoint位置，**此时该数据页将会位于old列表**。

当old列表中的数据被访问时，需要进行判断。如果当前页在old列表中存在时间超过了1秒，则将其移到列表首部。若存在时间小于1秒，则位置保持不变。该判断时间由`innodb_old_blocks_time`决定，默认为1000毫秒，即1秒。

另外一点需要注意的是，InnoDB管理数据的最小单位是页(page)，而不是数据库中的某一行。即如果两条记录处于同一页，在两次间隔时间超过`innodb_old_blocks_time`的不同行记录的访问也会将该页置于LRU的首部。

InnoDB如此设计的原因在于若使用朴素的LRU算法实现的，某些索引或者表扫描操作可能会将所有的索引页和数据页置换出去，而这些数据通常只是一次性使用的，热点数据被刷出之后，会严重的影响MySQL的性能。当采用midpoint实现后，至少能够保证5/8的数据都是热点数据，即使出现了大范围的表扫描和索引扫描。

让我们来具体分析下对大表进行顺序扫描的过程:

1. 当扫描开始时，InnoDB会一次性地取出16KB的一页数据，将其置于LRU 5/8的位置。
2. 由于是顺序扫描，那么同一页将会被访问多次，但是访问的间隔一定不会超过默认的`innodb_old_blocks_time`，即1秒。所以该页并不会被置于LRU列表的首部，也就不会将真正的热点数据置换而出。
3. 后续的数据扫描将会在下一页进行，重复上述过程。

通过执行`show engine innodb status\G;`可以查看InnoDB缓冲池的各种指标，包括当前缓冲池的大小。此外还有一个非常重要的性能指标: 内存命中率。

```bash
mysql> show engine innodb status\G;
Buffer pool hit rate 976 / 1000, young-making rate 0 / 1000 not 0 / 1000
```

其中Buffer pool hit rate即为缓冲池内存命中率，在一个线上服务中，如果要保证响应时间的话，命中率应不低于95%。 如果某台MySQL实例的hit rate低于此值的话，需要查看设置的缓冲池总大小，以及是否进行了频繁的大范围数据扫描导致LRU列表被污染。

除了增加缓冲池的大小来提高效率以外，当遇到持续的热点问题时，即预估将来的热点数据不止63%，也可以调整midpoint的值来减少热点数据被刷出的概率:

```bash
# innodb_old_blocks_pct默认值为37%, 即5/8
mysql> set global innodb_old_blocks_pct = 20;
```