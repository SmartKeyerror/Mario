---
layout: post
cover: false
navigation: true
title: MySQL-InnoDB中的锁
date: 2020-01-31 10:06:25
tags: ['MySQL']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

锁在InnoDB存储引擎中的使用远比我们想象中的更加频繁，及时是一条最为简单的`update set`语句，其中也涉及到了各种锁的使用。包括常说的一致性锁定读，解决幻读等场景中，同样包含了锁的大量使用。

<!---more--->

#### 1. Latch和Lock

在InnoDB存储引擎中，Latch(门闩)是用来保证并发线程操作临界资源的正确性，保证某些操作的原子性。通常又分为Mutex(互斥量)和RWLock(读写锁)，例如Python中`threading.Lock`，Java中`synchronized`，Golang中的`sync.Mutex`，Latch通常应用于操作缓冲池中的LRU列表元素(添加、删除以及移动)，部分场景下的`AUTO_INCREMENT`实现。用户通常不会直接地与Latch打交道，并且没有死锁检测。

Lock作用于事务之中，用来锁定表、页、行，锁的添加与释放通常会在事务的起始和结束时进行。数据库中的幻读问题解决就是通过Lock实现的，而非Latch。并且Lock存在死锁检测机制，当发生死锁时，会在某些情况下告知用户，例如在使用一致性锁定读(SELECT...FOR UPDATE)时产生的死锁，会直接抛出1213的Deadlock异常。

尽管Latch与Lock操作的对象均为数据，但是Latch更为底层，操作的对象更加细小。Lock的对象相对于Latch而言，则更加"粗放"，例如表、页数据，此外最重要的是Lock的作用域为事务，Latch则不是。

#### 2. InnoDB存储引擎中的Lock

为了方便叙述，下面均使用锁来指代InnoDB中的Lock(仍然要说明，Lock以及Latch都可以称为锁，这里只是为了方便叙述)。

InnoDB引擎支持行锁以及表锁，既可以锁定某一行，同时也可以锁定一整张表，先从行级锁说起。

InnoDB引擎实现了两种标准的行级锁:
- 共享行级锁(S Lock, Share Lock)
- 排他行级锁(X Lock, Exclusive Lock)

可以认为S Lock和X Lock分别表示读锁和写锁，如同RWLock一样。S Lock允许并发地读取数据，X Lock既限制并发地读取，同时也限制并发地修改。所以说，当某一行数据中存在S锁时，只能再次添加S锁，若想要添加X锁，则需要等待S锁的释放。行级锁X以及S Lock的兼容性如下:

|-     |   X    |   S    |
| :--: | :----: | :----: |
| X    | 不兼容 | 不兼容 |
| S    | 不兼容 | 兼容   |

同时，InnoDB支持表级锁，为了支持表级锁与行级锁这两个不同粒度的锁，InnoDB支持一种额外的上锁方式，称之为意向锁(Intention Lock)。

为了更好的理解意向锁，首先假设没有意向锁，只有表锁和行锁。当事务A在更新某一条数据时，会在该数据行上添加X锁。此时另外事务B申请整个表的写锁，如果事务B申请成功，那么它就能修改表中任意一行数据，这与事务持有的X锁冲突。

如果数据库想要避免该冲突，那么需要让事务B阻塞，直到事务A提交释放X锁。转而需要判断事务B阻塞的条件: ①当前表是否被其它事务添加表锁 ②判断表中是否存在行锁。这两个条件判断均可以在表层面实现，而无需遍历所有数据，只需要定义好数据结构即可。一个最简单的实现就是为表锁和行锁添加两个标识位，该标识位在添加和释放锁时进行原子更新，例如:

```bash
table_s_lock = false
table_x_lock = false
row_s_lock = true
row_x_lock = true
```

