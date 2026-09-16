---
title: MongoDB基础知识
cover: /img/xi.png
top_img: /img/xi.png
date: 2026-09-16 17:35:29
categories:
 - 中间件
tags:
 - MongoDB
 - 数据库
---

# MongoDB简介

> MongoDB 是**面向文档的 NoSQL 数据库**，数据以 **BSON**（类 JSON 的二进制格式）文档存储，无需预定义表结构。

**与 MySQL 概念对照**：

| MySQL | MongoDB | 说明 |
|---|---|---|
| database | database | 数据库 |
| table | **collection** | 集合，无固定结构 |
| row | **document** | 文档 |
| column | **field** | 字段 |
| primary key | **_id** | 主键，自动生成 `ObjectId` |

**核心特点**：文档模型字段灵活、内嵌文档减少关联查询、天生支持分片水平扩展。

---

# 一、基础操作

## （一）库与集合

```javascript
show dbs                  // 查看所有数据库
use test                  // 创建/切换数据库
db                        // 查看当前数据库
db.dropDatabase()         // 删除当前数据库

show collections          // 查看集合
db.users.drop()           // 删除集合
```

> **重点**：`use` 不存在的库不会立即创建，**插入数据后**才真正落库；集合也**无需显式创建**，插入文档时自动创建。

## （二）增删改查（CRUD）

**增（Insert）**：

```javascript
db.users.insertOne({ name: "张三", age: 20, tags: ["java", "mysql"] })
db.users.insertMany([{ name: "李四", age: 22 }, { name: "王五", age: 25 }])
```

> 不指定 `_id` 时，MongoDB 自动生成全局唯一的 `ObjectId`。

**查（Find）**：

```javascript
db.users.find()                          // 查询全部
db.users.findOne({ name: "张三" })        // 查询一条

db.users.find({ age: { $gte: 20 } })      // 比较运算
db.users.find({ $or: [{ age: 20 }, { age: 25 }] })   // 逻辑运算
db.users.find({ tags: "java" })           // 数组字段包含
db.users.find({ name: /张/ })             // 模糊查询（正则）
db.users.find({ age: { $gt: 20 } }, { name: 1, age: 1, _id: 0 })  // 投影：1 显示 / 0 隐藏

db.users.find().sort({ age: 1 })          // 排序：1 升序，-1 降序
db.users.find().skip(1).limit(2)          // 分页
```

**常用查询运算符**：

| 运算符 | 含义 | 运算符 | 含义 |
|---|---|---|---|
| `$gt` / `$gte` | 大于 / 大于等于 | `$in` / `$nin` | 在 / 不在集合内 |
| `$lt` / `$lte` | 小于 / 小于等于 | `$and` / `$or` | 逻辑与 / 或 |
| `$ne` | 不等于 | `$exists` | 字段是否存在 |

**改（Update）**：

```javascript
db.users.updateOne({ name: "张三" }, { $set: { age: 21, city: "北京" } })  // 修改字段
db.users.updateMany({ age: { $lt: 20 } }, { $set: { status: "未成年" } })   // 批量修改
db.users.updateOne({ name: "张三" }, { $inc: { age: 1 } })                 // 自增
db.users.updateOne({ name: "张三" }, { $unset: { city: "" } })             // 删除字段
db.users.updateOne({ name: "张三" }, { $push: { tags: "redis" } })         // 数组追加
db.users.updateOne({ name: "张三" }, { $pull: { tags: "mysql" } })         // 数组移除
```

> **易错点**：更新**必须配合 `$set` 等更新运算符**，否则会用新文档**整体替换**原文档，其他字段全部丢失。

**删（Delete）**：

```javascript
db.users.deleteOne({ name: "张三" })     // 删除单条
db.users.deleteMany({ age: { $lt: 18 } })// 删除多条
db.users.deleteMany({})                  // 清空集合
```

---

# 二、聚合操作（Aggregation）

> **聚合**用于对文档分组、统计、计算，以**管道（pipeline）**形式执行，文档依次流经各阶段，等价于 SQL 的 `GROUP BY` + 函数。

**常用管道阶段**：

| 阶段 | 作用 | 类比 SQL | 阶段 | 作用 | 类比 SQL |
|---|---|---|---|---|---|
| `$match` | 过滤文档 | `WHERE` | `$limit`/`$skip` | 限制/跳过 | `LIMIT` |
| `$group` | 分组计算 | `GROUP BY` | `$unwind` | 拆解数组 | — |
| `$sort` | 排序 | `ORDER BY` | `$lookup` | 关联集合 | `JOIN` |
| `$project` | 指定输出字段 | `SELECT` | — | — | — |

**常用累加器**（`$group` 内）：`$sum` 求和、`$avg` 平均、`$max`/`$min` 最大最小、`$push`/`$addToSet` 收集数组。

