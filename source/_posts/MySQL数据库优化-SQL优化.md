---
title: MySQL数据库优化-SQL优化
date: 2016-06-08 09:31:16
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

# 数据库优化
## 数据库优化的目的
- 避免出现页面访问错误
 - 由于数据库链接timeout产生页面5××错误
 - 由于慢查询造成页面无法加载
 - 由于阻塞造成数据无法提交

- 增加数据库的稳定性
 - 很多数据库问题都是由于低效的查询引起的

- 优化用户体验
 - 流畅页面的访问速度
 - 良好的网站功能体验

 <!-- more -->

## 可以进行数据库优化的方面
{% asset_img optimize.png optimize %}

可以看出SQL及索引的优化是最基本也是最有效的，如何组织结构良好且有效的SQL语句对于查询性能的影响至关重要。另外，表结构是否合理也是影响数据库性能的关键因素之一。

由于数据库本身对于CPU调用核数、内部锁机制等因素，往往代价最高的硬件优化反而对于数据库性能的提升并不明显。

# SQL及索引优化
## 开启MySQL慢查日志
1. 查看mysql是否开启慢查询日志
```sql
show variables like 'slow_query_log';
```
2. 设置没有索引的记录到慢查询日志
```sql
set global log_queries_not_using_indexes=on;
```
3. 查看超过多长时间的sql进行记录到慢查询日志
```sql
show variables like 'long_query_time'
```
4. 设置慢查询日志超时时间
```sql
set global long_query_time=0; (测试用，生产环境不能使用0)
```
5. 开启慢查询日志
```sql
set global slow_query_log=on；
```

## 慢查日志的信息
```bash
执行SQL的主机信息
# User@Host: root[root] @ localhost []
SQL的执行信息
# Thread_id: 49  Schema: sakila  QC_hit: No
# Query_time: 0.000313  Lock_time: 0.000098  Rows_sent: 2  Rows_examined: 2
# Rows_affected: 0
# Full_scan: Yes  Full_join: No  Tmp_table: No  Tmp_table_on_disk: No
# Filesort: No  Filesort_on_disk: No  Merge_passes: 0  Priority_queue: No
SQL执行时间
SET timestamp=1471250443;
SQL语句的内容
select * from store limit 10;
```

## 常用的慢查日志分析工具
### mysqldumpslow
```bash
mysqldumpslow -t 3 mariadb-slow.log | more
```
输出格式：
```bash
Reading mysql slow query log from mariadb-slow.log
Count: 3  Time=0.00s (0s)  Lock=0.00s (0s)  Rows_sent=3.3 (10), Rows_examined=3.
3 (10), Rows_affected=0.0 (0), root[root]@localhost
  show variables like 'S'

Count: 1  Time=0.00s (0s)  Lock=0.00s (0s)  Rows_sent=23.0 (23), Rows_examined=2
3.0 (23), Rows_affected=0.0 (0), root[root]@localhost
  show tables

Count: 1  Time=0.00s (0s)  Lock=0.00s (0s)  Rows_sent=6.0 (6), Rows_examined=6.0
 (6), Rows_affected=0.0 (0), root[root]@localhost
  show databases
```

### pt-query-digest
输出到文件：
```bash
pt-query-digest show-log > slow_log.report
```
输出到数据库表：
```bash
pt-query-digest show.log -review \
h=127.0.0.1,D=test,p=root,P=3306,u=root,t=query_review \
--create-reviewtable \
--review-history t=hostname_show
```

## 通过慢查日志定位优化语句
1. 查询次数多且每次查询占用时间长的SQL
通常为pt-query-digest分析的前几个查询

2. IO大的SQL
注意pt-query-digest分析中的Rows examine项

3. 未命中索引的SQL
注意pt-query-digest分析中Rows examine和Rows send的对比

