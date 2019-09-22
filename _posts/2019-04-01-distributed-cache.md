---
layout: post
cover: false
navigation: true
title: 分布式系统基础学习(05)--分布式缓存设计
date: 2019-04-01 10:17:46
tags: ['分布式系统']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

在单机缓存中， 并发的安全性问题与语言的并发安全问题完全可以归为一类， 缓存的穿透问题可以采用巧妙的数据结构进行处理， 很多问题本质上仍然是一些基础问题。

<!---more--->

#### 1. Cache Aside单机缓存模式
在业务应用中， Cache Aside是最常用的缓存模式。 其主要逻辑为当请求未命中缓存时， 从DB中取出数据并将其置于缓存中， 当数据发生更新时， 删除该数据所对应的缓存。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/distributed-cache/cache-aside.png)

以Django框架为例， 其伪代码为:

```python
redis = Redis("127.0.0.1", 6379)

class SomeView(APIView):
    def get(self, request):
        # 尝试获取缓存
        response_data = redis.get("some_key")
        # 缓存未命中
        if not data:
            try:
                original_data = SimpleModel.objects.get(id=15)
                data = SimpleModelSerializers(original_data).data
                response_data = {"code": 0, "message": "success", "data": data}
                # DB数据写入缓存， 并给予15分钟的过期时间
                redis.set("some_key", json.dumps(response_data)， 15 * 60)
            except Exception as e:
                # 标准错误处理流程
                logger.exception(e)
        return Response(response_data)


class SimpleModel(models.Model):
    name = models.CharField(max_length=15)

    def save(self, force_insert=False, force_update=False, using=None,
             update_fields=None):
        # 重写Model.save方法, 在数据更新时删除失效缓存
        redis.delete("some_key")
```

这看起来似乎很简单， 而且Cache Aside模式能够最大程度的减少由于并发所带来的脏数据问题， 但是不能完全避免脏数据问题。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/distributed-cache/cache-aside%E8%84%8F%E6%95%B0%E6%8D%AE%E9%97%AE%E9%A2%98.png)

如上图所示， 更新请求和查询请求先后发出， 由于更新操作需要进行表单验证等步骤， 操作时间要长一些， 在还没有删除掉失效缓存之前， 查询请求就从缓存中取到了脏数据并返回了。 这种情况出现的概率比较低， 受到影响的也仅仅只有紧跟更新请求的几个查询操作。 尽管如此， 仍然需要对这种情况进行处理， 目前比较好的实现就是为缓存添加过期时间。

#### 2. 高并发下带来的缓存问题
仍然是使用Cache Aside模式进行缓存的设计， 考虑这样一个场景: 两个查询请求并发执行, 并且此时缓存中没有对应的数据， 那么两个查询请求很有可能会将缓存数据重复写入， 如下图所示:

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/distributed-cache/%E7%BC%93%E5%AD%98%E9%87%8D%E5%A4%8D%E5%86%99%E5%85%A5.png)

2个查询请求并发执行， 缓存数据很有可能被重复写入2次。 那么N个查询请求并发执行， 缓存数据也有可能被重复的写入N次。

对于以json数据格式作为value的缓存数据来说， 重复写入问题也不大， 无非是将前一个结果覆盖了而已。 但是对于列表对象而言， 重复写入的问题就不得不去处理了。 列表的`lpush`或者是`rpush`操作并不会覆盖原有的数据， 而是直接追加， 这样一来就会造成严重的缓存数据重复问题，  并且多次的DB查询也会对系统整体的吞吐量造成影响。

限制资源的请求速率以及保证资源的唯一使用， 该怎么做？ 加锁。 在Python或者是Java语言层面， 为了保证操作的原子性以及并发安全性， 通常都会使用各种各样的互斥锁， 那么在这里也不例外， 只不过此时必须使用分布式锁。 因为Web服务要么是多进程多线程并行运行， 要么是多服务器组成的集群运行， 操作层面都在进程这一层， 只能使用分布式锁。

