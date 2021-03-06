---
layout: post
cover: false
navigation: true
title: Django处理数据并发问题
date: 2018-09-18 10:18:00
tags: ['Django', 'Concurrent', 'Lock']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

在Web开发中， 请求的并发处理通常会直接反映到数据库中数据的并发处理。 如果需要在并发的条件下保证数据的准确性， 则必须借助锁的力量来完成。 锁又分乐观锁和悲观锁， 表示了世界的两极。 本篇文章只是以Django作为载体， 来描述数据的并发处理。

<!---more--->


#### 1. 为什么要对数据加锁
看似简单的一句话: `select * from table`， 其背后都是无数条C/C++语句实现的，这一系列对数据进行访问和更新的操作指令， 组合成一个整体， 从而形成了事务。 单条SQL语句也是一个事务。

当我们对某一条数据进行更新时， 如果更新语句未执行完毕， 则其它的读操作将会被阻塞， 这是MySQL天然地为我们提供的数据读写锁。 那么为什么我们需要额外的对数据进行加锁呢? 因为业务是复杂的。

就拿减库存这个操作来讲， 首先我们取出库存数据， 判断库存数量是否大于0， 如果大于0， 则执行减库存操作， 否则返回库存不足。

```python
product = Product.objects.get(id=101)
if product.storage > 0:
    Product.objects.filter(id=101).update(storage=F("storage") - 1)
    # UPDATE product SET storage = storage - 1 WHERE id = 101;
```

在并发场景下， 这段代码很有可能会造成库存减至负数， 造成超卖的问题。 因为当线程A执行`update`时， 线程B线程可能已经将库存减至0了， 那么线程A再进行更新的话就会造成负数库存。

Python的线程锁， 或者是Java的CPU总线锁， 都无法解决这个问题， 因为服务可能分布在多台机器上。 此时要么采用分布式锁， 要么将锁下沉至数据库中。 本篇博文讨论后者。


#### 2. MySQL中的锁
如果按锁粒度来分的话， 会有表级锁， 行级锁， 页级锁。

如果按锁级别来划分的话， 会有共享锁， 排它锁。

- 共享锁可以称之为读锁， 为可重入锁， 即多个只读事务可以对当前数据同时进行加锁， 但是只允许一个事务进行更新操作。 若首先更新的事务未提交， 则其余事务阻塞并等待前一个事务的提交。使用`SELECT ... LOCK IN SHARE MODE;`来实现读锁， 具体说明见下文。

- 排它锁可以称之为写锁， 如果某个事务对数据加上了排它锁， 那么所有的事务都不能再在这些数据上添加任何的锁， 直到前一个事务结束。 使用`SELECT ... FOR UPDATE;`语句来实现。

在处理并发数据一致性的问题时， 常常会以使用方式来划分， 即乐观锁和悲观锁。

##### 2.1 悲观锁
悲观锁， 顾名思义， 很悲观， 不相信外界的任何东西， 只相信自己拿到手的。 所以悲观锁在对数据修改之前， 首先会对该数据加上排它锁， 然后修改数据， 事务结束时释放锁。 如果加锁失败了的话， 说明有其他的事务对该数据进行了加锁操作， 此时可以等待， 也可以使用等待超时的方式。 但是MySQL会有自己的超时时间， 不会让客户端永久等待，  具体的响应方式由开发人员决定。

悲观锁使用`SELECT ... FOR UPDATE;`语句来实现: 当一个事务执行到此句之后， 其余事务如果对相同的数据进行更新或者删除， 操作将会被阻塞， 直到前一个事务结束或者等待超时。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/blogv2/django_concurrent/select_for_update.gif)

用悲观锁来实现减库存操作:

```sql
begin;
select storage from product where id = 101 for update;
-- 这里就不写正规的存储过程了, 简单理解下逻辑
if storage > 0 then
    update product set storage = storage - 1 where id = 101;
    commit;
```

如果使用Django-ORM来做的话:

```python
from django.db import transaction
from django.db.models import F

with transaction.atomic():
    product = Product.objects.select_for_update().get(id=101)
    if product.storage > 1:
        Product.objects.filter(id=101).update(storage=F("storage") - 1)
```

如果用SQLAlchemy来做的话， 原理上也是一样的:

```python
from contextlib import contextmanager

@contextmanager
def transaction_scope():
    # 这种创建Session的方式只能在MySQL autocommit=False的情况下使用
    # 若autocommit=True, 则需要使用session.begin()显式开启事务
    session = Session()
    try:
        yield session
        session.commit()
    except:
        session.rollback()
        raise
    finally:
        session.close()

with transaction_scope() as session:
    product = session.query(Product).with_for_update().filter_by(id=101).first()
    if product.storage > 0:
        session.query(Product).filter_by(id=101).update({
            "storage": Product.storage - 1
        })
```

