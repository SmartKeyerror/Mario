---
layout: post
cover: false
navigation: true
title: 分布式系统基础学习(03)--消息队列(RabbitMQ)
date: 2019-01-11 09:52:09
tags: ['分布式系统']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---


消息队列常常被用于跨进程通信， 异步调用， 系统解耦以及流量削峰等场景下。 目前市场上的消息队列种类繁多， 有LinkedIn开源的`kafka`， 阿里开源的`RocketMQ`， 以及使用较为广泛的`RabbiMQ`。 `Redis`虽然也可以做消息队列， 但是更多的是用来做缓存和其它工具使用。 但是随着Redis 5.0 版本的发布， Redis的功能越来越丰富， 也不排除在未来的某一天也能够占据消息队列的重要一席。

<!---more--->

#### 1. 每种消息队列的特性以及适用场景
待填坑， `kafka`尚未实际使用过， 所以不贸然的评价。

#### 2. RabbitMQ架构形式

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-12-12%2017-01-57%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

与其它的消息队列有着明显不同的地方就是在`RabbitMQ`中有一个显式指定的组件： Exchange(交换机)。

如上图所示， 生产者将消息发送至某一个Virtual Host的某一个交换机， 交换机根据Routing Key将消息路由到相应的队列中， 然后由消费者进行消费。 可以看到消费者其实可以只和队列打交道， 完全屏蔽掉交换机。

那么Virtual Host， Exchange， Routing Key以及Queue又有什么作用呢？

Virtual Host其实就是一个字符串， 可以`/`， `/test`等等， 是一种逻辑隔离的手段。 如同Redis的db0, db1一样， 每一个Virtual Host中可以有多个交换机和队列， 但是不允许重名。

Exchang是`RabbitMQ`最为核心的组件， 从上图中可以看到交换机和队列是一种多对多的关系， 即一个交换机可以绑定多个队列， 一个队列也可以有多个交换机与其绑定。

Routing Key就是交换机和队列的绑定关系， 本质上就是一个字符串。 但是可以使用通配符的方式与交换机配合来达到广播的效果， 这一点在交换机的类型中将会详细说明。

Queue即为队列， 为消息实际存储的位置， 与`Redis`显著不同的是， 在`RabbitMQ`中的队列有自己的相关属性， 例如队列的过期时间， 是否持久化， 队列的最大长度等， 相关特性在队列一小节中也会详细说明。 此外， 队列是消费者直接连接并消费的地方。

这些组件构成了`RabbitMQ`的核心构成， 我们在日常进行开发和维护时， 也是和这些组件打交道， 所以理清每个组件是做什么的以及组件之间的关系是非常重要的。


#### 3. 交换机
在详细的解释交换机的种类之前， 首先看一下最基本的生产者代码：

```java
public class SimpleProducer {
    public static void main(String[] args) throws Exception{
        /* 创建连接工厂, 并设置IP, 端口以及Virtual Host参数 */
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("127.0.0.1");
        connectionFactory.setPort(5672);
        // 设置VirtualHost为/test, 这里可以是任意值， 前提是该VirtualHost已被添加
        connectionFactory.setVirtualHost("/test");

        Connection connection = connectionFactory.newConnection();
        Channel channel = connection.createChannel();

        /* 交换机类型 */
        String exchangeType = "direct";
        /* 交换机名称 */
        String exchangeName = "test_exchange";

        String routingKey = "routeApp";
        String queueName = "test_queue";

        /* 声明交换机, 类型为direct, 名称为test_exchange */
        channel.exchangeDeclare(exchangeName, exchangeType);

        /* 声明一个持久化的, 非独占的, 非自动删除且没有扩展参数的队列 */
        channel.queueDeclare(queueName, true, false, false, null);

        /* 将交换机和队列通过Routing Key进行绑定 */
        channel.queueBind(queueName, exchangeName, routingKey);

        /* 发送消息时, 可以只指定交换机名称以及Routing Key */
        channel.basicPublish(exchangeName, routingKey, null, "hello".getBytes());

        channel.close();
        connection.close();
    }
}
```

在我们声明了交换机和队列之后， 剩下的就是将交换机和队列通过Routing Key进行绑定， 在绑定完成之后， 就可以完全不管队列了， 因为通过交换机的名称以及Routing Key就能够找到所对应的队列。 在发送消息时， 我们也的确没有指定队列名称。

##### 3.1 direct交换机
`direct`交换机可以翻译成直接交换机， 当一个交换机为`direct`类型时， 消息至多会发送给一个队列。