分布式锁的基本思想也很简单， 多个进程在对某个资源进行修改时， 先向第三方服务申请一下， 申请通过了才能用， 没通过就等着(轮询)。 这里无意扩展分布式锁的内容， 所以就简单的使用Redis的`setnx`命令实现。 此时我们只需要简单的修改一下设置缓存的部分代码即可:

```python
# 从DB中获取数据并对其进行序列化操作
original_data = SimpleModel.objects.get(id=15)
data = SimpleModelSerializers(original_data).data
response_data = {"code": 0, "message": "success", "data": data}


if redis.set(name="some_key_lock", value="1", ex=1, nx=True):
    try:
        redis.set("some_key", json.dumps(response_data)， 15 * 60)
    except Exception as e:
        logger.exception(e)
    finally:
        # 虽然对分布式锁添加了1秒的过期时间, 但是为了提高系统吞吐量, 在这里手动删除该锁
        redis.delete("some_key_lock")
```

是不是这样就可以了？ 并不是， 这样写在某些情况下仍然会出现问题。 我们把两个查询操作的时间稍微错开几十毫秒， 就有可能出现下图的情况:

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/distributed-cache/%E7%BC%93%E5%AD%98%E9%87%8D%E5%A4%8D%E5%86%99%E5%85%A52.png)

缓存数据依然被写入了两次。 其实这个问题在很多的并发场景下都会有出现， 不单单只是缓存的设计。 例如不采用枚举类所实现的单例模式， 在文章[Java基础编程(04)--常用的设计模式(01)](https://smartkeyerror.com/Java%E5%9F%BA%E7%A1%80%E7%BC%96%E7%A8%8B-04-%E5%B8%B8%E7%94%A8%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F-01.html)中采用了双重校验锁的方式解决此类问题:

```java
public static  SingletonClass getSingletonClass() {
    /* 第一次校验是让实例已经被初始化之后直接返回 */
    if (singletonClass == null) {
        /* 如果此时singletonClass == null, 那么就需要线程安全的实例化对象 */
        synchronized (SingletonClass.class) {
            /* 再次判断, 此时为加锁判断, 保证变量不会被其它线程所修改, 即保持单例*/
            if (singletonClass == null)
                singletonClass = new SingletonClass();
        }
    }
    return singletonClass;
}
```

在这个问题中， 依然可以使用同样的方式来处理， 即在获取分布式锁之后， 更新缓存之前， 再进行一次判断。

```python
if redis.set(name="some_key_lock", value="1", ex=1, nx=True):
    try:
        # 再次判断缓存数据是否不存在
        if not redis.get("some_key"):
            redis.set("some_key", json.dumps(response_data)， 15 * 60)
    except Exception as e:
        logger.exception(e)
    finally:
        redis.delete("some_key_lock")
```

#### 3. 缓存穿透问题
缓存穿透是指查询一个根本不存在的数据， 缓存层和存储层都不会命中， 而在通常情况下， 空数据是不会做缓存的， 基于Restful-API来讲， 此时应该直接返回404。 这样一来， 大量的无效请求都会透到DB存储层， 会给存储层带来比较大的压力。

这个问题的解决办法还是蛮多的， 最简单的就是缓存空数据。 依然使用上面的代码， 目光主要聚集在无效数据的处理上:

```python
try:
    original_data = SimpleModel.objects.get(id=15)
    data = SimpleModelSerializers(original_data).data
    response_data = {"code": 0, "message": "success", "data": data}
    # DB数据写入缓存， 并给予15分钟的过期时间
    redis.set("some_key", json.dumps(response_data)， 15 * 60)
except Exception as e:
    # 标准错误处理流程
    logger.exception(e)
```

在Django中， `Model.objects.get`操作在数据不存在时会抛出`DoesNotExist`的异常， 此时就可以在异常处理中将空数据进行缓存。 遗留的问题就是如果缓存中空数据非常多的话， 非常占用服务器内存， 而且这些key是能够无限增长的。 比如网站攻击， 假如用户id最大值为1000， 而攻击方生成10亿个大于1000的id进行请求， 并限制请求速率以及使用代理服务器。 那么一段时间后服务器就会有10亿个无效数据key， 这时候内存崩没崩都不好说， 系统运行效率一定是降低的。

为这些key设置一个较短的过期时间(比如5秒)能够解决一部分问题， 但是总的来说还是会浪费一部分内存空间。

另一个解决方案就是使用Bitmap。 将所有存在的key通过哈希或者其它算法写入到Bitmap数组中， 作为第一道缓存过滤器。 当请求无效数据时， 发现Bitmap中没有这个key(时间复杂度为O(1))， 直接返回空结果即可。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/distributed-cache/Bitmap%E8%BF%87%E6%BB%A4.png)

