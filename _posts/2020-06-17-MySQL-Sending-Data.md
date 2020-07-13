---
layout: post
cover: false
navigation: true
title: MySQL向客户端发送数据，客户端不接收会发生什么?
date: 2020-06-17 10:50:25
tags: ['MySQL']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

MySQL服务端在发送数据时，为了减少数据在用户空间和内核空间的复制次数，往往会使用缓冲区对数据进行缓冲。那么，如果客户端在接收大量数据时，选择不接收，或者处理非常慢的时候，会影响MySQL的正常运行吗?

<!---more--->

### 1. TCP连接的收发模型

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/MySQL/MySQL-Send-Buffer/TCP-buffer.png)

当调用`write()`或者`send()`系统调用向socket写入数据时，并不会直接被发送至网络，而是被发送至位于内核空间的TCP发送缓冲区。在缓冲区中，由TCP连接的窗口控制发送频率和数量。当发送缓冲区已满时，阻塞模式下的`write()`调用将会一直阻塞，直到有可用的发送缓冲区为止。而非阻塞模式下则会直接返回-1，`errno`将会被置为`EAGAIN`或者是`EWOULDBLOCK`。

对于`read()`或者是`recv()`系统调用读取socket数据时，情况和发送数据基本类似，只不过是读取接收缓冲区中的内容。阻塞模式下如果接收缓冲区为空，那么将会阻塞，而非阻塞模式下则会立即返回，`errno`为`EAGAIN`或者是`EWOULDBLOCK`。

接收方与发送方的窗口大小、MSS大小以及网络状况都会对发送缓冲区的动态大小变化造成影响。当接收方的缓冲区较小时，也就意味地接收窗口较小，那么当发送方持续发送数据时，很有可能将发送缓冲区填满导致发送方数据写入的阻塞。同样地，如果发送方的网络出现波动导致大量的丢包，由拥塞避免阶段重新进入慢启动阶段，也会导致发送缓冲区的数据不能及时地发出。

在一般的Linux操作系统下，发送缓冲区和接收缓冲区的的大小默认为208K。

```bash
smartkeyerror@Zero:~$ cat /proc/sys/net/core/wmem_default
212992
smartkeyerror@Zero:~$ cat /proc/sys/net/core/rmem_default
212992
```


### 2. MySQL结果发送与客户端接收

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/MySQL/MySQL-Send-Buffer/MySQL-Send.png)

当客户端向MySQL请求查询数据时，MySQL会将结果暂存于net_buffer中，net_buffer的大小默认为16K，由参数`net_buffer_length`决定。当net_buffer已满或者是无更多结果时，调用网络接口将数据写入至TCP连接的发送缓冲区中。如果发送缓冲区已满，那么该查询请求的数据发送将会阻塞，直到有可用的发送缓冲区为止。

当TCP发送缓冲区已满时，通过`show processlist`将会得到Query语句"正在发送给客户端"的结果:

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/MySQL/MySQL-Send-Buffer/Sending-To-Client.png)

当出现State为`Sending to client`的查询语句时，表示当前线程已将TCP发送缓冲区填满，无法继续发送数据给客户端，并等待客户端接收数据。

如果出现了较多的`Sending to client`状态的线程，那么要么是客户端网络情况较差，要么是客户端在处理结果时过慢，此时需要优化客户端代码。

`Sending to client`状态需要和`Sending data`状态区分开来，前者表示正在等待客户端接收结果，而后者则表示事务正在执行(不一定在发送数据，也可能在等待锁)。

### 3. 客户端不接收MySQL的发送数据会发生什么?

#### 3.1 InnoDB的一致性非锁定读

在InnoDB存储引擎中，每条UPDATE语句都会默认的在该TABLE上添加一个意向写锁(IX Lock)，并且在要修改的数据行上添加X Lock，防止其它线程对其同时更新。而在这个过程中，普通的数据读取操作依然能够进行，并不会等待X Lock的释放，极大的提升了数据库的并发读取能力，该读取方式称为一致性非锁定读，由undo段实现。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/MySQL/MySQL-Send-Buffer/undo-read.png)

在undo段中，同一行数据可能因为事务并发的执行而导致出现多个版本的快照，在一致性非锁定读中，只需要选择其中一个快照数据进行读取即可。在RC隔离级别，总是读取最新版本的快照数据，而在RR隔离级别下，总是读取事务开始时的行数据版本。

#### 3.2 `Sending to client`导致undo段迅速膨胀

当服务端出现大量的线程处于`Sending to client`状态时，所执行的查询语句未完成，那么会导致长事务的产生。而如果此时数据更新较为频繁时，将会导致undo段空间迅速膨胀，因为长事务的进行导致undo页无法被回收。

最常见的例子就是使用`mysqldump`对数据进行热备时，备份存储节点的磁盘已满，导致`mysqldump`无法向备份文件中写入数据。数据由客户端的接收缓冲区开始堆积，到服务端的发送缓冲区，再到`net_buffer`，最终可能会导致undo段的数据堆积，使得MySQL服务出现大面积的异常。


### 4. 小结

综上，客户端如果在处理MySQL发送的大量数据时，应该尽可能地将其暂存在本地的某个缓冲区中，而后应用程序想怎么处理、以何种速度处理都没问题，避免让MySQL产生过多的长事务。