```javascript
// 分组统计 + 排序
db.users.aggregate([
    { $group: { _id: "$age", count: { $sum: 1 } } },
    { $sort: { count: -1 } }
])

// 过滤后整体统计
db.users.aggregate([
    { $match: { age: { $gte: 20 } } },
    { $group: { _id: null, avgAge: { $avg: "$age" } } }
])

// $unwind 拆数组，统计每个 tag 出现次数
db.users.aggregate([
    { $unwind: "$tags" },
    { $group: { _id: "$tags", count: { $sum: 1 } } }
])

// $lookup 关联查询（类似 JOIN）
db.orders.aggregate([
    { $lookup: { from: "users", localField: "userId", foreignField: "_id", as: "userInfo" } }
])
```

> **重点**：`$group` 的 `_id` 是分组依据，写 `null` 表示整体统计；字段引用必须加 `$` 前缀；`$match` 应放管道最前，可提前缩小数据量。

---

# 三、索引（Index）

> **索引**给字段建立有序结构，加速查询，避免全集合扫描（COLLSCAN）。

**索引类型**：

| 类型 | 说明 |
|---|---|
| 单字段 / 复合索引 | 单字段 / 多字段组合，复合索引**遵循最左前缀** |
| 唯一索引 | 字段值不能重复 |
| 文本索引 | 支持全文检索 |
| TTL 索引 | 文档到期自动删除 |
| 地理空间索引 | 2dsphere，支持位置查询 |

```javascript
db.users.createIndex({ name: 1 })                                  // 单字段（1 升序 / -1 降序）
db.users.createIndex({ age: 1, name: -1 })                         // 复合索引
db.users.createIndex({ email: 1 }, { unique: true })               // 唯一索引
db.logs.createIndex({ createTime: 1 }, { expireAfterSeconds: 3600 })// TTL 索引（3600 秒后过期）

db.users.getIndexes()               // 查看索引
db.users.dropIndex({ name: 1 })     // 删除索引
```

**执行计划（性能分析）**：

```javascript
db.users.find({ name: "张三" }).explain("executionStats")
// stage: "COLLSCAN" → 全表扫描（慢）    stage: "IXSCAN" → 索引扫描（快）
```

> **重点**：索引**加快读、拖慢写**，不宜滥用；复合索引 `{a:1,b:1}` 支持按 `a` 或 `a,b` 查，**不支持只按 `b` 查**。

---

# 四、Java 操作 MongoDB

## （一）依赖与配置

```xml
<!-- Spring Boot 官方 starter（推荐） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/test
```

## （二）方式一：MongoTemplate（灵活，适合复杂查询）

```java
@Autowired
private MongoTemplate mongoTemplate;

// 增
mongoTemplate.save(user);      // 有 _id 则更新，无则新增
mongoTemplate.insert(user);    // 强制新增

// 查（支持动态条件、分页、排序）
Query query = new Query(Criteria.where("age").gte(20));
query.with(Sort.by(Sort.Direction.DESC, "age")).skip(0).limit(10);
List<User> list = mongoTemplate.find(query, User.class);

// 改
Update update = new Update().set("age", 21).inc("score", 1);
mongoTemplate.updateFirst(Query.query(Criteria.where("name").is("张三")), update, User.class);

// 删
mongoTemplate.remove(Query.query(Criteria.where("name").is("张三")), User.class);

// 聚合
Aggregation agg = Aggregation.newAggregation(
    Aggregation.match(Criteria.where("age").gte(20)),
    Aggregation.group("age").count().as("count"),
    Aggregation.sort(Sort.Direction.DESC, "count")
);
mongoTemplate.aggregate(agg, "users", Document.class);
```

## （三）方式二：MongoRepository（简单 CRUD）

```java
// 1. 实体类：@Document 指定集合，@Id 映射主键
@Data
@Document(collection = "users")
public class User {
    @Id
    private String id;
    private String name;
    private Integer age;
}

// 2. 接口继承 MongoRepository 即拥有基础 CRUD，方法名可自动推导查询
public interface UserRepository extends MongoRepository<User, String> {
    List<User> findByName(String name);                 // 等价 find({name: "xxx"})
    @Query("{ 'age': { $gte: ?0 } }")                   // 或用注解自定义
    List<User> findByAgeGte(Integer age);
}

// 3. 调用
userRepository.save(user);
userRepository.findById(id);
userRepository.deleteById(id);
```

| 方式 | 适用场景 |
|---|---|
| **MongoTemplate** | 复杂查询、聚合、动态条件 |
| **MongoRepository** | 简单标准 CRUD，开发效率高 |

---

# 五、高可用部署

