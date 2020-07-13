---
layout: post
cover: false
navigation: true
title: TCP有限状态机
date: 2020-06-09 18:06:25
tags: ['Network']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

相较于Linux进程状态的变迁，TCP的状态变迁则会复杂许多，当然这与TCP本身的实现有关。当线上的Web服务或者是基于TCP连接的服务出现了时断时续的网络状况时，往往需要通过`tcpdump`以及TCP连接的状态进行问题定位。同时，这一复杂的有限状态机设计也能够为业务的设计提供指导性的帮助。

<!---more--->

### 1. TCP连接的建立

TCP连接本质上是两个进程的某些属性的状态值，在TCP连接的建立和断开过程中，实际是状态的变更。连接的建立需要经过三次握手，确认连接的双方能够正确应答，同时进行`MSS`、`Win`等信息的交换。

TCP协议是可靠传输协议，所以存在TCP序列号以及确认机制。序列号是指当前传输方向上数据报第一个字节的逻辑偏移，由于TCP连接是双向的，所以连接的双方都会有各自的序列号，记为Sequence Number，简写为Seq。

确认机制则由位于TCP报文段中大小为1bit的ACK确认位以及确认序号实现。当一方收到了Seq为$s$、Len为$l$的报文后，需要向对方确认已经收到了该报文。确认的方式是设置ACK确认位，并且将确认序号设置为$s+l$，表示下一次想要接收的数据字节序列号。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/Network/TCP-Status/TCP-build.png)

如上图所示，TCP连接的建立需要进行三次握手，其中的细节不再赘述。当客户端使用SYN发起连接时，其状态将会由`CLOSED`变迁为`SYN_SENT`，表示进程主动地打开了一个连接，并等待对端回应以此完成连接的建立。

当服务端收到来自客户端带有SYN控制位的数据包时，将回复带有SYN以及ACK控制位的数据包，并将状态置为`SYN_RECV`，等待对端发送的ACK数据包以建立连接。

客户端收到服务端的报文以后，将连接状态改变为` ESTABLISHED`，表示与对端TCP节点间的连接建立完成，可以正式地传输数据了。同样地，服务端收到对端的ACK确认包以后，也会将状态改变为`ESTABLISHED`，两者的连接正式建立。

在连接建立时，只有在确认对方能够正确应答(SYN发送且收到ACK应答)时才会进入`ESTABLISHED`状态。


### 2. TCP连接的拆除

TCP连接的建立相对来说比较简单，其状态变迁也相对较少，而连接的拆除则远比连接的建立复杂的多。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/Network/TCP-Status/TCP-disconnect.png)

发起连接断开的一段称为主动断开方，那么另一端则称为被动断开方。主动断开方执行`close()`系统调用，断开TCP连接，此时主动方将发送设置了FIN控制位的报文，并进入`FIN_WAIT_1`阶段，等待对方对该FIN包的应答。

当被动断开方收到FIN包以后，立即回送该FIN的确认包，并进入到`CLOSE_WAIT`阶段，**主要的工作就是等待上层应用主动调用`close`方法关闭连接**，在该阶段中上层应用任何的`read()`系统调用都将返回0，表示文件结尾。当应用层调用了`close()`系统调用之后，被动断开方将发送FIN结束包。

主动断开方收到FIN包以后，同样地回送ACK确认报文，并进入`TIME_WAIT`阶段，此时将固定的等待2倍MSL时间，MSL为报文的最大生存时间，Linux中默认的MSL时间为60S，那么主动断开方将等待120S后彻底关闭TCP连接。

被动断开方在收到ACK确认包以后，释放内核资源，完全关闭TCP连接。


#### 2.1 CLOSE_WAIT

`CLOSE_WAIT`状态只有在被动断开方才会出现，其过程的长短并不由内核控制，必须等待上层应用程序主动调用`close()`系统调用，才会从此状态变迁为`LAST_ACK`状态。所以，由于此阶段需要应用程序的主动参与，该阶段也是最容易出现问题的阶段。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/Network/TCP-Status/CLOSE_WAIT.png)

当服务器出现了大量的、长时间的`CLOSE_WAIT`状态的连接，就需要判断是否是应用程序存在BUG，导致TCP连接未主动地关闭。或者是锁争抢过于激烈，又或是CPU资源不足导致应用程序没有额外的CPU对连接进行处理。