## explain分析SQL执行计划
以以下查询语句为例：
```sql
explain select customer_id,first_name,last_name from customer;
```
输出结果：
```bash
+------+-------------+----------+------+---------------+------+---------+------+------+-------+
| id   | select_type | table    | type | possible_keys | key  | key_len | ref  | rows | Extra |
+------+-------------+----------+------+---------------+------+---------+------+------+-------+
|    1 | SIMPLE      | customer | ALL  | NULL          | NULL | NULL    | NULL |  599 |       |
+------+-------------+----------+------+---------------+------+---------+------+------+-------+
```

explain返回参数的含义：
- select_type：select查询的类型，主要是区别普通查询和联合查询、子查询之类的复杂查询
- table：输出的行所引用的表
- type：访问类型，是较为重要的一个指标，结果值从好到坏依次是`const > eq_ref > ref > range > index > ALL`
- possible_keys：指出MySQL能使用哪个索引在该表中找到行，如果为空，表示没有相关的索引。这时要提高性能，可通过检验WHERE子句，看是否引用某些字段，或者检查字段不是适合索引。
- key：显示MySQL实际决定使用的键。如果没有索引被选择，键是NULL。
- key_len：实际的索引长度。在不损失精确性的情况下，长度越短越好。这个值可以得出一个多重主键里mysql实际使用了哪一部分。
- ref：显示索引的哪一列或常数与key一起被使用。
- rows：这个数表示mysql认为必须检查用来返回请求的数据的行数，在innodb上是不准确的。
- Extra：
如果是Only index，这意味着信息只用索引树中的信息检索出的，这比扫描整个表要快；
如果是where used，就是使用上了where限制；
如果是impossible where 表示用不着where，一般就是没查出来值；
如果此信息显示`Using filesort`或者`Using temporary`的话，表示查询需要优化。WHERE和ORDER BY的索引经常无法兼顾，如果按照WHERE来确定索引，那么在ORDER BY时，就必然会引起Using filesort，这就要看是先过滤再排序划算，还是先排序再过滤划算。

## MAX()查询优化
```sql
explain select max(payment_date) from payment;
```
输出：
```bash
+------+-------------+---------+------+---------------+------+---------+------+-------+-------+
| id   | select_type | table   | type | possible_keys | key  | key_len | ref  | rows  | Extra |
+------+-------------+---------+------+---------------+------+---------+------+-------+-------+
|    1 | SIMPLE      | payment | ALL  | NULL          | NULL | NULL    | NULL | 16086 |       |
+------+-------------+---------+------+---------------+------+---------+------+-------+-------+
```
可以看出，执行这个语句需要扫描16086行数据，并且没有用到索引。这显然不是一个高效的查询。
我们为`payment_date`建立一个索引：
```sql
create index idx_paydate on payment(payment_date);
```
现在再分析一下之前的查询语句：
```bash
+------+-------------+-------+------+---------------+------+---------+------+------+------------------------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra                        |
+------+-------------+-------+------+---------------+------+---------+------+------+------------------------------+
|    1 | SIMPLE      | NULL  | NULL | NULL          | NULL | NULL    | NULL | NULL | Select tables optimized away |
+------+-------------+-------+------+---------------+------+---------+------+------+------------------------------+
```
可以看到现在不需要扫描全表了，直接通过`覆盖索引`就可以得出结果。这样极大的优化了数据库的I/O操作，提高了查询效率。

## COUNT()查询优化
例如我们需要查询2006年和2007年分别上映的电影数量，以下有两条**错误**的SQL语句：
1. 无法分开计算2006年和2007年的电影数量。
```sql
SELECT COUNT(release_year='2006' OR release_year='2007') FROM film;
```
2. release_year不可能同时为2006和2007，存在逻辑错误。
```sql
SELECT COUNT(*) FROM film WHERE release_year='2006' AND release_year='2007';
```

