---
layout: post
cover: false
navigation: true
title: binlog的正确打开方式
date: 2018-10-23 10:18:00
tags: ['MySQL']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---


在前面的主从复制中我们提到了bin-log， 主从复制中bing-log主要作为一种增量复制的方法进行主库与从库的同步。 在日常生产中， bin-log常常也作为实时数据恢复的必要手段。

<!---more--->


#### 1. 配置binlog

binlog二进制日志的格式有`statement`, `row`以及`mixed`。 其中`statement`格式只会记录增删改以及对表结构变动的SQL语句， 即`update xx set xxx`， 不会保存数据改动前的信息， 磁盘占用空间少。 `row`格式将会记录完整的数据改动前后信息， 对数据的修改更加直观， 但占用磁盘空间更多。 `mixed`为`statement`模式和`row`模式的混合， 当出现了对表结构的修改(修改， 删除字段)， 为了避免日志中记录海量的信息， 此时MySQL会采用`statement`模式记录`alter table`， 对于对数据的增删改记录每一行的信息变动。
具体的日志格式参考博文MySQL主从复制。
通常来讲二进制日志的打开比较的简单， ubuntu 18.04下：

```bash
vim /etc/mysql/mysql.conf.d/mysqld.cnf

server-id = 2                           # 保持server-id在MySQL集群中的唯一性
log_bin = /var/log/mysql/mysql-bin.log  # 记录binlog的文件位置
expire_logs_days = 10                   # binlog日志过期时间
max_binlog_size = 100M                  # 单个binlog允许的最大文件值
```

配置完成需要对MySQL进行重启。


#### 2. 选择哪一种日志格式

前面已经描述了3种日志格式， 在MySQL 5.7 版本中默认为`Row`格式。 `statement`格式在日志恢复以及主从复制中会出现较多的问题， 所以该格式不予考虑。 `Row`格式和`Mixed`格式基本能够满足我们的要求， 可以选择其中的一个。
此外， 在`Row`模式下还有3个可选的日志格式： `FULL`, `MINIMAL`, `NOBLOB`。 其中`FULL`格式将会记录所有的数据上下文， `MINIMAL`只会记录被修改的字段， `NOBLOB`在`text`字段下记录部分上下文， 其余为完整上下文。
当我们有足够的带宽以及磁盘空间， 并且能够保证主从复制之间的网络连接是稳定的情况下， 尽量使用`FULL`模式， 更多的信息会带来更好的恢复以及查看。 当主库的更新频率以及数量较大时， 选择`MINIMAL`以保证主从复制的低延迟性。


#### 3. 常用的查看相关配置的命令

```sql
-- 给出完整的binlog配置信息， 但是没有日志记录的路径以及二进制日志是否开启的信息
show variables like "%binlog%";

-- 给出binlog是否开启以及binlog所在磁盘位置信息
show variables like "%log_bin%";

-- 给出当前binlog写入文件名以及当前binlog的偏移量
show master status;

-- 给出所有的binlog所记录的偏移量
show master logs;

-- 在shell中对binlog进行解析， -vv参数针对于Row格式的binlog
sudo mysqlbinlog -vv mysql-bin.000005
```

#### 4. 错误的binlog数据恢复

binlog并没有我们想象中的那么复杂， 但是也没有那么容易。 如果在某个时间点数据遭到损坏， 然后使用：

```bash
myslbinlog -vv start_positon=153 end_position=724 --database=temp | mysql -uroot -p temp
```

这种方式进行恢复的话， 99%的情况下都会失败。 binlog恢复的原理与主从复制完全相同， 将一定时间区间内的SQL语句重新执行(statement)或者直接进行数据库修改(row)， 假如我们的binlog如下方所示：