#### 2.2 TIME_WAIT

`TIME_WAIT`阶段往往会被开发者所误解当做是优化的对象，因为该阶段的超时时间为2倍的MSL，开发人员认为该值过大，通常会将其减少至15S或者更短。但是，`TIME_WAIT`阶段的存在主要目的在于实现连接的可靠终止，以及让原有报文段在网络中过期失效，不会发送给新的连接。

##### 2.2.1 连接的可靠终止

在TCP四次挥手的过程中，首次发送FIN报文和最后一次发送ACK报文的都是主动断开方，被动断开方被"夹在中间"。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/Network/TCP-Status/NOT-TIME-WAIT.png)

如上图所示，假设没有`TIME_WAIT`阶段，并且主动断开方的最后一个ACK在网络中丢失。被动断开方迟迟等不到自己FIN包的ACK，当达到最大超时时间时将重传FIN包(此时无法触发快速重传)。但是，由于此时主动断开方的状态已经是`CLOSED`，即当前连接并不存在，则会返回一个RST，而该RST则会被视为错误。

##### 2.2.2 确保老的重复的报文在网络中过期失效

TCP的重传算法可能会导致生成重复的报文，并且根据路由的不同选择，这些重复的报文可能会在连接终止之后到达。

假设主动断开方在发送最后一个ACK包以后立即进入`CLOSED`状态，并且在该段又新建了一个与之前一样的连接(IP地址和端口号相同)，那么此连接就是原来连接的化身。在这种情况下，TCP必须确保上一次连接中老的重复报文不会在新的连接中被当成合法数据接收。当有TCP结点处于`TIME_WAIT`状态时是无法通过该结点创建新的连接的，这样就阻止了新连接的建立。

当一条连接处于`TIME_WAIT`阶段时，其向内核申请的端口号并不会得到释放。如果系统中同时存在大量的处于`TIME_WAIT`阶段的连接的话，可能无法再建立新的连接: 端口号资源不够。所以某些并发量较高的应用程序会选择缩短`TIME_WAIT`的时间，已获得更高的并发量。

但是，当系统因为`TIME_WAIT`而无法建立新的连接时，表示当前节点的资源已经吃紧，最好的办法是增加机器，而不是缩短`TIME_WAIT`的时间。


### 3. 使用netstat查看套接字状态

`netstat`可以显示出系统中Internet和UNIX域套接字的状态，当服务器出现网络问题时，可首先用此命令获取基本情况。

```bash
smartkeyerror@Zero:~$ netstat 
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 localhost:33406         localhost:7890          TIME_WAIT  
tcp        0      0 Zero:56626              183.61.83.4:https       ESTABLISHED
tcp        0      0 localhost:33528         localhost:7890          ESTABLISHED
tcp        0      0 Zero:54438              119.147.134.30:https    TIME_WAIT  
tcp        0      0 Zero:32856              220.181.107.131:https   ESTABLISHED
tcp        0      1 Zero:60558              media-router-fp1.:https LAST_ACK
```

- proto: 表示套接字所使用的协议，例如tcp、udp和unix。
- Recv-Q: 表示套接字接收缓冲区中还未被本地应用读取的字节数。对于UDP套接字
来说，该字段不只包含数据，还包含UDP首部及其他元数据所占的字节。
- Send-Q: 表示套接字发送缓冲区中排队等待发送的字节数。
- Local Address: 本地套接字所绑定的地址，格式为IP+端口号。
- Foreign Address: 对端套接字所绑定的地址。
- State: 套接字所处的状态。

通常来说我们并不关心UNIX域套接字的相关信息，所以可以使用`--tcp`进行过滤，得到的结果仅包含TCP套接字的相关信息。还有一些其他有用的选项:

| 选项   | 描述                                         |
| ------ |-------------------------------------------- |
| -a     | 显示所有套接字的信息，保证正在监听(LISTEN)的套接字 |
| -c     | 每隔一秒钟刷新显示套接字信息                     |
| -l     | 仅显示正在监听的套接字信息                      |
| -p     | 显示进程 ID 号以及套接字所归属的程序名称          |
| --tcp  | 显示 Internet域TCP(流)套接字的信息             |
| --udp  | 显示 Internet域UDP(数据报)套接字的信息          |
| --unix | 显示 UNIX 域套接字的信息                       |

