---
title: Redis 入门
date: 2026-08-04 21:23:12
categories:
  - Web后端
tags:
  - redis
  - 缓存
  - 数据库
---

# Redis 概述

Redis（Remote Dictionary Server，远程词典服务器）是一个开源的 key-value 数据库，可以用作数据库、缓存和消息中间件。

## 常见 NoSQL 数据库

| 类型 | 代表数据库 |
|:---|:---|
| 键值类型 | Redis |
| 文档类型 | MongoDB |
| 列类型 | HBase |
| 图类型 | Neo4j |

## SQL 与 NoSQL 对比

<table>
<thead>
<tr>
<th>对比项</th>
<th>SQL</th>
<th>NoSQL</th>
</tr>
</thead>
<tbody>
<tr><td>数据结构</td><td>结构化</td><td>非结构化</td></tr>
<tr><td>数据关联</td><td>关联的</td><td>无关联的</td></tr>
<tr><td>查询方式</td><td>SQL 查询</td><td>非 SQL 查询</td></tr>
<tr><td>事务特性</td><td>ACID</td><td>BASE</td></tr>
<tr><td>存储方式</td><td>磁盘</td><td>内存</td></tr>
<tr><td>扩展性</td><td>垂直</td><td>水平</td></tr>
<tr><td>使用场景</td><td>数据结构固定，对数据安全性、一致性要求高</td><td>数据结构不固定，对性能要求高</td></tr>
</tbody>
</table>

## Redis 特点

| 特点 | 说明 |
|:---|:---|
| 数据结构丰富 | 键值型，value 支持多种不同数据结构 |
| 单线程原子性 | 每个命令具备原子性 |
| 低延迟速度快 | 基于内存、IO 多路复用、良好的编码 |
| 持久化 | 支持数据持久化 |
| 高可用 | 支持主从集群、分片集群 |
| 多语言客户端 | 支持 Java、Python、Go 等多种语言 |

---

# Redis 常见数据类型

| 数据类型 | 英文名 | 说明 |
|:---|:---|:---|
| String | 字符串 | 可存储字符串、整数或浮点数 |
| List | 列表 | 可存储一个有序列表，类似于数组 |
| Set | 集合 | 元素唯一无序的集合 |
| Hash | 哈希表 | 可存储一个哈希表，类似于对象 |
| ZSet / Sorted Set | 有序集合 | 元素唯一且可以排序的集合 |

---

# Redis 常见命令

## String 类型命令

| 命令 | 说明 |
|:---|:---|
| `SET key value` | 设置指定 key 的值 |
| `GET key` | 获取指定 key 的值 |
| `SETEX key seconds value` | 设置指定 key 的值，并在指定秒数后过期 |
| `SETNX key value` | 只有在 key 不存在时设置 key 的值 |

## Hash 类型命令

| 命令 | 说明 |
|:---|:---|
| `HSET key field value` | 设置指定 key 的指定字段的值 |
| `HGET key field` | 获取指定 key 的指定字段的值 |
| `HDEL key field` | 删除指定 key 的指定字段 |
| `HKEYS key` | 获取指定 key 的所有字段 |
| `HVALS key` | 获取指定 key 的所有值 |

## List 类型命令

| 命令 | 说明 |
|:---|:---|
| `LPUSH key value` | 在指定 key 的头部插入一个值 |
| `RPUSH key value` | 在指定 key 的尾部插入一个值 |
| `LRANGE key start end` | 获取指定 key 指定范围的值 |
| `LPOP key` | 从指定 key 的头部删除一个值 |
| `RPOP key` | 从指定 key 的尾部删除一个值 |
| `LLEN key` | 获取指定 key 的长度 |

## Set 类型命令

| 命令 | 说明 |
|:---|:---|
| `SADD key member1 [member2 ...]` | 向集合添加一个或多个成员 |
| `SMEMBERS key` | 返回集合中的所有成员 |
| `SCARD key` | 获取集合的成员数 |
| `SINTER key1 [key2 ...]` | 返回多个集合的交集 |
| `SUNION key1 [key2 ...]` | 返回多个集合的并集 |
| `SREM key member1 [member2 ...]` | 从集合中删除一个或多个成员 |
| `SISMEMBER key member` | 判断指定成员是否存在于集合中 |

## ZSet 类型命令

