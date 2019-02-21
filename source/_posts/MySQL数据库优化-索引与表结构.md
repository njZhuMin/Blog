---
title: MySQL数据库优化-索引与表结构
date: 2016-06-08 20:31:16
tags: [mysql,optimize]
categories:
- database
- RelationalDB
---

> 数据库系列文章：
> https://zhum.in/blog/categories/database/

> 示例数据库：sakila database
> http://downloads.mysql.com/docs/sakila-db.tar.gz

> 日志分析工具：pt-query-digest
> https://www.percona.com/doc/percona-toolkit/2.2/pt-query-digest.html#downloading
- - -

# 索引优化
## 如何选择合适的列建立索引
1. 在where，group by，order by，on从句中出现的列（包含所有记录的索引称为覆盖索引）
2. 索引字段越小越好（因为数据库的存储单位是页，一页中能存下的数据越多，则I/O效率越高）
3. 离散度高得列放在联合索引前面
例如：
```sql
SELECT * FROM payment WHERE staff_id = 2 AND customer_id = 584;
```
对于这样的语句建立联合索引，是`index(staff_id, customer_id)`好还是`index(customer_id， staff_id)`好？

<!-- more -->

判断离散程度：
```sql
SELECT COUNT(distinct customer_id), count(distinct staff_id) FROM payment;
```
输出结果：
```bash
+-----------------------------+--------------------------+
| COUNT(distinct customer_id) | count(distinct staff_id) |
+-----------------------------+--------------------------+
|                         599 |                        2 |
+-----------------------------+--------------------------+
```
可以看出，唯一值越多，则离散程度越高。因此customer_id的离散程度更高，应该放在联合索引的前面。

## 索引的维护及优化
通常认为使用索引可以提高查询效率，也会降低写入效率。但实际上过多的冗余索引不仅影响写入语句的效率，也会影响查询的效率。这是由于查询分析的时候首先需要选择使用的索引，过多的索引会导致分析过程缓慢，从而影响查询效率。

- 如果相同的列以相同的顺序建立的同类型的索引，那么就产生了重复索引。例如下表中的`primary key`和`ID`列上的索引就是重复索引：
```sql
CREATE TABLE test(
	id INT NOT NULL PRIMARY KEY,
	name VARCHAR(10) NOT NULL,
	title VARCHAR(50) NOT NULL,
	unique(id)
) ENGINE=Innodb;
```

- 如果多个索引的前缀列相同，或是在联合索引中包含了主键的索引，那么就产生了冗余索引（Innodb引擎会在索引后自动追加主键索引）。例如下表中的`key(name,id)`就是一个冗余索引：
```sql
CREATE TABLE test(
	id INT NOT NULL PRIMARY KEY,
	name VARCHAR(10) NOT NULL,
	title VARCHAR(50) NOT NULL,
	key(name, id)
) ENGINE=Innodb;
```

在`information_schema`数据库下执行该SQL语句查询冗余或重复索引：
```sql
SELECT a.TABLE_SCHEMA AS '数据名', a.TABLE_NAME AS '表名', a.INDEX_NAME AS '索引1', b.INDEX_NAME AS '索引2', a.COLUMN_NAME AS '重复列名' FROM STATISTICS a
	JOIN STATISTICS b ON a.TABLE_SCHEMA = b.TABLE_SCHEMA AND a.TABLE_NAME = b.TABLE_NAME AND a.SEQ_IN_INDEX = b.SEQ_IN_INDEX AND a.COLUMN_NAME = b.COLUMN_NAME
		WHERE a.SEQ_IN_INDEX = 1 AND a.INDEX_NAME <> b.INDEX_NAME;
```

也可以使用`pt-duplicate-key-checker -u root -p 'password' -h 127.0.0.1`查询冗余信息。