只不过这种方式维护起来比较费劲， 因为需要保存所有的有效key， 如果是大规模集群缓存的话， 其复杂度以及维护成本都会相应增加。


#### 4. 雪崩问题
缓存雪崩问题是指当缓存层为存储层分担了绝大部分压力时， 缓存层因为服务器宕机， 网络连接异常等问题导致的崩溃， 使得所有请求全部压向存储层的现象。 此时存储层由于大量的查询而造成线程数量飙升， 连接数飙升， 内存和CPU使用率飙升， 很有可能发生系统崩溃， 导致服务大面积停机。

雪崩问题没有办法从代码层面去很好的解决， 只能通过高可用设计处理。 例如Redis-sentinel高可用架构， MySQL高可用架构等等， 保证系统能够及时、自动地切换节点。

#### 5. 复制
在分布式系统中， 由于种种原因， 例如机房故障， 负载均衡， 读写分离等， 需要将数据复制到多个副本部署到其它的机器上， 因此， Redis也提供了主从复制功能。

与MySQL的主从复制相比， Redis的复制功能要简单许多。 通常在主从复制的模型下， 如何发现和处理主从数据的延迟， 以及主/从节点的身份转换， 是需要我们去着重处理的。

##### 5.1 建立主从复制过程
Redis建立主从复制的方式有多种， 可以在从节点配置文件中进行配置， 也可以在从节点的客户端中进行配置。 命令只有一个:
```bash
slaveof masterHost masterPort
```

##### 5.2 断开复制
断开复制的命令为`slaveof no one`， 在从节点执行完该命令之后， 复制过程终止， 此时从节点仍然会保留原有的数据， 但是无法获取主节点的数据变化。

当我们在一个从节点断开复制之后， 可以与另一个节点建立主从复制的关系， 这个过程称为"切主操作"。 如果一个从节点与当前主节点断开复制关系， 与另外一个节点建立复制关系的话， 此时从节点的数据将会被完全清空。

举个不恰当的例子， 某一天你在网上认了一个干妹妹， 跟她分享了很多有趣的事情。 突然有一天她不想做你妹妹了， 单方面切断了这个联系， 并删除了你的微信， 所以你更新的朋友圈她是不知道的。 然后她又找了一个新的干哥哥， 抛弃了与你所有的记忆(扎不扎心， 老铁)。

所以， 在生产环境的切主操作要慎重进行， 避免因操作不当带来的数据损失。

##### 5.3 复制过程
Redis主从复制过程大致可以分为:
1. 从节点保存主节点信息
2. 从节点内部的定时任务发现新的主节点， 尝试与主节点建立连接
3. 从节点发送PING命令， 检测网络是否正常连接， 主节点是否可用
4. 权限验证
5. 首次同步时进行全量数据复制
6. 数据持续复制

当主节点需要密码登录时， 从节点必须设置`masterauth`配置项进行密码登录。 下面贴一个在建立复制时主节点的日志记录:

```bash
1060:M 16 Mar 16:05:17.727 * Slave 127.0.0.1:6380 asks for synchronization
1060:M 16 Mar 16:05:17.727 * Partial resynchronization not accepted: Replication ID mismatch (Slave asked for '695874fc4ce12a5de99170a5751f57adf33cc032', my replication IDs are '8826a80c2972c469cb65688c899b07ae249f6905' and '0000000000000000000000000000000000000000')
1060:M 16 Mar 16:05:17.727 * Starting BGSAVE for SYNC with target: disk
1060:M 16 Mar 16:05:17.727 * Background saving started by pid 1570
1570:C 16 Mar 16:05:17.729 * DB saved on disk
1570:C 16 Mar 16:05:17.729 * RDB: 0 MB of memory used by copy-on-write
1060:M 16 Mar 16:05:17.771 * Background saving terminated with success
1060:M 16 Mar 16:05:17.771 * Synchronization with slave 127.0.0.1:6380 succeeded
```

