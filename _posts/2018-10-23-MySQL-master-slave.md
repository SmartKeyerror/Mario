---
layout: post
cover: false
navigation: true
title: MySQL之主从复制
date: 2018-10-23 09:49:09
tags: ['MySQL']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---


MySQL的主从复制是建立读写分离以及MySQL集群的根本， 为了能够保证复制的正常运行， 那么就必然需要对其原理以及核心的配置项有足够的了解， 才能够在复杂的生产环境中对错误进行排查。

<!---more--->


#### 1. MySQL主从复制原理

MySQL之间数据复制的基础是`二进制日志`文件(binary log file)。 一台MySQL数据库一旦启用二进制日志后， 其作为master， 它的数据库中所有操作都会以“事件”的方式记录在二进制日志中， 其他数据库作为slave通过一个`I/O线程`与主服务器保持通信， 并监控master的二进制日志文件的变化， 如果发现master二进制日志文件发生变化， 则会把变化复制到自己的`中继日志中`， 然后slave的一个`SQL线程`会把相关的“事件”执行到自己的数据库中， 以此实现从数据库和主数据库的一致性，也就实现了主从复制。

![Alt text](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E5%8E%9F%E7%90%86.png)


#### 1. 实现MySQL主从复制需要进行的配置
1. 主服务器：
- 开启二进制日志
- 配置唯一的server-id
- 获得master二进制日志文件名及位置
- 创建一个用于slave和master通信的用户账号
2. 从服务器：
- 配置唯一的server-id
- 使用master分配的用户账号读取master二进制日志
- 启用slave服务

#### 2. binlog相关配置以及参数说明
```bash
mysql> show variables like "%binlog%";
+--------------------------------------------+----------------------+
| Variable_name                              | Value                |
+--------------------------------------------+----------------------+
| binlog_cache_size                          | 32768                |
| binlog_checksum                            | CRC32                |
| binlog_direct_non_transactional_updates    | OFF                  |
| binlog_error_action                        | ABORT_SERVER         |
| binlog_format                              | ROW                  |
| binlog_group_commit_sync_delay             | 0                    |
| binlog_group_commit_sync_no_delay_count    | 0                    |
| binlog_gtid_simple_recovery                | ON                   |
| binlog_max_flush_queue_time                | 0                    |
| binlog_order_commits                       | ON                   |
| binlog_row_image                           | FULL                 |
| binlog_rows_query_log_events               | OFF                  |
| binlog_stmt_cache_size                     | 32768                |
| binlog_transaction_dependency_history_size | 25000                |
| binlog_transaction_dependency_tracking     | COMMIT_ORDER         |
| innodb_api_enable_binlog                   | OFF                  |
| innodb_locks_unsafe_for_binlog             | OFF                  |
| log_statements_unsafe_for_binlog           | ON                   |
| max_binlog_cache_size                      | 18446744073709547520 |
| max_binlog_size                            | 104857600            |
| max_binlog_stmt_cache_size                 | 18446744073709547520 |
| sync_binlog                                | 1                    |
+--------------------------------------------+----------------------+
22 rows in set (0.00 sec)
```
通过在mysql客户端执行`show variables`命令， 可以看到关于`binlog`的配置一共有20几项之多。 但是大部分的配置项我们可以直接使用默认值， 有几个配置需要额外的进行关注：

##### 2.1 binlog_format
二进制日志格式有3种格式可选： Statement, Row以及Mixed。 其中Statement格式基于语句进行日志记录， Row格式基于数据修改进行记录。
首先准备一下测试的数据库：
```sql
create database repl_test;

CREATE TABLE `repl_test`.`new_table` (
  `id` INT NOT NULL,
  `name` VARCHAR(45) NULL,
  `age` VARCHAR(45) CHARACTER SET 'utf8mb4' NULL,
  PRIMARY KEY (`id`));
```
###### 2.1.1 Statement

简单的来说Statement格式就是记录了数据修改所执行的SQL语句， 那么在做主从复制时从库读取SQL语句并重新进行执行。
我们创建一个database以及一个table， 并在table中插入一些数据来观察一下：
```sql
set session binlog_format=statement;

insert into new_table values (1, "smart", 18), (2, "keyerror", 25);
delete from new_table where id=2;
```
在二进制日志保存的文件夹中查看二进制日志：
```bash
sudo mysqlbinlog mysql-bin.000001  # Statement格式日志
sudo mysqlbinlog -vv mysql-bin.000001  # Row格式日志
```
那么Statement格式的日志就会是这个样子：
```bash
insert into new_table values (1, "smart", 18), (2, "keyerror", 25)
....  # 中间一些其它内容
BEGIN
/*!*/;
# at 2297
#181011 10:16:38 server id 1  end_log_pos 2413 CRC32 0xccb8b51b         Query   thread_id=2     exec_time=0     error_code=0
SET TIMESTAMP=1539224198/*!*/;
delete from new_table where id=2
/*!*/;
# at 2413
...
```
可以看到在日志中完整了记录了每一条SQL语句的内容， 从库拿到这些语句重新执行就可以获得与主库相同的数据了。
因为记录的是SQL语句， 那么会极大的降低二进制日志文件的大小， 并且在复制的有着更快的网络传输效率。
缺点也显而易见：像Uuid()这样的函数每次执行返回不同的结果， 那么这样一来在主库和从库中数据就会有不一致的情况。 并且如果某一条SQL语句执行时间过长， 从库同样的也要执行很长时间， 这样一来复制的过程就可能会被阻塞， 主从之间的数据一致性在这段时间就会遭到破坏。 所以一般在生产环境中并不会使用这样的日志格式， 除非有特殊的需求需要进行临时的修改。