- 有时由于业务变更或者表结构的变更，数据库中可能存在不再需要使用的索引，可以进行删除。
目前MySQL中还没有记录索引的使用情况，但在PerconMySQL和MariaDB中可以通过`INDEX_STATICS`表来查看哪些索引未被使用。在MySQL中目前只能通过慢查日志配合`pt-index-usage`工具来进行索引情况的分析：
```bash
pt-index-usage -u root -p 'password' mysql-slow.log
```

# 数据库结构优化
## 选择合适的数据类型
1. 使用可以存下数据的最小的数据类型
2. 使用简单的数据类型，int要比varchar类型在MySQL处理上简单
3. 尽可能的使用NOT NULL定义字段
4. 尽量少用TEXT类型，非用不可时最好考虑分表

- 使用INT类型存储日期时间
使用`FROM_UNIXTIME()`、`UNIX_TIMESTAMP()`两个函数来进行转换：
```sql
CREATE TABLE test(
	id INT AUTO_INCREMENT NOT NULL,
	timestr INT,
	PRIMARY KEY(id));
INSERT INTO test(timestr) VALUES(UNIX_TIMESTAMP('2016-05-04 13:12:00'));
SELECT FROM_UNIXTIME(timestr) FROM test;
```

- 利用BIGINT来储存IP地址
使用`INET_ATON()`、`INET_NTOA()`两个函数进行转换：
```sql
CREATE TABLE sessions(
	id INT AUTO_INCREMENT NOT NULL,
	ipaddress BIGINT,
	PRIMARY KEY(id));
INSERT INTO sessions(ipaddress) VALUES(INET_ATON('192.168.0.1'));
SELECT INET_NTOA(ipaddress) FROM sessions;
```

## 表的范式化优化
范式是数据库设计的规范，目前说到范式化一般是指第三范式：要求数据库中不存在非关键字段对任意候选关键字的传递函数依赖。

例如：

|商品名称|价格|规格|有效期|分类|分类描述|
|-------|----|----|-----|----|-------|
| 可乐 | 3.00|250ml|2016.06|饮料|碳酸饮料|
| 雪碧 | 3.00|250ml|2016.08|饮料|碳酸饮料|

这个表中存在传递函数依赖关系：`商品名称 -> 分类 -> 分类描述`，也就是说存在非关键字段`分类描述`对关键字`商品名称`的传递函数依赖。

不符合第三范式要求的表存在以下问题：
1. 数据冗余:（分类、分类描述）对于每一个商品都会进行记录
2. 数据插入异常
3. 数据更新异常
4. 数据删除异常（删除所有饮料类商品，分类描述也就不存在了）

范式化优化：

|商品名称|价格|规格|有效期|
|-------|----|----|:-----:|
| 可乐 |3.00|250ml|2016.06|
| 苹果 |8.00|500g |  -   |

|分类|分类描述|
|-------|----|
| 酒水饮料 |碳酸饮料|
| 生鲜食品 |水果|

|分类|商品名称|
|-------|----|
| 酒水饮料 |可乐|
| 生鲜食品 |苹果|

## 表的反范式化优化
反范式化是指为了查询效率的考虑，把原本符合第三范式的表适当的增加冗余，达到优化查询效率的目的，是一种以空间换时间的优化方式。看下面的例子：

用户表

|用户ID|姓名|电话|地址|邮编|
|------|---|----|----|---|

订单表

|订单ID|用户ID|下单时间|支付类型|订单状态|
|------|------|-------|-------|-------|

订单商品表

|订单ID|商品ID|商品数量|商品价格|
|------|------|-------|-------|

商品表

|商品ID|名称|描述|过期时间|
|------|---|----|-------|

对这个表中信息查询需要关联很多的表，影响查询效率。而在这样的表结构下，也无法优化SQL查询语句。因此我们对表进行适当的反范式化：

订单表

|订单ID|用户ID|下单时间|支付类型|订单状态|订单价格|用户名|电话|地址|
|------|------|-------|-------|-------|-------|-----|----|----|

这样同样查询订单信息，则只需要查询一张表就可以了，提高了SQL效率，也简化编程的复杂度。