当某一行添加X锁时，将`row_x_lock`置为true，若其余事务想要添加表级别的X锁，则必须等待`row_x_lock`更新为false。反之若事务已经添加了表级别的X锁，将`table_x_lock`置为true，事务B若想在某一行添加X锁，则需要等待`table_s_lock`以及`table_x_lock`均更新为false。

虽然上面的标识位能够解决问题，但仍然有些奇怪，奇怪的点在于标识位的判断粒度不同。我们更加希望表级锁与表级锁进行兼容性判断，行级锁与行级锁进行兼容性判断，而不是表级锁与行级锁进行兼容性判断。由此，就有了意向锁的诞生。

意向锁(Intention Lock)将锁定的对象分为多个粒度，当想要对细粒度的数据进行加锁时，那么首先需要对粗粒度的对象添加意向锁。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/jojo/system-design/mysql/lock/intention-Lock.png)

例如，若需要对页上的记录R添加X锁，则需要分别对数据库、表、页添加意向锁IX，添加成功后才会对记录R添加X锁，若其中任何一部分导致等待，那么该操作需要等待粗粒度上锁环节的完成。现在来看在有了意向锁之后InnoDB存储引擎如何支持多粒度的锁。

意向锁同样分为两种: 共享和排他

- 意向共享锁(IS, Intention Share Lock)
- 意向排他锁(IX, Intention Exclusive Lock)

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/jojo/system-design/mysql/lock/intention-lock-example.png)

如上图所示，事务A为了给记录R添加X锁(排他锁)，则需要依次对数据库、表、页添加意向排他锁(IX)，假设添加均成功，最终记录R添加了X锁。此时事务B想要向表A中添加表级别的排他锁，由于表A中存在IX锁，与表级别的X锁并不兼容，故事务B等待，等待表A中IX锁的释放。可以看到，在有了意向锁之后，锁的兼容性比较将处理同粒度水平，而不是跨粒度进行比较。这让我想起了一个段子:

> 不要跟傻逼争论，他会把你拉到他的水平上，然后用他丰富的经验打败你

InnoDB存储引擎中意向锁和表级锁的兼容性如下:

|  -        |  IX    |  IS    |   X(表级别)  |  S(表级别)  |
| :-:       | :-:    | :-:    | :----------: | :---------: |
| IX        | 兼容   | 兼容   | 不兼容       | 不兼容      |
| IS        | 兼容   | 兼容   | 不兼容       | 兼容        |
| X(表级别) | 不兼容 | 不兼容 | 不兼容       | 不兼容      |
| S(表级别) | 不兼容 | 兼容   | 不兼容       | 兼容        |

在MySQL 5.5以上、5.7.14以下的版本中，用户可以通过`INFORMATION_SCHEMA`下的`INNODB_TRX`、`INNODB_LOCKS`以及`INNODB_LOCK_WAITS`这三张表简单地监控并分析可能存在的锁问题。

在MySQL 8.0版本中，则需要使用`performance_schema`下的`data_locks`以及`data_lock_waits`获取相关的锁以及锁等待信息。

而MySQL版本在5.7.14到8.0之间的用户，只能通过其它手段间接的获取上述信息。

##### 2.1 创建通用例程

```sql
CREATE TABLE `user` (
  `id` int NOT NULL AUTO_INCREMENT,
  `nickname` varchar(32) COLLATE utf8mb4_general_ci NOT NULL,
  `password` varchar(128) COLLATE utf8mb4_general_ci NOT NULL,
  `user_id` varchar(16) COLLATE utf8mb4_general_ci NOT NULL,
  `mobile` varchar(11) COLLATE utf8mb4_general_ci NOT NULL,
  `mobile_area` smallint NOT NULL comment "手机号码区域",
  `gender` tinyint DEFAULT 0,
  `avatar` varchar(128) COLLATE utf8mb4_general_ci DEFAULT NULL,
  `account_id` varchar(32) COLLATE utf8mb4_general_ci NOT NULL,
  `created_at` datetime DEFAULT CURRENT_TIMESTAMP,
  `updated_at` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `deleted_at` datetime,
  `status` tinyint DEFAULT 1 comment "用户状态",
  PRIMARY KEY (`id`),
  KEY `user_id` (`user_id`),
  KEY `mobile` (`mobile`),
  KEY `account_id` (`account_id`),
  KEY `created_at` (`created_at`),
  KEY `updated_at` (`updated_at`),
  UNIQUE (`user_id`),
  UNIQUE (`account_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