###### 2.1.2 Row

Row格式在MySQL5.7版本中为默认的二进制日志格式， 日志中会记录每一行数据的修改， 然后在从库中应用这些修改。
```sql
set session binlog_format=row;
insert into new_table values (3, "zhangsan", 18);
update new_table set age=19 where age=18;
```
那么此时日志所记录的内容为：
```bash
### INSERT INTO `repl_test`.`new_table`
### SET
###   @1=3 /* INT meta=0 nullable=0 is_null=0 */
###   @2='zhangsan' /* VARSTRING(45) meta=45 nullable=1 is_null=0 */
###   @3='18' /* VARSTRING(180) meta=180 nullable=1 is_null=0 */

### UPDATE `repl_test`.`new_table`
### WHERE
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2='smart' /* VARSTRING(45) meta=45 nullable=1 is_null=0 */
###   @3='18' /* VARSTRING(180) meta=180 nullable=1 is_null=0 */
### SET
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2='smart' /* VARSTRING(45) meta=45 nullable=1 is_null=0 */
###   @3='19' /* VARSTRING(180) meta=180 nullable=1 is_null=0 */
### UPDATE `repl_test`.`new_table`
### WHERE
###   @1=3 /* INT meta=0 nullable=0 is_null=0 */
###   @2='zhangsan' /* VARSTRING(45) meta=45 nullable=1 is_null=0 */
###   @3='18' /* VARSTRING(180) meta=180 nullable=1 is_null=0 */
### SET
###   @1=3 /* INT meta=0 nullable=0 is_null=0 */
###   @2='zhangsan' /* VARSTRING(45) meta=45 nullable=1 is_null=0 */
###   @3='19' /* VARSTRING(180) meta=180 nullable=1 is_null=0 */
```
可以看到每一行的修改都被记录了， 并且记录了修改行的前后的数据内容， 那么这样一来主库与从库就能够达到完全的数据一致性。
缺点也同样的显而易见， 数据记录的太多太啰嗦， 会占用大量的磁盘空间以及更长时间的网络传输。 但是， 是有办法优化的， 要不然MySQL官方也不会推荐我们使用Row格式。

###### 2.1.1 Mixed

实际上就是上面两种模式的混合。


##### 2.1 binlog_row_image
前面提到了`binlog_format=ROW`会带来很大的磁盘以及网络传输开销， 那么`binlog_row_image`参数就是为了优化`ROW`模式而存在的。
可选值有3个： FULL， MINIMAL, NOBLOB。其中FULL选项将会记录所有内容； MINIMAL仅会记录被修改了列， 无关列不会记录； NOBLOB记录了blog和text之外的所有字段。FULL模式所生成的日志格式就是上面我们看到的。
MINIMAL选项所生成的日志格式：
```sql
set session binlog_row_image=MINIMAL;
update new_table set age=25 where id=1;
```
```bash
### UPDATE `repl_test`.`new_table`
### WHERE
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
### SET
###   @3='25' /* VARSTRING(180) meta=180 nullable=1 is_null=0 */
```
其中`@1, @3`表示第几列， 也就是字段名称。这里的日志表示"将id(@1)为1的数据的年龄(@3)修改为25"， 其余未改动的字段(列)并没有记录在二进制日志中， 减少了日志的记录。

需要特别注意的是： 虽然MINIMAL能够减少日志的数量， 但是由于记录会选择的缺失， 那么通过这种格式的二进制日志对数据库进行恢复的难度就会提高， 排错也会有一些困难。 需要根据实际情况来正确的选择是使用`FULL`模式还是`MINIMAL`模式。


#### 3. 基于日志点的复制和基于GTID的复制
##### 3.1 基于日志点的复制
基于日志点的复制MySQL会记录当前日志的数据偏移量并且将该值传递给从库， 从库根据该偏移量进行复制。
通常来讲我们启用主从复制是在已经有了主库的情况下而添加从库的， 那么此时就需要将数据导入到从库中：
```bash
# 只允许读操作不允许写入
flush tables with read lock;
# 记录日志的偏移量
mysqldump --master-data [databses | --all-databases] > all.sql
or
xtrbackup --slave-info  # 第三方工具， 仅用于InnoDB， 属于热备工具
```
那么此时从库就可以开启复制链路了， 其中的`master_log_file`以及`master_log_pos`可以在刚才的数据库备份中找到：
```sql
将数据进行导入
change master to master_host="192.168.0.5",
                 master_user="",
                 master_password="",
                 master_log_file="二进制日志文件名",
                 master_log_pos=偏移量
start slave;

主库执行： unlock tables；
```

优点： 基于日志点的复制是MySQL最早支持的复制技术， 相对而言BUG比较少， 并且对SQL没有任何的限制， 故障处理较为容易。
缺点： 故障转移时重新获取新主的日志点信息比较困难。

##### 3.2 基于GTID的复制
基于GTID的复制原理上其实是执行每一个事务， 主库中每一个事务都有一个唯一的自增标志。
```bash
GTID = server_uuid:transaction_id
```
借助GTID， 在发生主备切换的情况下， MySQL的其它Slave可以自动在新主上找到正确的复制位置， 不再需要像基于日志点复制一样人工的添加日志记录的偏移量， 简化了复杂复制拓扑下集群的维护。 另外，基于GTID的复制可以忽略已经执行过的事务，减少了数据发生不一致的风险。
但是这种模式同样具有缺点： 其故障处理较为复杂， 并且对执行的SQL有一定的限制。


