---
layout: post
cover: false
navigation: true
title: MySQL权限管理
date: 2018-10-30 09:49:09
tags: ['MySQL']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

`MySQL`的权限管理重要性等同于服务器数据的重要性， 权限体系如果建立的不到位的话， 也就意味着生产数据处于危险状态。

<!---more--->

#### 1. Ubuntu 16.04 下安装MySQL

```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install mysql-server mysql-client
```

#### 2. 修改相关的配置文件
```bash
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf

bind-address = 0.0.0.0
max_connections = 3000

# binlog相关配置
server-id = 2
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 10
max_binlog_size = 100M

# 慢查询日志配置
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 2
```
编辑并保存后重启MySQL服务即可。另外需要在系统层面赋予MySQL最大文件描述符的配置：
```bash
sudo vim /lib/systemd/system/mysql.service

# 添加
LimitNOFILE=65535
LimitNPROC=65535
```

#### 3. 管理权限体系的建立
通常MySQL的权限分级与开发团队的组织形式直接相关， 包括初级工程师(只能select部分database), 中级工程师(能够对数据进行增删改查)， 高级开发工程师(全部权限)， 但是在现代的web开发， 使用ORM框架的条件下， 对应用的账号则必须赋予较高的权限， 那么能够查看代码的开发人员依然能够拿到较高权限。 所以MySQL的权限控制需要和Linux权限控制相结合： 涉及生产服务器MySQL数据库连接代码应只存放于服务器中， 并对其进行高权限控制， 即该文件只能由高级开发工程师进行查看或者修改， 以保证权限的统一。

##### 3.1 创建用户并赋予权限
创建用户使用`GRANT`命令， 常见参数及说明如下表所示：

| GRANT | ALL PRIVILEGES | ON database.table | TO user@address | IDENTIFIED BY 'password'
| -- |
| 创建用户命令 | 赋予什么权限 | 哪个数据库以及哪张表 | 用户名和登录地址 | 密码


创建smart用户， 并赋予其全部数据库的全部权限， 但只能在ip地址为144.35.177.20的机器上进行登录
```sql
mysql> GRANT ALL PRIVILEGES ON *.* TO 'smart'@'144.35.177.20' IDENTIFIED BY 'passwd';
mysql> FLUSH PRIVILEGES;

mysql> select * from mysql.user where User="smart" \G;
*************************** 1. row ***************************
                  Host: 144.35.177.20
                  User: smart
           Select_priv: Y
           Insert_priv: Y
           Update_priv: Y
           Delete_priv: Y
           Create_priv: Y
             Drop_priv: Y
           Reload_priv: Y
         Shutdown_priv: Y
          Process_priv: Y
          ...  中间省略
1 row in set (0.00 sec)
```
在赋予`ALL PRIVILEGES`之后该用户除了不能创建用户以外， 用于对数据库的所有控制权限。 这样的账号应该只有运维经理以及高级开发才能拥有。

下面来看一下MySQL具体都有哪些权限:

| 维度 | 权限 |
| - | - |
| 数据 | insert, select, update, delete |
| 表 | create, drop, references, index, alter, lock table, create temporary table |
| 主从复制 | replication client, replication slave |
| 服务器 | shutdown, processlist, grant, super, create role |

该表简单的对权限进行了一个分类， 作用仅是便于记忆而已。 根据`MySQL`官方文档所给出的权限以及每种权限的作用， 下面进行详细的整理。