这是一张非常普通但又普遍的用户信息表，其中包含了唯一主键，唯一辅助索引以及普通辅助索引。


##### 2.2 INNODB_TRX
`INNODB_TRX`表中主要记录了当前正在执行的事务信息，包括只读事务。首先来看字段和字段所表示的含义:

| 字段名称                   | 字段含义 |
| -------------------------- | -------- |
| TRX_ID                     | InnoDB存储引擎内部的唯一事务ID |
| TRX_WEIGHT                 | 事务权重(与事务修改的行数和锁定的行数有关)，当两个事务执行发生死锁时，InnoDB会选择权重较低的事务进行回滚 |
| TRX_STATE                  | 当前的事务执行状态，包括RUNNING, LOCK WAIT, ROLLING BACK, 以及COMMITTING，LOCK WAIT表示当前事务正等待某个锁的释放 |
| TRX_STARTED                | 事务开始时间，格式如2000-01-01 14:01:08 |
| TRX_REQUESTED_LOCK_ID      | 当前事务所等待的锁ID，该字段只有在状态为LOCK WAIT才有值，否则为NULL。可与`INNODB_LOCKS`通过LOCK_ID字段进行关联查询，获取更为详细的锁信息。|
| TRX_WAIT_STARTED           | 当前事务等待锁的起始时间，在状态为LOCK WAIT时才有值，否则为NULL。 |
| TRX_QUERY | 当前事务**正在**执行的SQL语句(不是事务所有的执行语句) |
| TRX_OPERATION_STATE        | 事务的当前操作状态，包括PREPARING, UPDATING, DELETING, COMMITTING以及NULL，该字段在绝大部分情况下均为NULL，捕捉某一事务的瞬间执行状态还是比较困难的(除非是大事务) |
| TRX_TABLES_IN_USE          | 正在执行的SQL语句所操作的表数量，是一个动态变化值，通常很难观测 |
| TRX_TABLES_LOCKED          | 当前事务在各个表中添加行锁的表数量 |
| TRX_LOCK_STRUCTS           | 当前事务持有的锁数量 |
| TRX_LOCK_MEMORY_BYTES      | 当前事务中锁结构的内存总占用 |
| TRX_ROWS_LOCKED            | 当前事务锁住的近似数据总行数 |
| TRX_ROWS_MODIFIED          | 当前事务插入、修改的总行数 |
| TRX_CONCURRENCY_TICKETS    | 表示当前事务在换出之前所能做的工作之和 |
| TRX_ISOLATION_LEVEL        | 当前事务隔离级别，包括READ UNCIMMITTED、READ COMMITTED、READ REPEATABLE以及SERIALIZABLE |
| TRX_UNIQUE_CHECKS          | 当前事务是否开启唯一性检查 |
| TRX_FOREIGN_KEY_CHECKS     | 当前事务是否开启外键检查 |
| TRX_LAST_FOREIGN_KEY_ERROR | 当前事务执行时最后发生的外键错误 |
| TRX_ADAPTIVE_HASH_LATCHED  | 当前事务是否锁定了自适应哈希索引 |

在这20多个字段中，较为重要的包括事务ID，事务执行状态，事务等待锁的起始时间，事务锁定的近似总行数。

##### 2.3 INNODB_LOCKS

`INNODB_LOCKS`表中记录了当前所有未释放的锁，包括行锁、页锁以及表锁，当某个事务发生严重的锁等待时，通常会在该表中查找蛛丝马迹，确定问题的根源。