首先就是从节点127.0.0.1:6380要求进行数据同步， 然后验证从节点的Replication ID， 来判断从节点是部分数据复制还是全量数据复制， 由于这是第一个建立复制， 所以必然是全量复制。 而后执行`BGSAVE`操作， fork子进程生成dump.rdb文件。 从日志上可以看出， 此时RDB文件是保存在磁盘中的， 并不是直接发送给从节点。 然后， 主节点通过网络传输， 将RDB文件发送给从节点， 从节点读取并写入数据， 复制工作就此开始。

当主节点的子进程开始执行BGSAVE操作时， 主节点仍然会处理写请求。 那么这部分的数据该如何处理？ Redis主节点会建立复制缓冲区， 这一段时间的数据更改都会写入复制缓冲区中， 当从节点加载完RDB文件数据之后， 主节点再将复制缓冲区的内容发送给从节点。 这样一来， 就能够保证数据的完整性。

如果主节点创建和传输RDB的时间过长， 对于高流量写入场景非常容易造成主节点复制缓冲区溢出， 默认配置为`client-output-buffer-limit slave 256MB 64MB 60`， 如果在60秒内缓冲区持续大于60MB或者超出了256MB， 主节点将主动关闭与从节点的连接， 全量复制终止。 所以， 开启从节点请选择月黑风高的凌晨。

##### 5.4 复制延迟
在主节点和从节点分别执行`info replication`命令可以查看此时数据复制的偏移量。 主节点为`master_repl_offset`， 从节点为`slave_repl_offset`， 单位为字节量。 使用

```bash
master_repl_offset - slave_repl_offset
```

能够很轻松的计算出主从复制的延迟。 当这个延迟值很高的时候， 例如20MB， 此时应用程序就需要做出反应， 暂时性的从主节点读取数据， 当延迟降低之后， 再从从节点读取数据。


#### 6. Redis Sentinel
哨兵是由Redis官方所提供的高可用架构解决方案， 实现了Redis数据节点的监控， 通知以及自动化的故障转移机制。

##### 6.1 为什么需要高可用架构
在一个中型服务中， Redis实例的数量可能不会特别多， 拓扑结构可能为1主1从或者是1主2从。 从库主要用于数据的读取， 主库用于数据的写入， 进行读写分离。 如果此时主节点发生宕机， 那么从节点与主节点断开连接， 复制终止， 并且应用层连接不上主节点， 无法进行数据写入， 造成服务部分功能无法使用。

此时要做的就是将一个从节点设置为主节点， 另一个从节点进行切主操作， 并且需要修改应用的代码， 将Redis主节点ip重新修改。 这样一套流程下来， 如果顺利的话， 也需要15分钟左右。 如果不顺利， 花费的时间将会更久。 并且人为操作还有可能出现错误， 导致数据丢失。

##### 6.2 Redis Sentinel高可用性
![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/distributed-cache/Sentinel.png)

如上图所示， Redis Sentinel是一种分布式多机架构， 其中包含了若干个Sentinel节点以及Redis数据节点。 这里的Redis数据节点表示主节点和从节点所组成的集合。 没一个Sentinel节点都会去监控数据节点和Sentinel节点， 当发现服务不可用时， 会对其做下线标识。 如果被标记的是主节点， 那么Sentinel节点将会和其它Sentinel节点进行投票， 当大多数节点都认为主节点不可达时， 它们会推举出一个Sentinel节点来完成自动的故障转移， 并将信息通知给应用服务。

同样以上图为例， 当Master节点不可用并且多数Sentinel节点确认了这个事实， 并且推举Sentinel-2来进行故障转移。 Sentinel-2随机的选取一个从节点作为新的主节点，例如Slave-1， 对其发送`slaveof no one`命令， 终止与原有主节点的复制， 并升级为新的主节点。 接着对Slave-2节点发送`slaveof newHost newPort`命令， 使其从新的Master节点进行数据复制。


