---
title: Redis使用简介
date: 2017-05-12 01:00:00
tags: [redis]
categories:
- database
- NoSQL
---

# NoSQL概述
NoSQL并不是No SQL，而是Not only SQL，泛指非关系型数据库。随着Web 2.0的兴起，NoSQL越来越广泛的被用于：
- 高并发读写场景：减轻数据库在实时动态页面静态化的压力；
- 海量数据高效率存储和访问场景：在上亿条记录中的SQL查询效率不可忍受；
- 高可扩展性和高可用性的应用场景：传统关系型数据库难以横向扩展。

非关系型数据库很大程度上解决了海量的、不同类型的大数据分布式存储以及数据库节点扩展的问题。

<!-- more -->

主流NoSQL的产品有MongoDB、Redis、Cassandra、Membase等等。非关系型数据库根据存储类型的不同可以分为键值对存储、列存储、文档数据库、图形数据库四大类。

|  存储类型   | 相关产品  |  典型应用  |  数据模型   |  优点 |  缺点 |
| ------------ | -------- | -------- | ---------- | ----- | ----- |
| 键值对存储   | Redis  | 内容缓存，主要用于处理大量数据的高访问负载    | 一系列键值对   | 快速查询  | 存储数据缺少结构化  |
| 列存储数据库 | Cassandra，HBase | 分布式的文件系统  | 以列簇式存储，将同一列数据存在一起 | 查找速度快，可扩展性强，更容易进行分布式扩展 | 功能相对局限  |
| 文档性数据库 | CouchDB，MongoDB | Web应用，与key-value类似，但其value是结构化的 | 一系列键值对  | 数据结构要求不严格  | 查询性能不高，缺乏统一的查询语法 |
| 图形数据库 | InfoGrid | 社交网络，推荐系统等，专注于构建关系图谱 | 图结构 | 利用图结构相关算法 | 需要对整个图做计算才能得出结果，不容易做分布式集群方案 |

NoSQL的主要特点可以总结为：
- 易扩展
- 灵活的数据模型
- 大数据量、高性能
- 高可用

# Redis概述
Redis是由C语言开发的、开源的、高性能的、基于键值对存储的数据库，提供多种键值数据类型来适应不同场景的需求。

Redis支持的键值数据类型有：
- 字符串类型
- 列表类型、
- 有序集合类型
- 散列类型
- 集合类型等

Redis的应用场景有：
- 缓存：数据查询、新闻内容、好友列表
- 任务队列：秒杀、抢购
- 网站访问统计
- 数据过期处理：精确到毫秒
- 应用排行榜
- 分布式集群架构中的Session分离

# Redis的基本使用
首先编辑Redis配置文件`redis.conf`：
```bash
# sudo nano /etc/redis.conf
# daemonize no
daemonize yes
```
使用`redis-server`后台运行Redis服务：
```bash
$ redis-server /etc/redis.conf
```
然后使用`redis-cli`连接本地服务器：
```bash
$ redis-cli
127.0.0.1:6379> ping    # 测试连接
# PONG
127.0.0.1:6379> set name "Hello Redis"  # 存储数据
127.0.0.1:6379> get name
# "Hello Redis"
127.0.0.1:6379> keys *      # 显示所有key
# 1) "name"
127.0.0.1:6379> del name    # 删除name键值对
```

# Jedis与Jedis连接池
Jedis是Redis官方推荐的Java客户端开发包。Jedis与Jedis连接池的使用方式如下：
```java
public class JedisDemo1 {

    /**
     * 单实例测试
     */
    @Test
    public void jedisTest() {
        // 设置IP地址和端口
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        // 保存数据
        jedis.set("name", "Hello Jedis");
        // 获取数据
        String value = jedis.get("name");
        System.out.println(value);
        // 释放资源
        jedis.close();
    }

    /**
     * Jedis连接池测试
     */
    @Test
    public void jedisPoolTest() {
        JedisPoolConfig config = new JedisPoolConfig();
        // 最大连接数
        config.setMaxTotal(30);
        // 最大空闲连接数
        config.setMaxIdle(10);
        // 获取连接池对象
        JedisPool jedisPool = new JedisPool(config, "127.0.0.1", 6379);
        // 获取Jedis对象
        Jedis jedis = null;

        try {
            // 通过连接池获得连接
            jedis = jedisPool.getResource();
            jedis.set("name", "Hello Jedis");
            String value = jedis.get("name");
            System.out.println(value);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 释放Jedis连接
            if(jedis != null)
                jedis.close();
            // 释放连接池
            if(jedisPool != null)
                jedisPool.close();
        }
    }
}
```

# Redis数据结构
Redis中的数据结构包括字符串（String）、字符串列表（list）、有序字符串集合（sorted set）、哈希（hash）以及字符串集合（set）五种。其中比较常用的是字符串和哈希类型。