但是，`INNODB_LOCKS`和`INNODB_LOCK_WAITS`这两张表在5.7.14以上版本中被废弃不用，在8.0版本中使用`data_locks`以及`data_lock_waits`进行代替，故以下内容均采用MySQL 8.0版本进行描述。

##### 2.4 data_locks

| 字段名称              | 字段含义 |
| --------------------- | -------- |
| ENGINE                | 申请或持有锁的存储引擎类型 |
| ENGINE_LOCK_ID        | 存储引擎内部的锁ID，该值会发生动态变化，外部系统不应该依赖该值 |
| ENGINE_TRANSACTION_ID | 持有锁的事务ID，与INNODB_TRX中的TRX_ID对应 |
| THREAD_ID             | 持有锁的线程ID |
| EVENT_ID              | 事件ID，该字段将于下方进行详细描述 |
| OBJECT_SCHEMA         | 锁所在的schema(database) |
| OBJECT_NAME           | 锁所在的表名称 |
| PARTITION_NAME        | 锁所在分片名称 |
| SUBPARTITION_NAME     | 锁所在的子分片名称 |
| INDEX_NAME            | 被添加锁的索引名称 |
| OBJECT_INSTANCE_BEGIN | 锁的内存空间起始地址 |
| LOCK_TYPE             | 锁类型，包含TABLE和RECORD |
| LOCK_MODE             | 锁的模式，包括S,X,IS,IX,AUTO_INC以及UNKNOWN |
| LOCK_STATUS           | 锁的状态，InnoDB引擎中包括GRANTED(已添加)和WAITING(等待中) |
| LOCK_DATA             | 锁覆盖的范围，该字段将于下方详细描述 |

以一个具体的例子为例:

```sql
mysql> begin;
mysql> SELECT * FROM Mario.user where id = 1 for update;

mysql> SELECT * FROM performance_schema.data_locks\G;
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140374385659344:1453:140374295256456
ENGINE_TRANSACTION_ID: 632837
            THREAD_ID: 56
             EVENT_ID: 28
        OBJECT_SCHEMA: Mario
          OBJECT_NAME: user
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140374295256456
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140374385659344:532:4:2:140374295253576
ENGINE_TRANSACTION_ID: 632837
            THREAD_ID: 56
             EVENT_ID: 28
        OBJECT_SCHEMA: Mario
          OBJECT_NAME: user
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140374295253576
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: 1
```

当我们使用`FOR UPDATE`一致性锁定读向id为1的行添加一个写锁时，可以看到`data_locks`中生成了两条记录。从`LOCK_TYPE`以及`LOCK_MODE`来看，第一条为表级别意向排他锁(IX)，第二条为行记录排他锁(X)。注意到X锁后面还有一个说明: `REC_NOT_GAP`，表示排他非间隙行锁，这是行锁的一种实现，将在后面小节中描述。

`LOCK_DATA`在IX项中为NULL，这是因为在InnoDB存储引擎中，该字段只会在`LOCK_TYPE`为`RECORD`时才有实际值，对于`TABLE`类型的锁而言，该值为NULL。`LOCK_DATA`根据不同的加锁方式会有不同具体值。当我们使用主键ID(primary key)进行加锁时，`LOCK_DATA`仅包含聚簇索引行记录，此时`LOCK_DATA`的值通常为主键ID。当我们使用辅助索引对记录加锁时，锁住的范围则会包括辅助索引+聚簇索引，所以此时`data_locks`会生成3条记录(表级别意向锁+索引记录锁+聚簇索引记录锁)，此时`LOCK_DATA`的值为"辅助索引字段值+主键ID"

例如:

```sql
mysql> begin;
mysql> SELECT * FROM Mario.user where user_id = "168236477" for update;

mysql> SELECT * FROM performance_schema.data_locks\G;
*************************** 1. row ***************************
        OBJECT_SCHEMA: Mario
          OBJECT_NAME: user
          INDEX_NAME: NULL
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
        OBJECT_SCHEMA: Mario
          OBJECT_NAME: user
           INDEX_NAME: user_id_2
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP
          LOCK_STATUS: GRANTED
            /* LOCK_DATA为FieldValue+记录对应的主键ID。由于user_id为unique，故此处仅一条记录 */
            LOCK_DATA: '168236477', 3
*************************** 3. row ***************************
        OBJECT_SCHEMA: Mario
          OBJECT_NAME: user
           INDEX_NAME: PRIMARY
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: 3
```

##### 2.5 data_lock_waits

`data_lock_waits`相比于`INNODB_TRX`以及`data_locks`而言则要更复杂一些，该表实际上是一个ManyToMany的关系表，记录了`data_locks`中锁之间的等待以及依赖关系，同时也记录了锁所对应的事务/会话信息。

| 字段名称                         | 字段含义 |
| -------------------------------- | -------- |
| ENGINE                           | 存储引擎类型 |
| REQUESTING_ENGINE_LOCK_ID        | 存储引擎内锁ID，对应于data_locks表中的ENGINE_LOCK_ID |
| REQUESTING_ENGINE_TRANSACTION_ID | 存储引擎内事务ID |
| REQUESTING_THREAD_ID             | 线程ID |
| REQUESTING_EVENT_ID              | 事件ID |
| REQUESTING_OBJECT_INSTANCE_BEGIN | 锁的内存空间起始地址 |
| BLOCKING_ENGINE_LOCK_ID          | 等待释放的锁ID   |
| BLOCKING_ENGINE_TRANSACTION_ID   | 等待结束的事务ID |
| BLOCKING_THREAD_ID               | 等待结束的线程ID |
| BLOCKING_EVENT_ID                | 等待结束的事件ID |
| BLOCKING_OBJECT_INSTANCE_BEGIN   | 等待结束的锁的内存空间起始地址 |

例如:

```sql
mysql> select * from performance_schema.data_lock_waits\G;
*************************** 1. row ***************************
                          ENGINE: INNODB
       REQUESTING_ENGINE_LOCK_ID: 140678484647376:532:4:2:140678365511992
REQUESTING_ENGINE_TRANSACTION_ID: 635403
            REQUESTING_THREAD_ID: 48
             REQUESTING_EVENT_ID: 15
REQUESTING_OBJECT_INSTANCE_BEGIN: 140678365511992
         BLOCKING_ENGINE_LOCK_ID: 140678484646504:532:4:2:140678365506120
  /*等待ID为635400的事务释放锁*/
  BLOCKING_ENGINE_TRANSACTION_ID: 635400
              BLOCKING_THREAD_ID: 47
               BLOCKING_EVENT_ID: 12
  BLOCKING_OBJECT_INSTANCE_BEGIN: 140678365506120
```

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/jojo/system-design/mysql/lock/data_lock_waits.png)

#### 3. InnoDB存储引擎行锁算法

InnoDB存储引擎存在3种行锁算法，分别为:

- Record Lock: 单个行记录上的锁
- Gap Lock: 间隙锁，锁定一个范围，单不包含记录本身
- Next-Key Lock: Record Lock+Gap Lock，锁定一个范围，并且锁定记录本身

Record Lock表示单个行记录上的锁，这非常好理解，例如我们`update`一条或多条数据时，事务会为这一条或者多条数据均添加X锁。当使用主键ID进行更新时，记录仅包含聚簇索引行记录。当使用辅助索引进行更新时，将会锁住聚簇索引记录+辅助索引记录。

```sql
mysql> begin;
mysql> update user set status = 2 where id = 1;
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
/* ...................表级别的意向排他锁，此处省略.............. */
*************************** 2. row ***************************
        /* 省略部分非关键信息 */
        OBJECT_SCHEMA: Mario
          OBJECT_NAME: user
           INDEX_NAME: PRIMARY
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: 1
mysql> commit;
```