#### 7. 如何对数据进行分区
在单机缓存模式下， 当我们处理了并发请求时的数据安全性， 解决了缓存穿透以及雪崩问题， 并且对缓存中big-key进行了足够好的优化之后， 剩下面临的问题就是单机内存容量的限制。

首先考虑一个更加具体化的问题， MySQL单表的最佳容量约为1000万数据， 也就是说， 假设有1亿数据， 那么最佳的分表方式就是将其拆分成10个表及以上。

##### 7.1 基于关键字区间

一个最简单的算法就是基于关键字区间进行分区， 例如`user_id`。 `user_id`在0-1000的写入表1， `user_id`在1001-200万的数据写入表2， 以此类推。 这样以来能够最大程度的维护单表数据相关性， 其缺点就是为了更均匀的分布数据， 开发人员需要找到适合数据本身分布特征的分区边界。 根据2/8法则， 贡献80%数据的用户只占所有用户的20%。 所以边界如何选取， 是基于关键字区间方法的重要因素。

##### 7.2 取模算法分区

另一个分区算法就是根据key进行取模。 如果拆分10个表， 就使用`user_id`对10进行取模， 再存储到数据对应的表中。

取模算法一个非常致命的问题就在于水平拓展非常复杂， 只能进行垂直拓展。 在项目设计之初， 开发人员预计某一个表中的数据最多能够到达1亿， 于是拆分了20个分区表， 这样一来系统最大能够存储2亿数据。 但是系统上线后用户量日益剧增， 很早的就达到了2亿数据。 此时再对数据进行取模， 单表存储将会超过1000万。 并且， 由于数据采用取模的方式进行存储， 如果增加分区表的话， 势必会打乱原有的存储结构， Web服务也有可能停机进行数据迁移。

解决这个问题也很简单， 在最初设计时， 就对分区表的数量取一个较大值， 例如100。 按照单表1000万的存储， 整体存储数据量为10亿。 我相信对于绝大部分应用而言， 10亿行数据的存储量， 是完全足够的， 再加上目前SSD不值钱， 即使是1TB的固态， 也只需要1000块。 总成本在1万以内可以搞定。

取模算法最大的优点就在于简单， 易操作和维护， 缺点就是水平拓展困难并且数据的分布均匀性较难保证， MySQL的简单分表方式选择取模算法是一个比较好的方法， 但是对分布式缓存来说， 其中会有一些问题出现。

假设目前有10台Redis缓存服务器， 编号0~9， 并使用了取模方式在这10台机上进行了数据缓存。 突然3号机挂掉了， 原本属于3号机的数据服务只能被迫转移到其它机器， 例如1号机。

```python
if key % 10 == 3:
    # 转至1号机服务
```

现在1号机也挂了， 开发人员又不得不再去整理规则， 属于1号机的服务转移至5号机...来来回回， 数据在整体集群中非常混乱， 取模的方式很难建立一个自动化故障转移的机制来处理突发情况。

##### 7.3 一致性哈希算法分区
一致性哈希算法能够最大程度上自动的处理数据分布不均匀问题， 并且能够提供自动化的故障转移机制。

通常我们会设计一个处理字符串的32位哈希函数， 当输入某个字符串时， 它会返回一个0和2^32 - 1之间近似随机的数值。 及时输入的字符非常相似， 返回的哈希值也会在上述数字范围内均匀分布。

既然范围限定在0～2^32-1之间， 那么对于一个哈希值， 只能取前4个字节， 这里取md5加密的前4个字节:

```python
def get_result(key)
    return int(hashlib.md5(str(key).encode("utf-8")).hexdigest()[:8], 16)
```

将需要缓存的key进行哈希操作， 并且对缓存节点的ip地址使用同样的方法进行哈希操作， 并将其置于一个环中， 如下图左侧所示:

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/distributed-cache/hash-circle.png)

在完成了这一部操作之后， 剩下的就是解决数据归属问题。 一致性哈希算法的思路就是找到某一个key在顺时针方向上最近的节点， 就是该key应该在的节点。 如上图所示， key1顺时针寻找， 离得最近的节点为Node1， 所以key1存储于Node1。 key2, key3存储于Node2， key4存储于Node3， key5, key6存储于Node4。

