---
layout: post
cover: false
navigation: true
title: MySQL慢查询日志分析工具
date: 2018-10-25 09:49:09
tags: ['MySQL']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

慢查询日志使我们对MySQL进行性能优化的关键指标， 只有在确定了哪些查询的确是慢查询之后才能对症下药， 进行性能优化， 而不是凭自身的感觉去判断， 结果有事往往出乎意料。 直接打开慢查询日志进行查看效率比较低效， 所以需要借助`pt-query-digest`工具来进行分析。

<!---more--->

#### 0. Degine: MySQL Version: 5.7

#### 1. Percona Toolkit工具包的下载与安装
`Percona Toolkit`工具包包含了`pt-slave-delay`， `pt-query-digest`， `pt-mysql-summary`等非常有用的工具。
> https://www.percona.com/downloads/percona-toolkit/LATEST/

找对对应的版本以及平台进行安装即可。

#### 2. pt-query-digest的基本使用
首先来看一下分析的基本结果：
```bash
pt-query-digest --report mysql-slow.log
```
核心输出结果如下：
```sql
# Current date: Tue Oct 16 11:24:55 2018
# Hostname: Zero
# Files: mysql-slow.log
# Overall: 7.77k total, 94 unique, 0.00 QPS, 0.02x concurrency ___________
# Time range: 2018-08-18 14:11:31 to 2018-10-16 10:01:13
# Attribute          total     min     max     avg     95%  stddev  median
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time         77201s   500ms    166s     10s     37s     15s      4s
# Lock time           203s       0     22s    26ms   138us   500ms    73us
# Rows sent          2.46M       0  78.34k  331.51  511.45   4.37k    0.99
# Rows examine      63.82G       0  52.93M   8.41M  28.56M  11.25M   3.68M
# Query size         5.06M       6  16.48k  682.18   2.06k  724.05  420.77
```
可以看到结果还是比较直观的， 包括执行时间， 加锁时间等， 横坐标中最有价值的信息为`95%`， 意义为将查询从小到大排列， 取位于整个排列的95%位置， 这样以来就会滤掉较大的值， 减少极值对统计的影响。 可以看到上面的输出`Exec time - 95% `输出结果为37s， 数值比较大， 所以完全有必要进行慢查询的优化。

继续来看输出：
```sql
# Profile
# Rank Query ID                      Response time    Calls R/Call  V/M
# ==== ============================= ================ ===== ======= =====
#    1 0x530A2CE72ED76F6FD03452E6... 20003.2369 25.9%   478 41.8478 17.77 SELECT lv_pt_order lv_pt_order_detail lv_pt_goods lv_user
#    2 0x1B1C071EBB0DECB4FCB8B4C9... 19870.1927 25.7%   855 23.2400  6.26 SELECT lv_pt_order lv_pt_order_detail lv_pt_goods lv_user
#    3 0x84CAC95FCB28351DEB798161...  9187.0368 11.9%   727 12.6369  8.70 SELECT lv_pt_order lv_pt_order_detail lv_pt_goods lv_user
#    4 0x32D67A543AD02B4178806916...  6330.5023  8.2%  1545  4.0974  5.78 UPDATE lv_session
```
其中`Query ID`是每一个查询的哈希值指纹， `Response time`包括所有查询的返回时间以及占比， `Calls`为查询的次数， `R/Call`为该查询的平均时间， `V/M`为方差均值比， 表示该查询的返回时间波动性， 该值越大越有优化的价值。

再往下面就是对`Profile`中的每一条进行的详细分析。
```sql
# Query 1: 0.00 QPS, 0.01x concurrency, ID 0x530A2CE72ED76F6FD03452E68015715F at byte 6465751
# This item is included in the report because it matches --limit.
# Scores: V/M = 17.77
# Time range: 2018-09-20 18:23:33 to 2018-10-13 00:03:20
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count          6     478
# Exec time     25  20003s     12s    166s     42s     97s     27s     32s
# Lock time      0    86ms    58us    23ms   179us   144us     1ms   108us
# Rows sent      0   8.98k      10      20   19.25   19.46    2.54   19.46
# Rows examine  24  15.74G  24.48M  52.93M  33.71M  51.29M  10.49M  27.20M
# Query size     3 183.98k     393     404  394.14  400.73    5.50  381.65
# String:
# Databases    we8
# Hosts
# Users        root
# Query_time distribution
#   1us
#  10us
# 100us
#   1mss
#  10ms
# 100ms
#    1s
#  10s+  ################################################################
# Tables
#    SHOW TABLE STATUS FROM `we8` LIKE 'lv_pt_order'\G
#    SHOW CREATE TABLE `we8`.`lv_pt_order`\G
#    SHOW TABLE STATUS FROM `we8` LIKE 'lv_pt_order_detail'\G
#    SHOW CREATE TABLE `we8`.`lv_pt_order_detail`\G
#    SHOW TABLE STATUS FROM `we8` LIKE 'lv_pt_goods'\G
#    SHOW CREATE TABLE `we8`.`lv_pt_goods`\G
#    SHOW TABLE STATUS FROM `we8` LIKE 'lv_user'\G
#    SHOW CREATE TABLE `we8`.`lv_user`\G
# EXPLAIN /*!50100 PARTITIONS*/
SELECT `o`.*, `od`.`attr`, `od`.`num`, `od`.`pic`, `od`.`goods_name`, `g`.`name` AS `goods_name`, `u`.`nickname` FROM `lv_pt_order` `o` LEFT JOIN `lv_pt_order_detail` `od` ON od.order_id=o.id LEFT JOIN `lv_pt_goods` `g` ON g.id=od.goods_id LEFT JOIN `lv_user` `u` ON u.id=o.user_id WHERE ((`o`.`is_delete`=0) AND (`o`.`store_id`=5)) AND (`o`.`is_cancel`=0) ORDER BY `o`.`addtime` DESC LIMIT 20\G
```
可以看到这条详细分析对应着`Profile`中的`Rank 1`, 其中`pct`即percentage， 表示该查询在该文件中的占比， 其余的基本大同小异。
在表格下方有一个直方图， 表示该查询的时间分布情况， 再往下就是实际的慢查询语句。
可以很直观的看到这条SQL语句的平均执行时间为42s， 最大查询时间166s,  总计执行了478次， 也算是颠覆了我对慢查询时间的认知， 我以为2s的查询就已经很糟糕了， 没想到这里竟然有100+s。


#### 3. pt-query-digest筛选参数
上面儿的内容基本上就是`pt-query-digest`所能够产生的结果， 另外可以添加一些参数来进行筛选。

```bash
# 给出最近1个小时的慢查询分析结果
pt-query-digest --report --since 3600s mysql-slow.log

# 给出一个时间区间的慢查询分析结果
pt-query-digest --report --since '2018-09-20 18:23:33' --until '2018-10-13 00:03:20' mysql-slow.log

# 给出某个用户的所有慢查询， 在设计应用时， 前台和后台可以使用不同的MySQL账户
pt-query-digest --filter '($event->{user} || "") =~ m/^root/i' mysql-slow.log
```

更多的内容请见官网。

> https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html#