| 命令 | 说明 |
|:---|:---|
| `ZADD key score1 member1 [score2 member2 ...]` | 向有序集合添加一个或多个成员 |
| `ZRANGE key start end [WITHSCORES]` | 返回有序集合中指定范围的成员 |
| `ZREVRANGE key start end [WITHSCORES]` | 返回有序集合中指定范围的成员，从大到小排序 |
| `ZRANK key member` | 返回有序集合中指定成员的排名 |
| `ZREVRANK key member` | 返回有序集合中指定成员的排名，从大到小排序 |
| `ZINCRBY key increment member` | 增加有序集合中指定成员的分数 |
| `ZCARD key` | 返回有序集合的成员数 |
| `ZSCORE key member` | 返回有序集合中指定成员的分数 |
| `ZREM key member1 [member2 ...]` | 从有序集合中删除一个或多个成员 |

## 通用命令

| 命令 | 说明 |
|:---|:---|
| `KEYS pattern` | 查找所有符合给定模式的 key |
| `EXISTS key` | 检查指定 key 是否存在 |
| `TYPE key` | 返回 key 所存储的值的类型 |
| `DEL key` | key 存在时删除 key |

---

# Redis 的 Java 客户端

## 常见客户端对比

<table>
<thead>
<tr>
<th>客户端</th>
<th>特点</th>
</tr>
</thead>
<tbody>
<tr><td>Jedis</td><td>以命令作为方法名称，线程不安全，多线程环境下需要基于连接池来使用</td></tr>
<tr><td>Lettuce</td><td>基于 Netty 实现，支持同步、异步和响应式编程方式，线程安全，支持 Redis 哨兵模式、集群模式和管道模式</td></tr>
<tr><td>Redisson</td><td>基于 Redis 实现的分布式、可伸缩 Java 数据结构集合，包含 Map、Queue、Lock、Semaphore、AtomicLong 等强大功能</td></tr>
</tbody>
</table>

## Spring Data Redis

Spring Data Redis 是 Spring 对 Redis 操作的封装，主要特点：

- 提供了对不同 Redis 客户端的整合（Lettuce 和 Jedis）
- 支持 Redis 哨兵和 Redis 集群
- 支持 Redis 的发布订阅模式
- 支持 Redis 的管道模式
- 支持基于 Lettuce 的响应式编程
- 支持基于 JDK、JSON、字符串、Spring 对象的数据序列化及反序列化
- 支持基于 Redis 的 JDK Collection 实现

---

# Jedis 使用

## 使用步骤

1. 引入 Maven 依赖
2. 建立连接
3. 使用 Jedis 操作 Redis
4. 关闭连接

### 建立连接

```java
public class JedisTest {

    private Jedis jedis;

    @BeforeEach
    public void setUp() {
        jedis = new Jedis("192.168.8.130", 6379);
        jedis.auth("123321");
    }
}
```

### 关闭连接

```java
@AfterEach
public void tearDown() {
    if (jedis != null) {
        jedis.close();
    }
}
```

### 使用 JedisPool 解决线程安全问题

```java
public class JedisConnectionFactory {

    public static final JedisPool jedisPool;

    static {
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxIdle(8);
        poolConfig.setMaxTotal(8);
        poolConfig.setMinIdle(0);
        poolConfig.setMaxWait(Duration.ofMillis(1000));

        jedisPool = new JedisPool(poolConfig, "192.168.8.130", 6379, 1000, "123321", 0, "JedisPoolConnection");
    }

    // 提供静态方法来获取 Jedis 对象
    public static Jedis getJedis() {
        return jedisPool.getResource();
    }
}
```

---

# Spring Data Redis 使用方式

1. 导入 Spring Data Redis 的 Maven 坐标
2. 配置 Redis 数据源
3. 编写配置类，创建 RedisTemplate 对象
4. 通过 RedisTemplate 对象操作 Redis

## SpringDataRedis API 说明

SpringDataRedis 中提供了 RedisTemplate 工具类，其中封装了各种对 Redis 的操作。并且将不同数据类型的操作 API 封装到了不同的类型中：

<table>
<thead>
<tr>
<th>API</th>
<th>返回值类型</th>
<th>说明</th>
</tr>
</thead>
<tbody>
<tr><td><code>redisTemplate.opsForValue()</code></td><td>ValueOperations</td><td>操作 String 类型数据</td></tr>
<tr><td><code>redisTemplate.opsForHash()</code></td><td>HashOperations</td><td>操作 Hash 类型数据</td></tr>
<tr><td><code>redisTemplate.opsForList()</code></td><td>ListOperations</td><td>操作 List 类型数据</td></tr>
<tr><td><code>redisTemplate.opsForSet()</code></td><td>SetOperations</td><td>操作 Set 类型数据</td></tr>
<tr><td><code>redisTemplate.opsForZSet()</code></td><td>ZSetOperations</td><td>操作 SortedSet 类型数据</td></tr>
<tr><td><code>redisTemplate</code></td><td>-</td><td>通用的命令</td></tr>
</tbody>
</table>