不管我们使用何种ORM， 何种Web框架， 只要支持`select ... for update`语法， 我们都可以写出相似的代码来实现相应的功能。

在语法的使用上要尤为注意索引的问题。 当`select ... where column=.. for update`中的`column`列未添加索引时， 行锁将会直接升级为表锁， 可能会造成不可预估的事故。 所以在使用时查询条件必须添加索引， 如果为了进一步优化事务执行效率， 添加唯一索引或者使用主键是更好的选择。

此外， 如果对悲观锁感兴趣的话， 也可以查看Django为我们提供的`get_or_create`, `update_or_create`方法的源码， 在这些方法里面， 应用了悲观锁的方式来实现相关功能。

##### 2.2 乐观锁
乐观锁严格意义上来讲并不是一种锁， 而是一种思维方式， 一种不对数据添加任何锁， 并且能在一定程度上解决数据并发问题的思维方式。

乐观锁， 大多是基于数据版本(version)记录机制实现。 什么是数据版本? 为数据增加一个版本标识， 通常是一个整型字段。

读取数据时， 将版本号一同读出， 之后更新时， 对此版本号加一。 此时， 将提交数据的版本数据与数据表对应记录的当前版本信息进行对比， 如果提交的数据版本号大于数据库表当前版本号， 则予以更新， 否则认为是过期数据。

其底层原型为:

```sql
update table set column = xx, version = version + 1 where id = xx and version = xx
```

模型非常的简单， 根据`update`方法返回的更新条数来判断当前更新是否成功， 如果结果为1， 说明更新成功。 若为0， 则更新失败， 说明此时有其它线程或者进程更新了该数据， 我们需要重新从数据库中取出数据， 判断并决定是否再次尝试更新。

代码写起来也非常简单， 以Django为例:

```python
# 这里就简单的使用暴力循环重试了, 更优雅的实现看业务场景
for i in range(10):
    product = Product.objects.get(id=101)
    if product.storage > 0:
        result = Product.objects.filter(id=101, version=product.version)\
                 .update(storage=F("storage") - 1, version=F("version") + 1)
        if result > 0:
            return True
        continue
    else:
        return False
```

假如说不想改变现有table的结构， 那么也可以使用`updated_at`字段来替代version， 让数据库自己帮我们去管理版本号的更新， 我们专注于常规业务逻辑的编写， 并且能够使得代码更加的简洁。 使用version整型字段的好处就在于我们能够很清晰的看到当前数据的更新次数， 也有利于我们做一些数据分析之类的场景需求。

乐观锁能够在一定程度上的解决并发的数据问题， 但是不是全部。

假如在秒杀这个场景下使用乐观锁来进行库存数量的扣减， 就会出现大量的用户查询库存存在， 但是却在减库存的时候失败了，因为会有其它线程的更新。  这样一来就会导致大面积的线程进行重试， 最终一部分用户达到重试的最大次数， 返回库存为0。 但是这个时候库存完全可能很充足， 只是因为线程之间的争抢更新导致无法更新， 造成用户下单失败。

在这种场景下， 不仅仅需要实现锁机制， 还需要实现限流等一系列机制来保证服务的准确与稳定。


##### 2.3 共享锁
这种模式笔者其实用的非常少， 总感觉其意义不大。 共享锁的原理为: 多个事务可同时对某一条数据添加共享锁， 但是只允许一个事务对该数据进行更新。 并且当某一个事务执行更新操作后， 该数据不允许其余事务继续添加共享锁。

```sql
-- 事务A
select * from auth_user where id = 1 lock in share mode;

-- 事务B
select * from auth_user where id = 1 lock in share mode;

-- 事务A
update auth_user set first_name = "keyeror" where id = 1;

-- 事务B
update auth_user set first_name = "keyeror" where id = 1;
-- 此时将会阻塞， 等待事务A的结束

-- 事务C
select * from auth_user where id = 1 lock in share mode;
-- 此时事务C无法对该数据进行共享锁的添加
```

#### 3. Django处理数据更新
有时候我们可能写出这样的代码:

```python
product = Product.objects.get(id=101)
# ...
# 中间一些业务逻辑
product.some_field = some_value
product.save()
```

在绝大多数场景下这么写都没有什么问题， 但是当涉及到对字段进行加减时， 就会出现问题。

比方说我们为一个用户的账户里面充钱， 只有一个操作， 就是更新用户的`account`字段。 并且假设模型如下:

```python
class Demo(models.Model):
    username = models.CharField(max_length=128)
    phone = models.CharField(max_length=11, unique=True)
    account = models.IntegerField()
```

使用`select->save`的方式在并发条件下会出现什么?