当我们在Node3和Node4之间新增一个节点Node5时， 受到影响的key只有Node3和Node5之间的少部分节点:

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/distributed-cache/add-node.png)

原本key5归属于Node4， 但是由于新增节点的缘故， 在顺时针方向上离key5最近的节点为Node5， 所以key5被重新分配了。 而在缓存这个场景下， 由于key存在过期时间， 再加上缓存数据的非相关性， 系统能够快速的将这些数据重新缓存至新的节点中， 并且只有一小部分的数据会收到影响。

删除节点同样只会影响一小部分的数据分布。 当删除图中的Node2节点之后,  顺时针方向上离的最近的节点为Node3， 那么缓存数据将会被重新分配至Node3节点, 如下图所示:

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/distributed-cache/remove-node.png)

有的时候系统中节点数据比较少， 在进行顺时针寻找节点时， 很有可能发生绝大多数key都去了同一个节点:

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/distributed-cache/%E5%88%86%E5%B8%83%E4%B8%8D%E5%9D%87%E5%8C%80.png)

在系统中一种有6个缓存数据， 其中有5个数据均存储在了Node2节点， 分布非常的不均匀。 解决方法为引入虚拟节点， 其实就是将一个节点的ip， 使用字符串后缀的方法哈希多个值， 产生虚拟节点， 数据在顺时针寻找节点时如果结果是虚拟节点的话， 程序做额外的处理工作， 将其存储至虚拟节点的真实节点上。

假如Node3的ip为`172.15.243.16`， 通过添加字符串后缀的方式来添加虚拟ip， 例如`172.15.243.16@1`, `172.15.243.16@2`， 目的就是让同一个节点能够产生多个哈希值， 从而使得数据分布均匀:

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/distributed-cache/visual-node.png)

以上就是一致性哈希算法的大致内容， 在实现上节点的存储可以采用AVL数或者是红黑树， 如果节点数量很少的话， 有序链表结构都可以。

一致性哈希算法在缓存设计中应用要更多一些， 一方面是因为缓存数据之间没有强关联性， 并且没有类似于关系型数据库的二级索引结构， 怎么分都可以。 更重要的是数据分布均匀性以及自动化的故障转移。

#### 8. 一致性哈希算法的热点key问题
虽然一致性哈希算法在引入虚拟节点后能够最大程度上的解决分区平衡问题， 但是很难处理热点数据。 一个非常极端的例子就是系统中所有缓存的读/写都是针对同一个关键字， 那么最终所有的请求都将被路由到同一个节点， 造成该节点负载急剧增加。

举个不恰当的例子， 微博声称目前的系统支持8位明星同时出轨(然而不能支持宣布结婚, hiahia)。 如果系统采用用户id或者是事件id作为缓存key， 几秒内的对同一个数据的读/写流量是非常巨大的。

热点key问题并没有一个非常好的解决方案， 只能通过应用层的scatter/gather来解决。 即对于热点的用户id或者是事件id进行随机数的添加， 将其分配至不同的分区上， 读取时再进行合并。

例如原本的热点key为`user_marriage_9527_list`, value为一个列表对象。 应用层生成[0-50)的随机数， 添加至`user_marriage_9527_list`的尾部， 每次的写操作都进行随机数的追加， 那么得到的key就有`user_marriage_9527_list0`, `user_marriage_9527_list1`...将这些list数据写入至不同的节点中。 在读取时， 从0到50遍历所有热点key， 结果进行合并， 去重， 返回。

因为读取时的额外操作， 所以通常只对极少数热点key添加随机数才有意义。


#### 9. 小结
分布式缓存设计是一个相当庞大的话题， 单靠一篇博文没有办法将其完整的描述， 以及对各种问题给出确切的解决方法， 所以本文也仅是在宏观角度上去分析一些最为常见的问题。

对于中小型服务而言， 我认为将缓存设计成为分布式并不是一个很好的选择， 能够进行垂直拓展的服务尽量先进行垂直拓展， 当垂直拓展满足不了需求之后， 再考虑分布式服务设计。