```sql
### INSERT INTO `repl_test`.`new_table`
### SET
###   @1=6 /* INT meta=0 nullable=0 is_null=0 */
###   @2='haha' /* VARSTRING(45) meta=45 nullable=1 is_null=0 */
###   @3='29' /* VARSTRING(180) meta=180 nullable=1 is_null=0 */
# at 2761

### INSERT INTO `repl_test`.`student`
### SET
###   @1=3 /* INT meta=0 nullable=0 is_null=0 */
###   @2='Ray' /* VARSTRING(45) meta=45 nullable=1 is_null=0 */
###   @3='women' /* VARSTRING(135) meta=135 nullable=1 is_null=0 */
# at 3036

### UPDATE `repl_test`.`new_table`
### WHERE
###   @1=6 /* INT meta=0 nullable=0 is_null=0 */
###   @2='haha' /* VARSTRING(45) meta=45 nullable=1 is_null=0 */
###   @3='29' /* VARSTRING(180) meta=180 nullable=1 is_null=0 */
### SET
###   @1=6 /* INT meta=0 nullable=0 is_null=0 */
###   @2='haha' /* VARSTRING(45) meta=45 nullable=1 is_null=0 */
###   @3='27' /* VARSTRING(180) meta=180 nullable=1 is_null=0 */
# at 3325

### DELETE FROM `repl_test`.`new_table`
### WHERE
###   @1=5 /* INT meta=0 nullable=0 is_null=0 */
###   @2='biubiu' /* VARSTRING(45) meta=45 nullable=1 is_null=0 */
###   @3='24' /* VARSTRING(180) meta=180 nullable=1 is_null=0 */
# at 3602

### INSERT INTO `repl_test`.`new_table`
### SET
###   @1=7 /* INT meta=0 nullable=0 is_null=0 */
###   @2='cherry' /* VARSTRING(45) meta=45 nullable=1 is_null=0 */
###   @3='0' /* VARSTRING(180) meta=180 nullable=1 is_null=0 */
# at 3878
```

有一些长， 但是其实内部的结构是很简单的， 基本上就是：

```bash
new_table表插入数据 -> student表插入数据 -> new_table更新数据 -> new_table删除数据 -> new_table插入数据
```

其中`new_table删除数据`就是一个误操作， 我们需要进行挽回。 然后我们在此基础上执行：

```bash
sudo mysqlbinlog -vv mysql-bin.000007 --start-position=2761 --stop-position=3325 | mysql -uroot -pkeyerror repl_test
```

因为`DELETE`操作的偏移量为3602， 所以我们恢复上一个偏移量的数据， 也就是3325, 那么这条语句无情的抛出了异常：

```bash
ERROR 1062 (23000) at line 64: Duplicate entry '3' for key 'PRIMARY'
```

因为在此时此刻， `new_table`中id为3的数据是存在的， 并没有被删除， 那么我再重新执行这条`insert`语句， 报错是必然的。 那么这个时候怎么恢复？
非常遗憾， 这个时候这样的条件， 只能手动的一条数据一条数据的重新插入， 没有其它的办法。 那么到底该如何的使用binlog来进行自动的恢复呢？


#### 5. 正确的binlog数据恢复

binlog其实是一种增量恢复的模式， 那么既然是增量， 就必然需要有基量以及增量。 基量从哪儿来？ 来自数据库的完整备份， 即`mysqldump`。 增量从哪儿来？ 来自于上次完整备份时的binlog到现在的binlog。
可以看下面的示意图：