```python
def get_and_update_account():
    demo = Demo.objects.get(id=1)
    demo.account += 100
    demo.save()


if __name__ == "__main__":
    threads = []

    for i in range(10):
        t = Thread(target=get_and_update_account)
        t.start()
        threads.append(t)

    for thread in threads:
        thread.join()

    demo = Demo.objects.get(id=1)
    print(demo.account)
```

最终结果可能是10个线程执行完毕， 但是`account`数额可能远小于1000， 导致用户的账户余额异常。

应该尽量在代码中避免此种更新方式， 就算它是并发安全的更新。

```python
from django.db.models import F

def update_account():
    Demo.objects.filter(id=1).update(account=F("account")+100)
```

如果要对原有字段数据进行加减操作， 请使用`F`函数， 上面的更新语句所执行的SQL语句为:

```sql
update tx_demo set account = account + 100 where id = 1;
```

#### 4. 关于Django中的`update_or_create`方法
Django为用户提供了两个方便的方法: `get_or_create`以及`update_or_create`， 从函数名称上就可以知道这两个方法到底做了什么， 前者表示有则获取， 无则创建; 后者表示有则更新， 无则创建。 在官方文档中， 有这样的一个描述:

> This method is atomic assuming that the database enforces uniqueness of the keyword arguments (see unique or unique_together). If the fields used in the keyword arguments do not have a uniqueness constraint, concurrent calls to this method may result in multiple rows with the same parameters being inserted.

原文链接:

> [https://docs.djangoproject.com/en/2.2/ref/models/querysets/#get-or-create](https://docs.djangoproject.com/en/2.2/ref/models/querysets/#get-or-create)

大致的意思就是当table中未包含`unique`约束时， 在并发条件下将会创建出多条重复数据。

在`update_or_create`方法中， 也有相同的描述:

> As described above in get_or_create(), this method is prone to a race-condition which can result in multiple rows being inserted simultaneously if uniqueness is not enforced at the database level.

原文链接:

> [https://docs.djangoproject.com/en/2.2/ref/models/querysets/#update-or-create](https://docs.djangoproject.com/en/2.2/ref/models/querysets/#update-or-create)

但是当我们翻看其源码时， 发现它使用了`select_for_update`方法， 即对表中数据添加了行锁。 既然如此， 为何还会创建出多条数据呢?

```python
def update_or_create(self, defaults=None, **kwargs):
    defaults = defaults or {}
    lookup, params = self._extract_model_params(defaults, **kwargs)
    self._for_write = True
    with transaction.atomic(using=self.db):
        try:
            obj = self.select_for_update().get(**lookup)
        except self.model.DoesNotExist:
            # Lock the row so that a concurrent update is blocked until
            # after update_or_create() has performed its save.
            ...
```

其原因就在于Django使用的事务隔离级别为`READ COMMITTED`, 并非是`REPEATABLE READ`, 至于为什么使用提交读的事务隔离级别， 文档给出了这样的解释:

> Django works best with and defaults to read committed rather than MySQL’s default, repeatable read. Data loss is possible with repeatable read. In particular, you may see cases where get_or_create() will raise an IntegrityError but the object won’t appear in a subsequent get() call.

但是我认为， 数据一致性高于服务可用性， 当出现数据不一致时， 服务可用性也将毫无意义。

当MySQL的事务隔离级别为`read committed`时， 对数据库中未存在的数据添加悲观锁， 并不会对数据添加间隙锁或者是Next-Key Lock，那么多个事务可同时执行INSERT

MySQL官方对这两种隔离级别的`FOR UPDATE`悲观锁做了比较详细的解释:

> [https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)

关键解释:

> For locking reads (SELECT with FOR UPDATE or FOR SHARE), UPDATE statements, and DELETE statements, InnoDB locks only index records, not the gaps before them, and thus permits the free insertion of new records next to locked records. Gap locking is only used for foreign-key constraint checking and duplicate-key checking.

这也就是为什么在默认情况下， Django的2个方法可能创建出多条重复数据的原因。解决方法也很简单， 将事务隔离级别修改为`repeatable read`即可。

```python
# settings.py

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        "OPTIONS": {
            "isolation_level": "repeatable read"
        }
    }
}
```


#### 5. 小结
Django的ORM只是一个载体， 不管使用何种ORM， MySQL的底层原理与并发原理都是相同的， 所以即使是换成SQLAlchemy或者其它语言的ORM框架， 上述内容也同样适用。

悲观锁是由数据存储层所提供的一种事务更新排它锁， 拥有着较强的数据一致性保证， 但是当大量用户涌入时会有大量锁争抢的问题， 可能会有一定的效率问题。

乐观锁则采用版本控制的方式对数据的实时有效性进行保证， 整体实现无锁， 由业务端来选择实现方式， 更加的灵活。 但是在高并发场景下仍然会有些许不足。

所以， 锁并不是用来解决高并发问题的， 而只是保证并发场景下的数据一致性。