我们知道Redis的数据结构是key-value类型的，需要注意的是key的定义最好遵循统一的命名规范，同时也不要过长，否则占用大量内存并且降低数据库速度。

## String存储
Redis中的String类型数据存储是二进制安全的，即存入和获取的数据完全相同。其value最多可容纳数据长度是512M。
```bash
set key value     # 赋值
get key value     # 取值
getset key value      # 获取当前值后更新
del key       # 删除键值对
incr key      # 递增key的值，若不存在默认设置为0后递增
decr key      # 递减key的值
incrby key offset     # 将key对应的value递增offset
decrby key offset     # 将key对应的value递减offset
append key str    # 向当前key的value追加字符串str，返回Integer为字符串长度
```

## Hash存储
Redis中的Hash数据类型可以看作`Map<String key, String value>`，非常适合存储用户名-密码之类的信息。
```bash
hset hashtable key value  # 向hashtable中存储Map<key, value>键值对
hmset hashtable key1 value1 key2 value2  # 向hashtable中存储多对键值对
hget hashtable key    # 获取属性值
hmget hashtable key1 key2     # 同时获取多个key的值
hgetall hashtable      # 获取所有key和value
hdel hashtable key1 key2      # 删除<key, value>，返回0表示字段不存在
del hashtable         # 删除整个Hash集合
hincrby hashtable key offset     # 将hashtable中的key对应的value递增offset
hexists hashtable key     # 判断是否存在key，存在返回1，否则返回0
hlen hashtable            # 返回hashtable中的key的个数
hkeys hashtable           # 返回hashtable中的keys
hvals hashtable            # 返回hashtable中的values
```

## list存储
Redis中支持存储链表数据结构list，其元素按照插入顺序排列。如果插入的链表不存在，Redis将创建这个链表对象；反之如果链表对象中的所有元素都被删除了，Redis将同时从数据库中删除该链表对象。

当我们在链表两端操作数据时，即使对于已有百万元素的链表，也可以在常量时间内完成。但如果我们在链表中间操作数据，那么效率则比较低了。

Redis处理list对象的方式，有`ArrayList`数组方式（维护索引，查询速度快，插入删除速度慢）、`LinkedList`双向链表方式（维护指针，插入删除速度快，索引较慢）。
```bash
lpush list a b c      # 从左侧向list中添加元素，返回list长度
rpush list a b c      # 从右侧向list中添加元素，返回list长度
lrange list start end     # 查看链表中的元素
# lrange list 0 -1
# "c" "b" "a"
```
这里要注意的是，插入元素的方式可以类比成进栈操作。从左侧push的过程即为`c -> b -> a`，因此`lpush`指的是进栈方向，而数据存储的方向实际上是相反的。
```bash
lpop list     # 返回并弹出list左侧第一个元素
rpop list     # 返回并弹出list尾部第一个元素
llen list     # 返回list元素个数
lpushx list key   # 若list存在，则在左侧插入，否则不插入
rpushx list key   # 若list存在，则在右侧插入，否则不插入

# 删除list中couont个值为value的元素
# count > 0，自左向右删除count个；count < 0，自右向左删除count个
# count = 0，删除所有值为value的元素
lrem list count value

# 设置list中index位置的元素值为value，若不存在抛出异常
lset list index value
# 在元素key前插入元素值value
linsert list before key value
# 在元素key后插入元素值value
linsert list after key value
# 将list1右侧的元素弹出，插入list2中的左侧
rpoplpush list1 list2
```
Redis中的list存储通常用于应用中消息队列的维护。假设在一个消息队列场景中，生产者队列崩溃，可能会导致应用数据丢失。而使用Redis的`rpoplpush`就可以在每接收到一个消息后，将该消息存储到一个消息队列的备份中；在消息处理完成后，再将其从备份队列删除。当备份队列中的消息过期时，可以重新将其放回主消息队列中。我们通过维护这样一个守护队列，就可以保证应用的健壮性。

## Set存储
我们可以将set类型看作无序的、无重复元素的list类型数据。和list不同的是，Set集合中不允许出现重复元素。使用Set数据类型，可以在服务器端完成多个不同尺寸的list的聚合计算操作，如union、diff等。
```bash
sadd set a b c    # 向set中添加元素，返回添加元素个数3
sadd set c        # 添加无效，返回0
srem key1 key2    # 从set中删除元素
smembers set      # 查看set中的元素
sismember set key   # 判断元素key是否在set集合中，存在返回1否则返回0

sdiff set1 set2     # 对set1、set2执行差集运算，返回差集元素
sinter set1 set2    # 对set1、set2执行交集运算，返回交集元素
sunion set1 set2    # 对set1、set2执行并集运算，返回并集元素

scard set           # 返回set中的元素数量
srandmember set     # 随机返回set中的一个元素
sdiffstore diffset set1 set2        # 将set1与set2中的差集元素保存到diffset中
sinterstore interset set1 set2      # 将set1与set2中的交集元素保存到interset中
sunionstore unionset set1 set2      # 将set1与set2中的并集元素保存到unionset中
```