当使用主键ID进行一致性锁定读时，`data_locks`生成两条锁记录，一条为table IX，另一条为行记录的X锁，注意`LOCK_MODE`后面的附加声明: REC_NOT_GAP，表示当前锁的算法仅为行记录锁，非间隙锁。

```sql
mysql> begin;
mysql> update user set status = 2 where user_id = "174269548";
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
/* ...................表级别的意向排他锁，此处省略.............. */
*************************** 2. row ***************************
        OBJECT_SCHEMA: Mario
          OBJECT_NAME: user
           INDEX_NAME: user_id_2
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: '174269548', 1
*************************** 3. row ***************************
        OBJECT_SCHEMA: Mario
          OBJECT_NAME: user
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140678365506464
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: 1
```

而使用辅助索引进行一致性锁定读时，除了table IX以及聚簇索引的X锁以外，还会有额外的辅助索引X锁，`LOCK_MODE`同样备注了非间隙锁的标识。

间隙锁的存在主要是为了解决幻读问题，幻读是指当某事务读取一定范围内的数据时，其余事务在该范围内插入了一条或多条数据，或者删除了一条或多条数据，导致前一个事务读取的数据条数发生改变，如同出现幻觉，所以称为幻读。

```sql
mysql> begin;
mysql> select * from user where id > 2 for update;
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
/* ...................表级别的意向排他锁，此处省略.............. */
*************************** 2. row ***************************
        OBJECT_SCHEMA: Mario
          OBJECT_NAME: user
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140678365506120
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record
*************************** 3. row ***************************
        OBJECT_SCHEMA: Mario
          OBJECT_NAME: user
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140678365506120
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 3
```

当我们对某一个范围使用一致性锁定读时，就可以看到间隙锁的产生。`LOCK_MODE`仅为X时，就表示当前锁添加了间隙锁。并且在`LOCK_DATA`有supremum pseudo-record的解释说明，该说明表示MySQL决定锁定最大间隙范围。在本例中，为id大于2的所有数据，故另一个事务执行:

```sql
insert into user(id, nickname, password, user_id, mobile, mobile_area, gender, avatar, account_id, status) values(9999, "jojo", "passwd", "147523659", "13555555555", 1, 2, "https://jojo.com", "1753681429", 1);
```

将会被阻塞，直至前一个事务释放间隙锁或者当前事务锁等待超时。

在理解了间隙锁以后，Next-Key Lock就很容易理解了，锁定一个记录+一个范围。上面例子均有一个特点，就是不管是主键ID，还是user_id，它们都具有unique约束，而对于非唯一的辅助索引而言，即使是精确查询并加锁，也会添加Gap Lock，此时就是Next-Key Lock。

```sql
mysql> begin;
mysql> select * from user where updated_at = "2020-01-23 21:32:52" for update;
/*此时DB中仅存在一条数据更新时间为"2020-01-23 21:32:52"*/
mysql> select * from performance_schema.data_locks\G;
*************************** 1. row ***************************
/* ...................表级别的意向排他锁，此处省略.............. */
*************************** 2. row ***************************
           INDEX_NAME: updated_at
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record
*************************** 3. row ***************************
           INDEX_NAME: updated_at
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 0x99A56F5834, 1
*************************** 4. row ***************************
           INDEX_NAME: PRIMARY
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: 1
```

`updated_at`字段仅添加了普通索引，并且值为"2020-01-23 21:32:52"的记录主键ID为1，从`data_locks`的最后一条记录也可以看出。在该表的第二行和第三行中分别添加了间隙锁，第三行的`LOCK_DATA`字段值为16进制数+主键ID。