## 表的垂直拆分
所谓垂直拆分，就是把原来有很多列的表拆分成多个表，解决了表的宽度问题。通常垂直拆分可以按照以下原则：
1. 把不常用的字段单独存放到一个表中
2. 把大字段独立存放在一个表中
3. 把经常使用的字段放在一起

## 表的水平拆分
当表的数据比较多的时候，为了提高查询与I/O效率，可以选择将表进行水平拆分。水平拆分并没有改变表的结构，仅是将原本存放在同一个表中的数据放到了多个结构一样的表中。

常用的水平拆分方法：
1. 对id进行hash运算，如果拆分成5个表，使用mod(id, 5)取出0-4个值
2. 针对不同的hashID把数据存储到不同的表中。

水平拆分会带来两个问题：跨分区表进行数据查询，以及统计后台报表操作。
解决技巧是对于前端强调效率的操作，使用分表查询；对于后端对时效性要求较低的操作，使用总表。这样也能减小前后端的相互影响。

# 数据库系统配置优化
## 操作系统的优化
1.网络方面，修改/etc/sysctl.conf文件

```bash
# 增加tcp支持的队列数
net.ipv4.tcp_max_syn_backlog = 65535
# 减少断开连接时的资源回收
net.ipv4.tcp_max_tw_buckets = 8000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 10
```

2.打开文件数的限制
使用`ulimit -a`查看目录的各位限制。修改/etc/security/limits.conf文件，增加以下内容以修改打开文件数量的限制：

```bash
* soft nofile 65535
* hard nofile 65535
```
3.关闭iptables、selinux等防火墙软件，使用硬件防火墙代替，以降低连接性能损耗。

## MySQL配置优化
Linux中MySQL的配置文件通常位于/etc/my.cnf或/etc/mysql/my.cnf。可以使用以下命令查看MySQL配置文件顺序（后面的配置文件会覆盖前面的）：
```bash
/usr/sbin/mysqld --verbose --help | grep -A 1 'Default options'
```

### InnoDB缓冲配置
- innodb_buffer_pool_size：InnoDB缓冲池配置。如果数据库中只有InnoDB表，推荐配置为总内存的75%。

- innodb_buffer_pool_instance：MySQL 5.5后引入的参数，可以控制缓冲池个数，默认为1。增加缓冲池数量有利于增加数据库并发性。

- innodb_log_buffer_size：InnoDB日志缓冲区大小，由于日志最长每秒都会刷新，因此不用配置太大。

- innodb_flush_log_at_trx_commit：关键参数，对InnoDB的I/O效率影响很大。取值为0,1,2，默认为1。建议设置为2，但如果对数据安全性要求比较高可以使用默认值1。

- innodb_read_io_thread和innodb_write_io_thread：InnoDB读写I/O的进程数，默认为4

- innodb_file_per_table：关键参数，控制InnoDB每一个表使用独立的表空间，默认为OFF，也就是所有的表都会建立在共享表空间中。独立表空间有利于增加并发读写效率，并有利于表空间的回收。

- innodb_status_on_metadata：决定了MySQL在什么情况下刷新InnoDB表的统计信息。设置为OFF可以增加效率。

### 第三方配置工具
https://tools.percona.com/wizard

# 数据库硬件优化
## 选择CPU
应该选择更快的单核CPU还是核数更多的CPU？

1. MySQL有些工作只能用到单核CPU：Replicate、SQL查询等

2. MySQL对CPU核数的支持并不是越多越快：一般不超过32核

## Disk I/O优化
- RAID0：多个磁盘连接成一个硬盘使用，I/O最好，但安全性较低，一旦损坏数据全部丢失

- RAID1：镜像备份，至少两个磁盘，互为镜像

- RAID5：至少3块硬盘合并成1个逻辑盘，数据可以基于奇偶校验恢复

推荐使用 RAID 1+0 的结合，兼顾效率与安全性

SNA 与 NAT是否适合数据库存储？
1. 常用于高可用解决方法
2. 顺序读写效率高，但是随机读写不如人意
3. 数据库随机读写比率很高