正确的查询：
```sql
SELECT COUNT(release_year='2006' OR NULL) AS '2006 movies', COUNT(release_year='2007' OR NULL) AS '2007 movies' FROM film;
```
> COUNT(*)语句会包含值为NULL的记录，而COUNT(column)则不包含NULL的记录

## 子查询的优化
通常情况下，需要把子查询优化为JOIN连接查询。但是在优化时要注意关联键是否存在一对多的关系，注意重复数据（使用DISTINCT去重）。

例如：查询sandra出演的所有影片
```sql
explain SELECT title,release_year,LENGTH FROM film WHERE film_id IN(
	SELECT film_id FROM film_actor WHERE actor_id IN(
		SELECT actor_id FROM actor WHERE first_name='sandra'));
```

## GROUP BY查询优化
```sql
explain SELECT actor.first_name, actor.last_name, COUNT(*) FROM sakila.film_actor
	INNER JOIN sakila.actor USING(actor_id)
		GROUP BY film_actor.actor_id \G;
```
我们看到由于缺少WHERE条件语句，MySQL对于这条查询语句的过程解释为了两个查询，并且出现了Using temporary和Using filesort。这是非常低效的。
```bash
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: actor
         type: ALL
possible_keys: PRIMARY
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 200
        Extra: Using temporary; Using filesort
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: film_actor
         type: ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 2
          ref: sakila.actor.actor_id
         rows: 1
        Extra: Using index
```

优化后的查询语句：
```sql
explain SELECT actor.first_name, actor.last_name, c.cnt
	FROM sakila.actor INNER JOIN(
		SELECT actor_id, COUNT(*) AS cnt FROM sakila.film_actor
			GROUP BY actor_id) AS c USING(actor_id) \G;
```
优化后的查询路径：
```bash
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: actor
         type: ALL
possible_keys: PRIMARY
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 200
        Extra:
*************************** 2. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
         type: ref
possible_keys: key0
          key: key0
      key_len: 2
          ref: sakila.actor.actor_id
         rows: 27
        Extra:
*************************** 3. row ***************************
           id: 2
  select_type: DERIVED
        table: film_actor
         type: index
possible_keys: NULL
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 5462
        Extra: Using index
```

## LIMIT查询优化
LIMIT常用于分页处理，时常会伴随ORDER BY从句使用，因此大多会使用Filesorts，这样会造成大量的I/O问题。
例如：
```sql
SELECT film_id, description FROM sakila.film ORDER BY title LIMIT 50, 5;
```
可以看见这个SQL语句执行过程中使用了`ALL`扫描了全表1000行数据，而且使用了`filesort`操作：
```bash
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: film
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 1000
        Extra: Using filesort
```

> 优化步骤1：使用有索引的列或主键进行ORDER BY操作

```sql
SELECT film_id, description FROM sakila.film ORDER BY film_id LIMIT 50, 5;
```
执行计划：
```bash
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: film
         type: index
possible_keys: NULL
          key: PRIMARY
      key_len: 2
          ref: NULL
         rows: 55
        Extra:
```
可以看出扫描行数已经缩减到了55行。但是当行数增加，例如执行`LIMIT 5000,5`时，则至少扫描5005行，也就是随着翻页往后，相应速度会越来越慢。因此我们需要进行进一步的查询优化。

> 优化步骤2：记录上一次返回的主键，在下次查询时使用主键过滤
```sql
SELECT film_id, description FROM sakila.film WHERE film_id > 55 AND film_id <= 60 ORDER BY film_id LIMIT 1, 5;
```

优化之后避免了数据量大时扫描过多的记录，使得查询效率比较固定。

然而这样的方式要求主键id必须是有序的，否则可能会出现中间记录空缺从而少于5条的情况。可以通过新增一个自增有序的列并且增加索引以达到相同的效果。
```bash
           id: 1
  select_type: SIMPLE
        table: film
         type: range
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 2
          ref: NULL
         rows: 5
        Extra: Using where
```