此外，**需要尤为注意的是，`READ COMMITTED`事务隔离级别下，将不会有间隙锁的添加**。在文章[Django处理数据并发问题](https://smartkeyerror.com/django-concurrent-data-process)中描述了使用Django默认的`READ COMMITTED`事务隔离级别所带来的问题。


#### 4. 自增长与锁

自增长在数据库中是非常常见的属性，MySQL提供`AUTO_INCREMENT`属性使得列可具备自增长的功能。在InnoDB存储引擎内存结构中，对每个含有自增长值的表都有一个自增长计数器。

最初自增长是采用特殊的表锁实现，称为AUTO_INC Locking，为了提高插入的性能，锁并不是在事务结束时才释放，而是在完成对自增长值插入的SQL语句后立即释放。虽然AUTO_INC Locking从一定程度上提高了并发插入的效率，但是仍存在性能问题: 事务必须等待前一个事务插入语句的结束。所以，后续就有了轻量级的互斥量自增长实现。

互斥量的实现就是文章最开头所说的Latch，由硬件协助实现。该实现方式只有在确定所插入的行数时才会使用，否则，将仍然使用AUTO_INC Locking。

#### 5. Metadata Lock

Metadata Lock，又称为MDL，相较于行锁和表锁，其范围更广，对象包括数据库、表、行以及触发器和外键等，与InnoDB其它锁一样，在事务开始时获取，事务结束时释放，其设计目的在于保证在事务执行过程中表的结构不会被修改。

通常来讲，只有在修改表结构的时候我们才会直接地与MDL打交道，例如向某张表添加一列，或者删除某一列。在DML执行非常频繁的应用中，当我们执行ALTER TABLE table ADD column时，很有可能出现整个MySQL挂掉的情况，其原因就在于表结构修改语句获取MDL时阻塞，导致后续对该表的查询、修改和删除等语句阻塞。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/jojo/system-design/mysql/lock/MDL.png)

session A与session B会向表user添加只读MDL，而session C想要申请可写的MDL，由于前面两个事务均为提交，故只能阻塞。此时由于session C写锁的申请，导致session D以及后续的所有DML操作均会被阻塞，简单来说，此时表user不可读写。

如果user表中的读写非常频繁，将会导致大量的查询或更新语句阻塞，且状态均为`waiting for metadata lock`。此时若客户端存在超时重试机制，那么会导致大量新的会话建立，最后达到MySQL线程数量的限制，导致整个DB不可用。

在MySQL 5.6版本以上支持Online DDL，其过程如下:
- ALTER TABLE table ADD column语句获取MDL写锁
- 获取成功后，将其降级为MDL读锁
- 执行真正的DDL操作，如添加、删除列，期间可以执行DML语句
- 升级MDL读锁为写锁
- 释放MDL写锁，整个DDL过程结束

真正导致数据库不可读写的步骤为1、4，第3步为实际运行时间最长的步骤，不会影响表的读写操作，只要内存和磁盘容量足够，数据量再多也灭有关系。所以，DDL的关键影响因素不在于数据量，而是在于数据读写的QPS。这也是为什么表结构修改操作要放到月黑风高的凌晨进行操作的原因: 那时候访问量最少，而不是数据量最少。

在更改表结构时造成大面积读写操作阻塞的另一个原因就是长事务，即长时间运行的事务。即使QPS非常小，但是系统中存在长事务，同样会造成DDL语句获取写锁阻塞，从而阻塞后续的读写语句。


#### 6. Reference

- [https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_latch](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_latch)
- [https://dev.mysql.com/doc/refman/8.0/en/innodb-trx-table.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-trx-table.html)
- [https://dev.mysql.com/doc/refman/8.0/en/data-locks-table.html](https://dev.mysql.com/doc/refman/8.0/en/data-locks-table.html)
- [https://dev.mysql.com/doc/refman/8.0/en/data-lock-waits-table.html](https://dev.mysql.com/doc/refman/8.0/en/data-lock-waits-table.html)