## 通过 Java 代码操作 Redis

```java
@SpringBootTest
public class RedisTest {

    @Autowired
    private RedisTemplate redisTemplate;

    @Test
    public void testString() {
        ValueOperations valueOperations = redisTemplate.opsForValue();
        // SET key value
        valueOperations.set("name", "liSi");
        // SETEX key seconds value
        valueOperations.set("validate", "523621", 1, TimeUnit.MINUTES);
        // SETNX key value
        valueOperations.setIfAbsent("name", "zhangSan");
        // GET key
        System.out.println(valueOperations.get("name"));
    }

    @Test
    public void testHash() {
        HashOperations hashOperations = redisTemplate.opsForHash();
        // HSET key field value
        hashOperations.put("user", "name", "zhangSan");
        hashOperations.put("user", "age", "18");

        // HGET key field
        System.out.println(hashOperations.get("user", "name").toString());
        // HKEYS key
        hashOperations.keys("user").forEach(System.out::println);
        // HVALS key
        hashOperations.values("user").forEach(System.out::println);
    }

    @Test
    public void testList() {
        ListOperations listOperations = redisTemplate.opsForList();
        // LPUSH key value: 在指定 key 的头部插入一个值
        listOperations.leftPush("mylist", "a");
        // RPUSH key value: 在指定 key 的尾部插入一个值
        listOperations.rightPush("mylist", "b");
        // LRANGE key start end: 获取指定 key 的指定范围的值
        listOperations.range("mylist", 0, -1).forEach(System.out::println);
        // LPOP key: 从指定 key 的头部删除一个值
        System.out.println(listOperations.leftPop("mylist"));
        // RPOP key: 从指定 key 的尾部删除一个值
        System.out.println(listOperations.rightPop("mylist"));
        // LLEN key: 获取指定 key 的长度
        System.out.println(listOperations.size("mylist"));
    }

    @Test
    public void testSet() {
        SetOperations setOperations = redisTemplate.opsForSet();
        // SADD key member1 [member2 ...] 向集合添加一个或多个成员
        setOperations.add("myset", "a", "b", "c");
        // SMEMBERS key: 返回集合中的所有成员
        setOperations.members("myset").forEach(System.out::println);
        // SCARD key: 获取集合的成员数
        System.out.println(setOperations.size("myset"));
        // SINTER key1 [key2 ...]: 返回多个集合的交集
        setOperations.add("myset2", "b", "c", "d");
        Set<String> intersectResult = setOperations.intersect("myset", "myset2");
        System.out.println("[" + String.join(",", intersectResult) + "]");
        // SUNION key [key2 ...]: 返回多个集合的并集
        setOperations.union("myset", "myset2").forEach(System.out::println);
        // SREM key member1 [member2 ...]: 从集合中删除一个或多个成员
        setOperations.remove("myset", "a", "b");
    }

    @Test
    public void testZSet() {
        ZSetOperations zSetOperations = redisTemplate.opsForZSet();
        // ZADD key score1 member1 [score2 member2 ...]: 向有序集合添加一个或多个成员
        zSetOperations.add("myzset", "zhangSan", 80);
        zSetOperations.add("myzset", "liSi", 90);
        zSetOperations.add("myzset", "wangWu", 70);
        // ZRANGE key start end [WITHSCORES]: 返回有序集合中指定范围的成员
        zSetOperations.range("myzset", 0, -1).forEach(System.out::println);
        // ZREVRANGE key start end [WITHSCORES]: 返回有序集合中指定范围的成员，从大到小排序
        zSetOperations.reverseRange("myzset", 0, -1).forEach(System.out::println);
        // ZRANK key member: 返回有序集合中指定成员的排名
        System.out.println(zSetOperations.rank("myzset", "zhangSan"));
        // ZREVRANK key member: 返回有序集合中指定成员的排名，从大到小排序
        System.out.println(zSetOperations.reverseRank("myzset", "zhangSan"));
        // ZINCRBY key increment member: 增加有序集合中指定成员的分数
        zSetOperations.incrementScore("myzset", "zhangSan", 5);
        // ZCARD key: 返回有序集合的成员数
        System.out.println(zSetOperations.size("myzset"));
        // ZSCORE key member: 返回有序集合中指定成员的分数
        System.out.println(zSetOperations.score("myzset", "zhangSan"));
        // ZREM key member1 [member2 ...]: 从有序集合中删除一个或多个成员
        zSetOperations.remove("myzset", "zhangSan", "liSi");
    }
}
```