我们可以使用Redis Set数据类型跟踪维护唯一性的数据信息，如将每次访问的ip地址保存到Redis中，统计所有访客的ip信息。也可以用户维护数据对象之间的关联关系，如将购买同一个商品的用户ID保存到一个集合中，就可以通过交集操作获取同时购买两种商品的用户。

## Sorted Set
Sorted Set中的每一个元素都会有一个权重与之关联，Redis通过权重对元素进行排序。Redis对Sorted Set类型的数据操作的时间复杂度为$log(n)$，由于数据是有序的，因此即使对于中部数据的访问也是比较高效的。
```bash
# 向sset中添加元素key1，key2并按weight排序，返回添加元素个数
zadd sset weight1 key1 weight2 key2     
# 对于相同的key，重复添加仅更新权重
zadd sset weight3 key1      # 将key1的weight更新为weight3，返回0
# 获取ssset中的元素
zscore sset key     # 获取sset中key的权重
zcard sset          # 获取sset中元素个数
zrem sset sset key1 key2    # 从sset中删除元素key1，key2

# 获取sset范围内的元素[含权重]，默认从小到大排序
zrange sset start end [withscores]
# 获取sset范围内的元素[含权重]，从大到小排序
zrevrange sset start end [withscores]
# 按排序删除排名rank1到rank2的元素
zremrangebyrank sset rank1 rank2
# 按权重删除权重范围为score1到score2的元素
zremrangebysocre sset score1 score2
# 返回权重范围在score1到score2范围中的元素并排序[含权重][取前end-begin个]
zrangebyscore sset score1 score2 [withscores] [limit begin end]
# 返回权重范围在score1到score2范围中的元素个数
zcount sset score1 score2
```
Sorted Set通常被用于游戏用户积分排行榜、微博热点话题之类的应用场景或构建索引数据。

# Redis Keys通用操作
Redis中的`keys`可以用于获取所有的存储对象，这些keys还支持以下操作：
```bash
keys *      # 获取所有存储对象
keys my?    # 获取所有以my开头的存储对象
del key     # 删除名为key的存储对象
exists key          # 是否存在名为key的存储对象
rename key          # 将key1重命名为key2
expire key time     # 设置过期时间，单位s
ttl key             # 查看key剩余过期时间，若没有设置则返回-1
type key            # 查看key的存储类型
```

# Redis特性
## 多数据库支持
一个Redis实例支持多个数据库，类似于在一个MySQL中支持多个database。一个Redis实例最多支持16个数据库，下标从0到15，可以使用`select`命令选择需要连接的数据库。Redis默认连接数据库0。
```bash
select 1        # 使用1号数据库
move key db2    # 将key移动到db2中
```

## 事务支持
Redis通过`multi`、`exec`和`discard`关键字来支持事务处理。在Redis中所有的事务操作会被串行化，并且期间Redis将会拒绝提供其他连接操作来保证事务的原子性。但是与关系型数据库不同，Redis中一项事务操作失败了，Redis还将继续执行后续的操作。

我们可以通过`multi`开启一个事务，使用`exec`提交事务，使用`discard`回滚事务。

# Redis持久化概述
Redis的高性能来源于其将所有数据维持在内存中，因此我们需要将数据持久化到硬盘上来永久保存数据。Redis提供了`RDB`和`AOF`两种数据持久化的方式。

- RDB（默认）：按照一定时间频率将内存中的数据或者快照保存到磁盘
- AOF：通过日志记录服务器操作，启动时读取日志构建数据库
- 无持久化：配置禁用持久化，达到缓存效果
- RDB与AOF共用

## RDB持久化
RDB持久化方式的优点：
- 可将数据库保存为一个文件，便于备份
- 可方便移植
- 性能最大化，可开启小的进程处理持久化操作

缺点：不能保证数据完整性；当数据过大进程处理将会延迟几百到上千毫秒。

RDB持久化的配置：编辑redis.conf文件中的时间戳配置，可配置秒、分、时与之存储量。RDB文件默认保存在redis安装目录。

## AOF持久化
AOF持久化方式的优点：
- 同步写入频率高
- 不破坏写入日志数据
- 当数据过大，可启动修改重写机制，保证修改数据的更新
- 日志文件格式清晰，便于重建数据

缺点：效率低、文件偏大与rdb文件。

AOF持久化的配置：编辑redis.conf文件中的`appendonly`设置，将no修改为yes即可。