> 单机 MongoDB 存在容量与单点故障的瓶颈，生产环境通过**副本集**与**分片集群**解决。

## （一）副本集（Replica Set）

> 一组维护相同数据的节点集群，解决**高可用 + 数据冗余**。

| 角色 | 说明 |
|---|---|
| **Primary** | 唯一可读写的节点，写操作都走它 |
| **Secondary** | 从主节点同步数据，默认只读，可分担读请求 |
| **Arbiter** | 不存数据，只参与投票选举 |

```
客户端 ──写──→ Primary ──oplog复制──→ Secondary / Secondary
```

- **oplog**：主节点记录写操作，从节点拉取重放以保持数据一致
- **故障转移**：主节点宕机后，剩余节点**自动选举**出新主（默认几秒）
- 标准架构为 **1 Primary + 2 Secondary**（三节点），保证挂一台仍能选举

```javascript
// 启动节点（--replSet 指定副本集名）后初始化
rs.initiate({
    _id: "rs0",
    members: [
        { _id: 0, host: "localhost:27017" },
        { _id: 1, host: "localhost:27018" },
        { _id: 2, host: "localhost:27019" }
    ]
})
rs.status()   // 查看状态，重点看 stateStr 是否为 PRIMARY
```

> **重点**：连接串要带 `replicaSet=rs0` **和多个节点地址**，才能在主节点切换时自动重连；写仍集中在单 Primary，**无法水平扩展写能力**。

## （二）分片集群（Sharded Cluster）

> 把数据分散到多个分片，解决单机**存储容量与写入吞吐**瓶颈，用于**海量数据**。

| 组件 | 作用 |
|---|---|
| **Shard（分片）** | 实际存储数据，**每个分片本身是一个副本集** |
| **Config Server** | 存储集群元数据（分片规则、数据分布） |
| **Mongos** | 客户端统一入口，**路由请求**到对应分片，对客户端透明 |

```
客户端 ──→ Mongos ──→ Config Server(元数据)
             │
      ┌──────┼──────┐
   Shard1  Shard2  Shard3 （均为副本集）
```

**分片键（Shard Key）**：决定数据如何分布，是分片集群的核心设计点。

| 策略 | 特点 |
|---|---|
| 范围分片 | 范围查询高效，但易**数据倾斜**（热点） |
| 哈希分片 | 分布均匀，但范围查询需**广播** |

```javascript
sh.enableSharding("test")                              // 开启数据库分片
sh.shardCollection("test.users", { userId: "hashed" }) // 按哈希分片键分片
sh.status()                                            // 查看分片状态
```

> **重点**：分片键选择至关重要——基数低、单调递增（如时间戳）的字段易造成**热点**，建议用**哈希分片或复合分片键**。

---

# 六、安全认证

> MongoDB 默认**不开启认证**，生产环境必须开启。

| 概念 | 说明 |
|---|---|
| 认证 Authentication | 验证"你是谁"（账号密码） |
| 授权 Authorization | 验证"你能做什么"（角色权限） |

**常用内置角色**：`read` 只读、`readWrite` 读写、`dbAdmin` 库管理、`userAdmin` 用户管理、`root` 超级管理员。

**开启认证步骤**：

```javascript
// 1. 先以无认证方式启动，创建管理员用户
use admin
db.createUser({ user: "admin", pwd: "123456", roles: [{ role: "root", db: "admin" }] })

// 2. 创建业务库普通用户
use test
db.createUser({ user: "appUser", pwd: "app123456", roles: [{ role: "readWrite", db: "test" }] })
```

```bash
# 3. 重启并开启认证
mongod --auth --dbpath /data/db
# 或配置文件 security.authorization: enabled
```

```bash
# 4. 连接串：authSource 表示认证库
mongodb://appUser:app123456@localhost:27017/test?authSource=test
```

**其他安全措施**：`bindIp` 限制内网访问（禁止暴露公网）、开启 TLS 传输加密、WiredTiger 存储加密、业务账号遵循**最小权限**。

---

# 小结

| 模块 | 一句话理解 |
|---|---|
| 基础操作 | 增 `insert`、删 `delete`、改 `update`（**必须带 `$set`**）、查 `find` |
| 聚合 | `aggregate` 管道：`$match` 过滤 → `$group` 分组 → `$sort` 排序 |
| 索引 | 加快读、拖慢写；复合索引遵循**最左前缀** |
| Java 操作 | 灵活用 `MongoTemplate`，简单用 `MongoRepository` |
| 副本集 | 多副本 + 自动故障转移，解决**高可用** |
| 分片集群 | Mongos 路由 + Config 元数据 + 多 Shard，解决**海量数据** |
| 安全认证 | 建用户 → 开 `--auth` → 连接带 `authSource`，遵循**最小权限** |