![Alt text](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-10-13%2011-18-32%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

当我们为机器安装MySQL并且开启了binlog之后， 就有了mysql-bin文件的产生， 如果我们不去维护binlog的话， 那么当到达了最大容量限制或者MySQL重启时会进行自动切割。 当某一天数据出现了问题，需要使用binlog进行恢复时发现， mysql-bin因为过期时间的配置前面的binlog已经不见了， 并且数据库的备份是11天以前的， 完全无法进行恢复， 欢声笑语打出GG。
通常的做法是在执行数据库备份计划的同时， `flush logs`将日志进行主动的切割， 并在本地保留所有binlog日志。 假设我们的备份是每天的凌晨3点进行整体的数据库备份， 然后切割binlog。 在当天的下午2点数据需要进行恢复， 流程如下：

```bash
1. 停止MySQL服务， 并删除对应的database。
2. 导入备份数据(这时候是凌晨3点时的数据)。
3. 找到分割的binlog， 并找到误删数据的语句， 确定偏移量。
4. 绕开误删除语句的偏移量， 进行数据恢复。
```

这里面就有了一些很重要的步骤： 必须要知道当前的备份所对应的binlog偏移量， 必须要知道误删除语句的binlog偏移量。 只有知道了这些信息才能够进行无损的数据恢复。


#### 6. 实践binlog数据恢复

首先我们需要一个更加复杂的数据库结构以应对生产环境的复杂情况， 这里的话就使用`Django`框架来自动的生成表结构以及填充数据。

```python
# 测试binlog数据恢复的本地数据模型

class BaseModel(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    updated = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class Chatroom(BaseModel):
    username = models.CharField(max_length=20, unique=True)
    nickname = models.CharField(max_length=50)
    description = models.CharField(max_length=200)
    owner = models.CharField(max_length=20)
    head_img = models.URLField(max_length=500)


class ChatroomMember(BaseModel):
    username = models.CharField(max_length=20, unique=True)
    nickname = models.CharField(max_length=50)
    head_img = models.URLField(max_length=500)
    city = models.CharField(max_length=50)
    gender = models.BooleanField()
    description = models.CharField(max_length=200)


class ChatroomRelation(BaseModel):
    chatroom = models.ForeignKey(Chatroom, on_delete=models.SET_NULL, null=True)
    member = models.ForeignKey(ChatroomMember, on_delete=models.SET_NULL, null=True)
    joined = models.DateTimeField(auto_now_add=True)
    chatroom_nickname = models.CharField(max_length=50)

    class Meta:
        unique_together = ("chatroom", "member")
```

在填充数据之后进行`mysqldump`进行备份， 需要注意的是一定要记录当前dump的binlog偏移量， 即添加`--master-data`参数， 并且在备份时仍有数据进行插入

```bash
mysqldump --master-data --single-transaction -uroot -pkeyerror myProjects > /home/smartkeyerror/mysql_dumps/test.sql
```

打开`test.sql`， 可以看到当前备份的偏移量：

```bash
CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000007', MASTER_LOG_POS=99082;

# client-session查看
mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |      3370 |
| mysql-bin.000002 |     19108 |
| mysql-bin.000003 |       177 |
| mysql-bin.000004 |       201 |
| mysql-bin.000005 |    115978 |
| mysql-bin.000006 |       201 |
| mysql-bin.000007 |    125900 |
| mysql-bin.000008 |       154 |
+------------------+-----------+
8 rows in set (0.00 sec)
```

因为在测试的过程中突然断电了(刚好遇到了一个特殊情况)， 所以binlog日志又被切割出去了， 一个很奇怪的现象， 起始位置不是0， 而是154， 我们再切割一个：

```bash
mysql> flush logs;
Query OK, 0 rows affected (0.07 sec)

mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |      3370 |
| mysql-bin.000002 |     19108 |
| mysql-bin.000003 |       177 |
| mysql-bin.000004 |       201 |
| mysql-bin.000005 |    115978 |
| mysql-bin.000006 |       201 |
| mysql-bin.000007 |    125900 |
| mysql-bin.000008 |       201 |
| mysql-bin.000009 |       154 |
+------------------+-----------+
9 rows in set (0.00 sec)
```

`mysql-bin.000008`多了47个偏移， 新切割的binlog起始偏移仍然是154。
回到数据恢复， 在备份时我们知道了日志的偏移量为99082， 并且当时的binlog为`mysql-bin.000007`， 首先我们先删除一部分数据， 然后再写入一些数据。
时序图：

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-10-13%2016-13-53%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

那么我们只需要做两件事：找到删除数据的binlog文件位置以及偏移量， 删库(这回不用跑路)后恢复数据。
那就找呗， 在0007 99082至009 38924之间查找`DELETE FROM`, 通过使用`grep`:

```bash
# -C参数会在匹配的关键字前后各打印30行的数据
sudo mysqlbinlog mysql-bin.000007 -vv | grep -C 30 "DELETE FROM"
sudo mysqlbinlog mysql-bin.000008 -vv | grep -C 30 "DELETE FROM"
```

最终在009 451至009 943发现了`DELETE FROM`语句， 那么我们就可以开始恢复数据了。

```bash
1. DROP DATABASE myProjects

2. 导入数据， 抛了异常
ERROR 3021 (HY000) at line 22: This operation cannot be performed with a running slave io thread; run STOP SLAVE IO_THREAD FOR CHANNEL '' first.

3. 运行 STOP SLAVE IO_THREAD FOR CHANNEL "";

4. 重新导入数据

5. 恢复0007 99082到尾部的数据
sudo mysqlbinlog --start-position=99082 -vv mysql-bin.000007 | mysql -uroot -pkeyerror myProjects

6. 恢复008以及009 451之前的数据
这里有一个如何选择结束点的问题， 首先来看一下grep信息， 选择370报错， 选择287正常， 也不知道为什么

BEGIN
/*!*/;
# at 287
#181013 15:41:44 server id 2  end_log_pos 370 CRC32 0x4d8bbd3e 	Table_map: `myProjects`.`devOps_chatroomrelation` mapped to number 109
# at 370
#181013 15:41:44 server id 2  end_log_pos 451 CRC32 0xf7657cfe 	Delete_rows: table id 109 flags: STMT_END_F

BINLOG '
uKHBWxMCAAAAUwAAAHIBAAAAAG0AAAAAAAEACm15UHJvamVjdHMAF2Rldk9wc19jaGF0cm9vbXJl
bGF0aW9uAAcDEhISDwMDBQYGBsgAYD69i00=
uKHBWyACAAAAUQAAAMMBAAAAAG0AAAAAAAEAAgAH/4BqAAAAmaEa7SEEkKWZoRrtIQSQ5pmhGu0h
BJFGCGFva0pWcEdPJQAAABMAAAD+fGX3
'/*!*/;
### DELETE FROM `myProjects`.`devOps_chatroomrelation`
.... # 中间被我干掉了
# at 451
#181013 15:41:44 server id 2  end_log_pos 534 CRC32 0xade8611d 	Table_map: `myProjects`.`devOps_chatroomrelation` mapped to number 109
# at 534
#181013 15:41:44 server id 2  end_log_pos 615 CRC32 0x55e89f5b 	Delete_rows: table id 109 flags: STMT_END_F

sudo mysqlbinlog -vv mysql-bin.000008 | mysql -uroot -pkeyerror myProjects

sudo mysqlbinlog -vv mysql-bin.000009 --stop-position=287 | mysql -uroot -pkeyerror myProjects

sudo mysqlbinlog -vv mysql-bin.000009 --start-position=1119 | mysql -uroot -pkeyerror myProjects

7. 确认数据正常恢复， 并开启对外服务。
```


#### 7. 总结

上面就是使用binlog进行数据恢复的整个过程， 没有什么特别复杂的地方， 只不过步骤比较繁琐， 偏移量必须准确才能够成功。 对一些比较重要的操作进行整理：
1. 备份时必须添加`--master-data`参数； `-F`切割二进制日志可选， 添加之后会更加的方便
2. 必须找到误删数据的偏移量， 备份时绕过这些数据修改。
3. 因流程较为复杂， 确认无误后进行操作。

另外使用`mysqlbinlog`对binlog进行解析的话查看起来并不是很方便， 可以在`client`中使用

```sql
show binlog events [in 'binlog-name'] [from position] \G;
```

来更加直观的查看相关内容以及偏移量， 不过这种方式查看的话没有具体数据信息， 可以结合`mysqlbinlog`命令共同使用。