- ALTER: 允许更改表结构， 例如添加索引， 删除字段等。 `ALTER TABLE`语句需要有CREATE以及INSERT权限。 当我们对一个表进行重命名的时候， 需要有ALTER和DROP以及CREATE， INSERT的权限。
- ALTER ROUTINE: 使用ALTER存储过程的权限。
- CREATE: 创建表的权限。
- CREATE ROLE: 创建角色的权限， 当一个用户拥有了CREATE USER权限时， 该权限随即拥有。
- CREATE ROUTINE: 允许使用CREATE存储过程。
- CREATE TEMPORARY TABLES: 允许使用`CREATE TEMPORARY TABLE`语句来创建临时表， 这张临时表的 DROP TABLE， INSERT， UPDATE等权限随之赋予。
- CREATE USER: 能够使用 ALTER USER， CREATE ROLE， CREATE USER， DROP ROLE， DROP USER, RENAME USER， 以及REVOKE ALL PRIVILEGES等语句， 属于管理员权限。
- CREATE VIEW: 允许创建视图。
- DELETE: 允许从表中删除数据。
- DROP: 删除整张表以及数据的权限， TRUNCATE TABLE命令需要有该权限才能够执行。
- DROP ROLE: 允许删除某一个用户， 当用户具有了CREATE USER的权限时， 该权限随之赋予。
- GRANT OPTION: 允许为用户添加权限。
- INDEX: 允许添加和删除索引， 当用于具有CREATE的权限时， 该权限随之赋予。
- INSERT: 向表中插入数据的权限。
- LOCK TABLES: 锁表的权限。
- PROCESS: 允许查询当前数据库所运行的后台线程信息， 例如主从复制线程信息。
- RELOAD: 允许运行flush-xxx相关命令， 包括刷新权限， 日志等， 属于管理员权限。
- REPLICATION CLIENT： 允许使用SHOW MASTER STATUS, SHOW SLAVE STATUS, SHOW BINARY LOGS等语句， 主要用于主从复制。
- REPLICATION SLAVE: 允许更新主库的变化。
- SUPER: 这是一个相当重要的权限， 权限非常大， 并且`MySQL`在将来的版本中将会移除这个权限， 的确， 该权限比较危险。 SUPER权限能够在运行时修改系统变量， 该更主从复制相关信息， 以及更重要的， 该权限可以在`MySQL Session`中删除binlog日志文件， 即`purge master logs`命令。
- UPDATE: 更新表数据的权限。

可以看到， 比较危险的权限也就是DROP, CREATE USER， GRANT OPTION， RELOAD以及SUPER权限， 其中最危险的就是SUPER权限， **除了管理员以外， 任何账号均不能有SUPER权限**。


###### 3.1.1 创建mysqldump账户并赋予权限
`mysqldump`命令的权限讲实话我并不清楚， 查阅网上相关资料之后进行创建并赋予：

```sql
create user 'dumper'@'localhost' identified by 'passwd';
grant select on myProjects.* to 'dumper'@'localhost';
grant show view on myProjects.* to 'dumper'@'localhost';
grant lock tables on myProjects.* to 'dumper'@'localhost';
grant trigger on myProjects.* to 'dumper'@'localhost';
// 如果需要在备份时刷新二进制日志， 还需要以下权限
grant reload on myProjects.* to 'dumper'@'localhost';
grant replication slave, replication client on myProjects.* to 'dumper'@'localhost';
```
那么此时使用该账户进行登录， 进行查看`myProjects`库， 其余的增删改操作均不能进行。


###### 3.1.2 创建主从复制的账号并赋予权限
```sql
grant replication slave, replication client on *.* to repl@'192.168.0.%' dentified by 'passwd';
```
这里使用`%`通配符来对该局域网内的所有机器进行权限的赋予。

###### 3.1.3 创建项目账号并赋予权限
```sql
grant all privileges on myProjects.* to 'shop'@'192.168.1.6' identified by 'complex-passwd';
```

#### 3.2 查看某个用户的权限
```sql
show grants for 'user'@'address';
```

#### 3.3 回收用户权限以及删除用户
回收用户所有权限
```sql
revoke all on *.* from 'user'@'address';
```
回收用户部分权限
```sql
revoke drop on myProjects.* from 'user'@'address';
```
删除用户
```sql
drop user 'user'@'address';
```

修改某个用户的密码

```sql
update mysql.user set authentication_string=password("passwd") where user="user";
```

在执行完上面儿的语句之后， 尽量的执行`flush privileges;`命令刷新一下权限。

#### 4. 关于权限管理的一些杂谈
MySQL并没有提供不给用户授予什么权限的命令， 也就是没有`exclude`这种语法， 但是我们可以先给用户授予全部的权限， 然后将不必要的权限进行回收。 像`drop`这种很危险的权限就不要随便给， 如果库里面儿只有逻辑删除的话， `delete`权限都可以不给， 有需要的时候让管理员进行协助处理。 尽可能的用最小权限做更安全的事情， 毕竟使用`binlog`进行数据恢复也不可能保证100%成功， 将危险扼杀在摇篮里才是正解。

权限管理对于管理员来讲确实是比较麻烦的一件事情， 很多团队`root`账号满天跑， 包括笔者在内的团队在初期也是这样的。 在付出了血淋淋的代价之后才开始对权限进行管理， 亡羊补牢为时尚晚， 埋过的雷总会炸的。