![ | center](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-12-14%2018-39-23%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

我们对`test_exchange`这个交换机和`test_queue`这个队列使用了`routeApp`这个Routing Key进行了绑定， 由于交换机是`direct`类型的， 那么如果我们在发送消息时， 指定的路由键不是`routeApp`， 并且该交换机下没有与之对应的路由键的话， 该消息就会被丢弃。

也就是说， `direct`类型的交换机与Routing Key之间是一个完全绑定的关系， 在发送消息时所指定的路由键必须与之完全匹配才能够投递到队列中。

##### 3.2 fanout交换机
`fanout`, 译为扇出， 那么`fanout`类型的交换机将会把消息路由到所有与之绑定的队列上， 相当于一种无条件的广播。

```java
String exchangeType = "fanout";
String exchangeName = "test_exchange";
channel.exchangeDeclare(exchangeName, exchangeType);

/* 声明3个队列 */
channel.queueDeclare(queueName + "_01", true, false, false, null);
channel.queueDeclare(queueName + "_02", true, false, false, null);
channel.queueDeclare(queueName + "_03", true, false, false, null);

/* 将fanout交换机和3个队列以不同的Routing Key进行绑定 */
channel.queueBind(queueName + "_01", exchangeName, "fake_01");
channel.queueBind(queueName + "_02", exchangeName, "fake_02");
channel.queueBind(queueName + "_03", exchangeName, "fake_03");

/* 向test_exchange发送消息, 并指定一个从未出现过的Routing Key */
channel.basicPublish(exchangeName, "fake_key", null, "hello".getBytes());
```

在这里我们声明了3个队列， 并使用3种路由键将其与交换机进行绑定， 然后在发送消息时， 使用了一个非常随意的路由键， 消息依然能够以广播的形式路由到这3个队列中。

![ | center ](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-12-14%2019-10-05%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)


##### 3.3 topic交换机
`topic`交换机与`fanout`交换机很类似， 只不过`topic`交换机是以通配符来进行路由键匹配， 而`fanout`类型的交换机所有的路由键都可通过。

在进行交换机和队列绑定时， Routing Key可以传入一个模糊匹配词， 如`*`或者是`#`。 其中`*`表示匹配一个单词， `#`匹配任意单词(可以是零个)。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-12-17%2010-22-28%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

如上图所示， 我们一共声明了4种队列， 并使用4种不同的Routing Key与其进行绑定， 通常我们称之为这种绑定的关系为Binding Key。 生产端在进行消息产出时会指定一个Routing Key和交换机名称， 那么在这个时候所有与usa相关的信息均会被路由到第一个队列中。 所有与asia相关的信息都会被路由到最后一个队列中， 所有的新闻消息将会被路由到第二个队列中， 所有的天气信息将会被路由到第三个队列中。 这样一来就完成了信息的不同维度的分类。

对信息进行不同维度的分类是`topic`类型交换机的一个功能之一， 此外， 在死信队列这种特殊的交换机下， 通常也会用到模糊匹配的功能。

以上就是`RabbitMQ`所提供的三种类型的交换机， 尽管来讲可能`direct`类型的交换机是使用最为广泛的， 但是每一种类型都是有其所适用的场景的。

##### 3.4 声明交换机时的其它参数
在上面的demo中， 只是简单的使用交换机名称和交换机类型， 除此之外， 还有一些额外的有用的参数， 具有完整参数的函数如下：

```java
public DeclareOk exchangeDeclare(String exchange,
                                 String type,
                                 boolean durable,
                                 boolean autoDelete,
                                 boolean internal,
                                 Map<String, Object> arguments) throws IOException
```

前两个自然就是交换机名称和交换机类型， `durable`是一个布尔类型的参数， 表示交换机是否持久化， 若为`true`， 则该交换机将会被保存至磁盘中。 若`RabbitMQ`重启， 则该部分数据不会丢失。 `autoDelete`表示是否自动删除， 当该交换机与之绑定的队列均与其解绑时， 该交换机将会被自动删除， 通常来讲都会将其设置为`false`。 `internal`表示是否用于`RabbitMQ`内部， `arguments`则是一些拓展参数。


#### 4. 队列
在`RabbitMQ`中， 队列是保存消息的唯一载体， 生产者所生产的消息经过交换机均会路由到相应的队列， 或者是在Routing Key不匹配的情况下将消息丢弃。 相比于交换机， 队列的类型就只有一种， 队列就是队列。 所以直接来看在声明队列时可以传递的参数：

```java
public ....DeclareOk queueDeclare(String queue,
                                  boolean durable,
                                  boolean exclusive,
                                  boolean autoDelete,
                                  Map<String, Object> arguments) throws IOException
```

参数的数量并不是很多， 但是都很重要。 前两个是队列的名称以及是否持久化， 不再赘述。

`exclusive`表示是否排他， 是一个很容易用错的一个参数。 **如果一个队列被声明为排他队列， 那么该队列仅对首次声明它的连接可见， 并在连接断开时自动删除， 不管有没有指定其持久化**。 所以说， 同一个连接下的不同`Channel`， 是能够对访问该队列的， 也就是说， 如果一个队列是排他队列， 那么只有声明方可以发送和接收信息。 通常会用于"自娱自乐"这种情况下， 也就是自己发送消息， 自己接收消息。

`autoDelete`与交换机参数的`autoDelete`基本相似， 译为是否自动删除。 当一个队列的所有连接均断开时， 这个队列才会被自动删除， 而不是在没有连接的情况下被删除(如果是这样的话， 那么在声明队列之后， 没有客户端连接， 难道就直接删除该队列吗？)， 这点需要注意。

`arguments`为拓展参数， 队列的拓展参数就比交换机的拓展参数要丰富的多。

- x-message-ttl： 指定队列中所有消息在被路由到当前队列的存活时间。
- x-expires： 指定当前队列在不活跃时的存活时间， 有些类似于Java线程池的线程存活时间。
- x-max-length： 指定当前队列所能容纳的最大消息数量。
- x-dead-letter-exchange： 用于指定队列中有死信产生时， 将消息投放至哪个交换机。
- x-dead-letter-routing-key： 与上一个参数配合， 指定死信消息的Routing Key。
- x-max-priority： 指定当前队列的最大优先级。

这些参数在日常使用中经常会用到， 例如死信队列的相关参数和配置等。 死信队列的内容将会在下面进行描述。


#### 5. 消息
在介绍完`RabbitMQ`的交换机与队列之后， 那么剩下一个核心组件就是流转于这些组件的消息。 消息同样会有许多的属性， 例如优先级， 消息id， 过期时间等等。 一些常见的消息属性以及添加属性的方式如下：

- deliveryMode： 消息是否持久化， 2为持久化， 1为非持久化
- content_type： 消息内容的类型
- content_encoding： 消息内容的编码格式
- priority： 消息的优先级
- expiration： 消息于队列中的存活时间， 可以对某一条消息特别指定
- message_id： 消息id
- timestamp：消息时间戳

使用方式：

```java
AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder()
        .deliveryMode(2)
        .expiration("10000")
        .contentEncoding("utf-8")
        .build();
```


#### 6. 可靠消息投递
虽然`RabbitMQ`能够最大程度上的保证消息的可靠性投递， 但是由于网络情况的复杂性， 我们不得不再设计一个系统， 用于99.9%的保证消息的准确送达于消费。

这里给出两种常用的方式， 第一种方式实现较为简单， 但是并发性会有一些问题； 第二种方式要相对更加复杂一些， 但是有较好的并发性。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-12-12%2017-01-44%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

如上图所示， 主要的实现方式就是添加一个`message table`以及一个定时任务。 在一条消息发送之前， 将该消息以及当前的时间进行入库， 并给定一个状态值(`status`)， 例如已发送但未确认为0， 消息经由客户端并返回了当前消息的ACK之后， 更新该消息的状态值。 定时任务每隔1分钟或者3分钟扫描一次该table， 筛选出超时的消息， 并告知Producer重发该消息。

这里有一些细节需要注意， 发送消息的前提条件是业务数据以及消息入库正常。 生产端能够接收ACK， 那么就需要开启消息确认模式， 该模式将会在下文叙述。 此外， 重发消息一定要是定时任务通知Producer重发该消息， 因为对重发的消息也需要进行确认。

这种实现方式相对来说比较简单， 对于一条消息， 我们可以使用16位的时间戳进行唯一标记， 并在DB中对该字段加上`unique`索引来加速查找。 这种模式在几百或者几千的QPS下能够运行良好， 但是如果并发超过该值， 那么第二种方式效率更高。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-12-12%2017-01-34%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

从图示来看， 主要是采用了一个延迟信息投递的方式， 并且采用了2组交换机。 Producer发送2条消息， 一条为即时发送， 一条为延迟发送， 当客户端成功接收即时消息之后， 向Exchange2交换机发送一个当前消息的确认信息， 第三方服务接收该消息并将该消息入库。 一定时间后， 例如1分钟后， 延迟消息抵达至Exchange2， 同样地， 第三方服务接收该消息判断该消息是否存在于`message table`中， 若不存在， 则通知Producer重发该消息。

这种方式采用异步写库的方式来完成可靠消息的确认， 并且移除了定时脚本， 这样一来就省去了扫描`message table`部分区间的时间。 但是也带来了其它的复杂性， 例如延迟消息的实现， Exchange2的管理和维护等。


#### 7. 消息的ACK与重回队列
在上一小节的同步DB实现中， 需要用到消息的ACK。 消息的ACK分为2种， 自动确认和手动确认， 自动确认是指当消费者接收到了这条消息之后， 由`RabbitMQ`自动的回送ACK消息， 手动确认顾名思义就是需要显式的回送ACK消息。

生产端需要开启确认模式， 主要是针对`Channel`而言的：

```java
channel.confirmSelect();

channel.addConfirmListener(new ConfirmListener() {
    public void handleAck(long l, boolean b) throws IOException {
        // 其中l为deliveryTag, b为是否批量
        System.out.println("------ACK------");
    }

    public void handleNack(long l, boolean b) throws IOException {
        System.out.println("------NACK------");
    }
});
```

消费端在进行消费时， 通过传入布尔参数`b`来指定是否为自动签收， `true`为自动签收， `false`为手工签收。 当我们采用手工签收时， 在消费完消息之后， 需要显式地回复ACK：

```java
channel.basicAck(envelope.getDeliveryTag(),false);
// channel.basicNack();  回送一个NACK确认， 即否定确认
```

可能有人会说既然已经有了ACK确认消息， 为什么还需要消息入库这种可靠性的手段。 因为ACK消息也会丢失， 所以只能作为可靠消息投递的一个组件使用， 而不能完全依赖ACK消息。 在[通信(TCP/UDP)](https://smartkeyerror.com/%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E5%9F%BA%E7%A1%80%E5%AD%A6%E4%B9%A0-01-%E9%80%9A%E4%BF%A1-TCP-UDP.html)这篇文章中有介绍到为什么TCP连接需要序列号， ACK包以及校验和， 定时器来确保数据分组的可靠传输。 可以看到这些内容都是非常相似的， 或者说理念就是完全相同的， 除非有了绝对的可靠传输链路， 否则这些内容是不会过时的。


#### 8. 消息的限流
有时候消费端由于意外需要进行重启， 这个时候队列可能积压了大量的信息， 在没有限流的情况下重启消费端， 成百上千的消息一次性的涌入， 可能会造成服务器资源在短时间内用尽， 导致服务假死。 所以就需要有限流机制， 限流的设置也非常简单， 需要注意一点的就是此时**必须使用手工签收**。

```java
/* 自定义消费者类, 继承自DefaultConsumer, 并覆盖handleDelivery方法 */
class MyConsumer extends DefaultConsumer {

    public MyConsumer(Channel channel) { super(channel); }

    @Override
    public void handleDelivery(String consumerTag, Envelope envelope,
                               AMQP.BasicProperties properties, byte[] body)
            throws IOException {
        System.out.println("recv message: " + new String(body));

        // 限流时必须手工回送ACK消息
        channel.basicAck(envelope.getDeliveryTag(),false);
    }
}

// 每次最多只能有60条消息进行消费
channel.basicQos(60);

// 关闭自动签收， 并添加消费者
channel.basicConsume(QUEUE_NAME, false, new MyConsumer(channel));
```

#### 9. 死信队列
与`Redis`一样， `RabbitMQ`提供了类似监听Key过期的机制， 只不过实现方式要稍微复杂一些。 在`Redis`中， 我们可以设置Key过期的发布订阅， 当Key过期时， 订阅方能够接受到该Key， 但是无法拿到Value， 这是我比较疑惑的地方， 如果能拿到Value， 那么这个功能就相当出色了。

回到`RabbitMQ`， 死信队列其实是指当某一个消息在成为"死信"时， `RabbitMQ`会将该消息发送至所绑定的"死信交换机"中， 由该交换机路由至某一个队列。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-12-17%2015-43-23%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

所以说， 如果我们想要使用死信队列的话， 就需要有额外的交换机， Routing Key以及队列。 当然， 队列和Routing Key可以有多个， 完成死信消息的分拣功能。

那么消息在什么情况下会变成死信呢？

- 消息被拒绝(basic.reject/basic.nack), 并且requeue为false
- 消息TTL过期
- 队列达到最大长度或队列空间已满

在这些情况下队列中的消息将会成为死信， 如果在声明队列时添加了如下设置， 那么该队列就会成为死信队列。

```java
Map<String, Object> arguments = new HashMap<String, Object>();
arguments.put("x-dead-letter-exchange", deadLetterExchange);
```

说白了， 死信队列就是当某一个消息变成死信时， `RabbitMQ`将会根据我们的设置将该消息重新投递到所指定的交换机中。 通过使用死信队列， 我们就可以实现延迟发送的功能。


#### 10. 小结
在本篇文章中， 介绍了`RabbitMQ`消息中间件的基本使用以及一些高级功能， 例如消息的限流， 死信队列等。 在可靠消息投递的设计中充分的借鉴了TCP协议的可靠数据传输模型， 针对于并发这个因素制定了2种不同的解决方案。

除此之外， 消息队列的高可用的拓扑结构以及运维相关的内容， 将会在后续文章中进行讨论。 最后附一张思维导图， 便于记忆与整理。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-12-17%2016-02-38%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)


