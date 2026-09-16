---
title: ElasticSearch弹性搜索
cover: /img/nagisang1.png
top_img: /img/nagisang1.png
date: 2026-09-07 17:14:06
categories:
 - 中间件
tags:
  - ElasticSearch
  - 搜索
  - 倒排索引
  - DSL
---

# ElasticSearch 弹性搜索


ES 要解决的只有一个核心问题：**当数据量达到百万、千万级，且用户的搜索需求不再是简单的"包含"匹配时，如何让搜索既快又准。**

整条学习主线如下：

| 阶段 | 要解决的问题 | 对应内容 |
|---|---|---|
| 为什么用 ES | 数据库模糊查询慢且功能单一 | 正向索引 vs 倒排索引 |
| 中文分词 | 官方分词器对中文逐字切分 | IK 分词器 |
| 数据建模 | ES 里的数据长什么样 | Mapping 映射属性 |
| 写数据 | 索引库与文档怎么增删改查 | DSL CRUD + `_bulk` |
| Java 写操作 | 代码里怎么操作 ES | `RestHighLevelClient` |
| 查数据 | 怎么表达复杂搜索条件 | DSL 查询语法 |
| Java 读操作 | 代码里怎么查询与解析 | `SearchRequest` + `QueryBuilders` |
| 统计分析 | 分类、品牌等过滤项怎么来 | 数据聚合 |
| 业务落地 | 搜索功能最终怎么实现 | search-service + MQ 同步 |

---

## 一、为什么需要搜索引擎

### (一) 数据库模糊查询的两个致命问题

黑马商城最初的搜索是基于 MySQL 的 `like '%关键字%'` 实现的，数据量小时能跑，但有两个硬伤。

**1. 查询效率低 —— 模糊查询不走索引**

`like '%手机%'` 这种前后都带通配符的写法，B+ 树索引直接失效，退化为全表扫描。黑马商城商品表不到 9 万条数据，数据库搜索接口的耗时已经是 ES 的数倍。

更关键的是两者的**增长曲线**完全不同：数据库随数据量增加性能下降非常明显，而 ES 基于倒排索引，性能衰减很平缓。十万级尚且如此，到了百万、千万级差距会夸张到不可接受。

**2. 功能单一 —— 匹配条件过于苛刻**

数据库模糊查询要求字段**恰好包含**关键字。而真实场景里，用户可能输错别字、用拼音、用同义词，这些数据库统统无能为力。

> 结论：海量数据搜索或复杂搜索需求，必须使用专门的搜索引擎。

### (二) MySQL 与 ES 的分工

需要明确一点：**上 ES 不等于删 MySQL**，两者擅长的事情不同。

| 维度 | MySQL | Elasticsearch |
|---|---|---|
| 擅长 | 事务型操作，保证数据安全与一致性 | 海量数据的搜索、分析、计算 |
| 索引结构 | 正向索引（B+ 树） | 倒排索引 |
| 典型场景 | 下单、支付、库存扣减 | 商品搜索、日志分析、数据统计 |

企业中标准的做法是**两者结合**：

- 对安全性要求高的**写操作** → MySQL
- 对查询性能要求高的**搜索需求** → ES
- 两者之间通过某种机制**同步数据**，保证最终一致性

### (三) 概念对照

ES 的概念与 MySQL 基本一一对应，对照着理解会非常快：

| MySQL | Elasticsearch | 说明 |
|---|---|---|
| Table（表） | Index（索引） | 同类文档的集合 |
| Row（行） | Document（文档） | 一条数据，JSON 格式 |
| Column（列） | Field（字段） | JSON 文档中的字段 |
| Schema（表结构） | Mapping（映射） | 索引中文档的字段约束 |
| SQL | DSL | JSON 风格的请求语句，用来操作 ES |

---

## 二、倒排索引 —— ES 为什么快

### (一) 正向索引的困境

先看 MySQL 这类正向索引，以 `tb_goods` 表为例：

| id | title | price |
|---|---|---|
| 1 | 小米手机 | 3499 |
| 2 | 华为手机 | 4999 |
| 3 | 华为小米充电器 | 49 |
| 4 | 小米手环 | 49 |

`id` 上有 B+ 树索引，按 id 查极快。但 `title` 只存在于叶子节点上，**按 title 搜索只能遍历所有叶子节点**：

```sql
select * from tb_goods where title like '%手机%';
```

执行过程是：逐行取出数据 → 判断 title 是否包含"手机" → 符合则放入结果集 → 继续下一行。

> 正向索引是**根据文档找词条**，适合精确匹配；模糊匹配只能全表扫描。

### (二) 倒排索引的两个核心概念

- **文档（Document）**：用来搜索的数据，每一条就是一个文档。例如一个网页、一条商品信息
- **词条（Term）**：对文档内容或用户输入按某种算法**分词**后，得到的具备含义的词语

例如"我是中国人"可以拆出：我、是、中国人、中国、国人。

### (三) 倒排索引如何构建

只有三步：

1. 把每个文档的内容用分词算法拆分，得到一个个词条
2. 建表，每行记录**词条 + 该词条所在文档 id + 位置**等信息
3. 因为词条唯一，可以给词条建索引

于是同样的数据，倒排索引长这样：

| 词条（索引） | 文档 id |
|---|---|
| 小米 | 1，3，4 |
| 手机 | 1，2 |
| 华为 | 2，3 |
| 充电器 | 3 |
| 手环 | 4 |

搜索"华为手机"的完整流程：

1. 用户输入"华为手机"
2. 对**用户输入**分词，得到词条：华为、手机
3. 拿词条在倒排索引中查找（词条有索引，极快），得到文档 id：1、2、3
4. 拿文档 id 到正向索引中取出完整文档（id 有索引，也极快）

**两次查询都走索引，全程无需全表扫描。**

### (四) 正向 vs 倒排

| | 正向索引 | 倒排索引 |
|---|---|---|
| 方向 | 根据文档找词条 | 根据词条找文档 |
| 优点 | 可给多个字段建索引；按索引字段搜索、排序极快 | 按词条搜索、模糊搜索极快 |
| 缺点 | 非索引字段或部分词条查找只能全表扫描 | 只能给词条建索引；**无法根据字段做排序** |

> 记住倒排索引"无法根据字段排序"这个缺点 —— 这正是后面 `text` 类型字段不能排序、不能聚合的根源。

---

## 三、中文分词 —— IK 分词器

> 全文基于 ES **7.12.1**（8 以上版本的 Java API 变化极大，企业中主流仍是 8 以下）。所有 DSL 均在 Kibana 的 **DevTools** 控制台中编写，它带语法提示，是学习 DSL 的主战场。

### (一) 为什么需要 IK

ES 官方的 standard 分词器对中文是**逐字切分**的：

```json
POST /_analyze
{
  "analyzer": "standard",
  "text": "黑马程序员学习java太棒了"
}
```

结果是：黑 / 马 / 程 / 序 / 员 / 学 / 习 / java / 太 / 棒 / 了 —— 每个字一个词条，完全没法用。IK 分词器就是专门解决中文分词的。

### (二) 两种模式

| 模式 | 说明 | 示例：黑马程序员学习java太棒了 |
|---|---|---|
| `ik_smart` | 智能语义切分，粗粒度 | 黑马 / 程序员 / 学习 / java / 太棒了 |
| `ik_max_word` | 最细粒度切分 | 词条最多，尽可能穷举所有可能组合 |

### (三) 扩展与停用词典

互联网新词默认词库里没有，IK 会把它拆碎。解决办法是配置扩展词典：

```xml
<!-- IK 的 config/IKAnalyzer.cfg.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
    <comment>IK Analyzer 扩展配置</comment>
    <!-- 扩展词典：新词加进去 -->
    <entry key="ext_dict">ext.dic</entry>
    <!-- 停用词典：敏感词、无意义词 -->
    <entry key="ext_stopwords">stopword.dic</entry>
</properties>
```

然后在 `config` 目录新建 `ext.dic`，一行一个词，重启 ES 后生效。

> 分词器的作用有**两个**，别只记一个：**创建倒排索引时对文档分词**，以及**用户搜索时对输入内容分词**。两边必须用同一套规则，否则就会搜不到。

---

## 四、Mapping 映射属性

### (一) Mapping 是什么

Mapping 是对索引库中文档的**约束**，类似表结构定义。常见属性：

| 属性 | 说明 |
|---|---|
| `type` | 字段数据类型 |
| `index` | 是否创建索引，默认 `true` |
| `analyzer` | 使用哪种分词器 |
| `properties` | 该字段的子字段 |

### (二) 字段类型

| 类型 | 说明 |
|---|---|
| `text` | **可分词**的文本，参与倒排索引，如商品名称、文章正文 |
| `keyword` | **不分词**的精确值，如品牌、分类、国家、IP 地址 |
| `long` / `integer` / `short` / `byte` / `double` / `float` | 数值 |
| `boolean` | 布尔 |
| `date` | 日期 |
| `object` | 对象，可包含子字段 |

> **text 与 keyword 的选择是 Mapping 设计中最核心的决策**：
> - 需要被**分词搜索**的 → `text`
> - 需要**精确匹配 / 排序 / 聚合**的 → `keyword`
> - 只想存起来展示、完全不参与搜索的 → 加 `"index": false`，省空间也省写入开销

### (三) 黑马商城商品索引库

搜索页面需要三类字段：

- **过滤字段**：分类、品牌、价格
- **排序字段**：更新时间（默认）、销量、价格
- **展示字段**：id、图片、名称、价格、评价数、销量、是否广告

对应的 Mapping —— 注意不是把 MySQL 表原样搬过来，而是**按搜索需求裁剪**：

```json
PUT /items
{
  "mappings": {
    "properties": {
      "id":           { "type": "keyword" },
      "name":         { "type": "text", "analyzer": "ik_max_word" },
      "price":        { "type": "integer" },
      "stock":        { "type": "integer" },
      "image":        { "type": "keyword", "index": false },
      "category":     { "type": "keyword" },
      "brand":        { "type": "keyword" },
      "sold":         { "type": "integer" },
      "commentCount": { "type": "integer", "index": false },
      "isAD":         { "type": "boolean" },
      "updateTime":   { "type": "date" }
    }
  }
}
```

设计要点：

- 价格以**分为单位**存 `integer`，避免浮点精度问题
- `image`、`commentCount` 只用于展示，不参与搜索 → `index: false`
- `name` 用 `ik_max_word` 细粒度分词，提高召回率
- `category`、`brand` 要用于精确过滤和聚合 → 必须是 `keyword`

---

## 五、索引库操作

ES 是 Restful 风格，四种操作对应四个 HTTP 方法：

| 操作 | 请求 |
|---|---|
| 创建 | `PUT /索引库名` + mappings |
| 查询 | `GET /索引库名` |
| 删除 | `DELETE /索引库名` |
| 修改（加字段） | `PUT /索引库名/_mapping` |

### (一) 创建

```json
PUT /heima
{
  "mappings": {
    "properties": {
      "info":  { "type": "text", "analyzer": "ik_smart" },
      "email": { "type": "keyword", "index": false },
      "name":  {
        "properties": {
          "firstName": { "type": "keyword" },
          "lastName":  { "type": "keyword" }
        }
      }
    }
  }
}
```

### (二) 修改 —— 只能加字段

**索引库一旦创建，已有字段的 mapping 无法修改。** 原因是改了字段类型或分词器，就意味着已有数据的倒排索引全部失效、必须重建，代价太大。

允许做的只有**新增字段**（不影响已有倒排索引）：

```json
PUT /heima/_mapping
{
  "properties": {
    "age": { "type": "integer" }
  }
}
```

> 这意味着**建索引库时一定要想清楚 mapping**。真要改，只能新建索引库再迁移数据。

---

## 六、文档操作

| 操作 | 请求 |
|---|---|
| 新增 | `POST /索引库名/_doc/文档id` + JSON 文档 |
| 查询 | `GET /索引库名/_doc/文档id` |
| 删除 | `DELETE /索引库名/_doc/文档id` |
| 全量修改 | `PUT /索引库名/_doc/文档id` + JSON 文档 |
| 局部修改 | `POST /索引库名/_update/文档id` + `{"doc": {...}}` |

### (一) 修改的两种方式

**全量修改**的本质是两步：先按 id 删除，再新增一个同 id 的文档。

> 坑点：如果按 id 删除时 id **不存在**，第二步的新增依然会执行 —— 于是"修改"变成了"新增"。这也是为什么在 RestClient 中新增和全量修改用的是同一个 API。

**局部修改**只改指定字段：

```json
POST /heima/_update/1
{
  "doc": {
    "email": "ZhaoYun@itcast.cn"
  }
}
```

### (二) 批处理

批量操作用 `_bulk`，请求体是**两行一组**：第一行是操作声明（元数据），第二行是操作数据。

```json
POST /_bulk
{ "index":  { "_index": "heima", "_id": "3" } }
{ "info": "黑马程序员C++讲师", "email": "ww@itcast.cn" }
{ "delete": { "_index": "heima", "_id": "4" } }
{ "update": { "_index": "heima", "_id": "1" } }
{ "doc": { "email": "new@itcast.cn" } }
```

| 动作 | 说明 |
|---|---|
| `index` | 新增，重复执行会覆盖 |
| `create` | 新增，重复执行会报错 |
| `delete` | 删除，**没有第二行数据** |
| `update` | 更新 |

> 注意 `delete` 只占一行，其他操作都是两行。

---

## 七、Java 客户端 RestHighLevelClient

### (一) 依赖与版本覆盖

```xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
</dependency>
```

**必须覆盖版本** —— SpringBoot 默认的 ES 版本是 7.17.10，与我们要用的 7.12.1 不一致：

```xml
<properties>
    <elasticsearch.version>7.12.1</elasticsearch.version>
</properties>
```

### (二) 初始化

```java
RestHighLevelClient client = new RestHighLevelClient(
        RestClient.builder(HttpHost.create("http://192.168.150.101:9200"))
);
```

### (三) 通用套路

所有 Java API 都遵循同一套流程，掌握它就不用死记：

| 步骤 | 索引库操作 | 文档操作 |
|---|---|---|
| 1. 创建 Request | `CreateIndexRequest` / `GetIndexRequest` / `DeleteIndexRequest` | `IndexRequest` / `GetRequest` / `UpdateRequest` / `DeleteRequest` / `BulkRequest` |
| 2. 准备参数 | 创建时需要 Mapping JSON | 新增、更新、批量时需要 |
| 3. 发送请求 | `client.indices().xxx()` | `client.xxx()` |
| 4. 解析结果 | —— | 查询时需要 |

**关键区别**：索引库操作要先 `client.indices()` 拿到索引操作对象；文档操作直接由 `client` 调用。

### (四) 常用示例

创建索引库：

```java
@Test
void testCreateIndex() throws IOException {
    CreateIndexRequest request = new CreateIndexRequest("items");
    request.source(MAPPING_TEMPLATE, XContentType.JSON);
    client.indices().create(request, RequestOptions.DEFAULT);
}
```

新增文档（数据库 → 文档）：

```java
@Test
void testAddDocument() throws IOException {
    // 1. 查数据库
    Item item = itemService.getById(100002644680L);
    // 2. 转为索引库实体
    ItemDoc itemDoc = BeanUtil.copyProperties(item, ItemDoc.class);
    // 3. 序列化
    String doc = JSONUtil.toJsonStr(itemDoc);

    IndexRequest request = new IndexRequest("items").id(itemDoc.getId());
    request.source(doc, XContentType.JSON);
    client.index(request, RequestOptions.DEFAULT);
}
```

查询文档（重点是解析 `_source`）：

```java
@Test
void testGetDocumentById() throws IOException {
    GetRequest request = new GetRequest("items").id("100002644680");
    GetResponse response = client.get(request, RequestOptions.DEFAULT);
    // 文档数据在 _source 里
    String json = response.getSourceAsString();
    ItemDoc itemDoc = JSONUtil.toBean(json, ItemDoc.class);
}
```

局部更新：

```java
@Test
void testUpdateDocument() throws IOException {
    UpdateRequest request = new UpdateRequest("items", "100002644680");
    request.doc("price", 58800, "commentCount", 1);
    client.update(request, RequestOptions.DEFAULT);
}
```

### (五) 批量导入 —— 分页循环

`BulkRequest` 本身没有参数，本质是**把多个普通 CRUD 请求打包一起发**。数十万条数据不可能一次性导入，要分页循环，每批 1000 条左右：

```java
@Test
void testLoadItemDocs() throws IOException {
    int pageNo = 1, size = 1000;
    while (true) {
        // 分页查数据库
        Page<Item> page = itemService.lambdaQuery()
                .eq(Item::getStatus, 1)
                .page(new Page<>(pageNo, size));
        List<Item> items = page.getRecords();
        if (CollUtils.isEmpty(items)) {
            return;
        }
        // 组装批量请求
        BulkRequest request = new BulkRequest("items");
        for (Item item : items) {
            ItemDoc itemDoc = BeanUtil.copyProperties(item, ItemDoc.class);
            request.add(new IndexRequest()
                    .id(itemDoc.getId())
                    .source(JSONUtil.toJsonStr(itemDoc), XContentType.JSON));
        }
        client.bulk(request, RequestOptions.DEFAULT);
        pageNo++;
    }
}
```

> 批量导入还有一条路是用 Logstash：无需编码，但要学它的配置语法，且对数据的再加工能力弱。Java API 方案更灵活，可以在写入前任意加工数据。

---

## 八、DSL 查询

### (一) 查询的两大分类

理解了这个分类，整套 DSL 就有了脉络：

- **叶子查询（Leaf query clauses）**：在特定字段里查询特定值，属于简单查询，很少单独使用
- **复合查询（Compound query clauses）**：以逻辑方式组合多个叶子查询，或更改叶子查询的行为方式

查询的固定语法结构：

```json
GET /{索引库名}/_search
{
  "query": { "查询类型": { "查询条件" } }
}
```

`_search` 是固定路径。无条件查询用 `match_all`：

```json
GET /items/_search
{
  "query": { "match_all": {} }
}
```

> **默认只返回 10 条**。ES 出于安全考虑限制了默认查询页数，别以为查出来 10 条就只有 10 条数据。

### (二) 叶子查询

**1. 全文检索查询** —— 先对用户输入分词，再拿词条去倒排索引搜索

```json
// match：单字段
{ "query": { "match": { "name": "脱脂牛奶" } } }

// multi_match：多字段
{ "query": { "multi_match": { "query": "脱脂牛奶", "fields": ["name", "category"] } } }
```

**2. 精确查询（Term-level）** —— 不对输入分词，作为整体精确匹配

```json
// term
{ "query": { "term": { "brand": { "value": "华为" } } } }

// range
{ "query": { "range": { "price": { "gte": 10000, "lte": 30000 } } } }
```

范围关键字：`gte` 大于等于、`gt` 大于、`lte` 小于等于、`lt` 小于。

> **精确查询只能查 keyword、数值、日期、boolean 类型的字段。** 对 `text` 字段用 term 往往搜不到，因为它存的是分词后的词条，你输入的整个短语反而匹配不上。

**3. 地理坐标查询** —— 用于附近的酒店、商家之类的场景

- `geo_bounding_box`：按矩形范围搜索
- `geo_distance`：按中心点和半径搜索

### (三) 复合查询之 bool

bool 查询用逻辑运算组合多个子查询，支持四种运算：

| 运算 | 含义 | 是否参与算分 |
|---|---|---|
| `must` | 必须匹配，类似"与" | ✅ 参与 |
| `should` | 选择性匹配，类似"或" | ✅ 参与 |
| `must_not` | 必须不匹配，类似"非" | ❌ 不参与 |
| `filter` | 必须匹配 | ❌ 不参与 |

```json
GET /items/_search
{
  "query": {
    "bool": {
      "must":     [ { "match": { "name": "手机" } } ],
      "should":   [ { "term": { "brand": { "value": "vivo" } } } ],
      "must_not": [ { "range": { "price": { "gte": 2500 } } } ],
      "filter":   [ { "range": { "price": { "lte": 1000 } } } ]
    }
  }
}
```

> **性能要点**：与搜索关键字无关的条件（品牌过滤、分类过滤、价格范围）一律用 `filter` 或 `must_not`，不参与算分。只有用户**输入框里的关键字**才需要算分，用 `must` + `match`。

### (四) 复合查询之 function_score（竞价排名）

ES 默认按相关性算分 `_score` 降序排列，自 5.1 版本起采用 **BM25** 算法（更早是 TF-IDF）。但实际业务常有"掏钱多的排前面"的需求，这就需要人为干预算分。

`function_score` 由四部分组成：

| 组成 | 说明 |
|---|---|
| 原始查询 `query` | 基于该条件搜索文档，用 BM25 打分，得到**原始算分** |
| 过滤条件 `filter` | 只有符合该条件的文档才会被重新算分 |
| 算分函数 `functions` | 决定函数算分的算法 |
| 运算模式 `boost_mode` | 原始算分与函数算分如何合成最终结果 |

算分函数有四种：

- `weight`：常量
- `field_value_factor`：以文档中某个字段值作为结果
- `random_score`：随机数
- `script_score`：自定义脚本

运算模式：`multiply`（相乘）、`replace`（替换）、`sum`、`avg`、`max`、`min`。

**示例**：让 IPhone 品牌的手机算分乘以 10

```json
GET /items/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "name": "手机" } },
      "functions": [
        {
          "filter": { "term": { "brand": "Iphone" } },
          "weight": 10
        }
      ],
      "boost_mode": "multiply"
    }
  }
}
```

执行流程：原始条件查询并算分 → 按 filter 过滤 → 符合条件的按算分函数计算 → 原始算分与函数算分按 boost_mode 合并。

> **坑点**：`boost_mode` 的值拼写是 `multiply`（相乘），不是 `multipy`，写错会直接报错。

### (五) 排序

ES 默认按 `_score` 排序，也支持自定义排序：

```json
{
  "query": { "match_all": {} },
  "sort": [ { "price": { "order": "desc" } } ]
}
```

> **能参与排序的字段类型只有：keyword、数值、地理坐标、日期。** 分词字段（`text`）无法排序 —— 这正是前面倒排索引那个"无法根据字段排序"缺点的直接体现。

### (六) 分页

**基础分页**：

```json
{
  "query": { "match_all": {} },
  "from": 0,
  "size": 10,
  "sort": [ { "price": { "order": "desc" } } ]
}
```

`from` 从第几条开始，`size` 取几条，类似 MySQL 的 `limit ?, ?`。

**深度分页问题**：

ES 数据分片存储。假设索引有 10 万条数据分布在 4 个分片，查第 99 页（`from=990, size=10`），实际执行是：

1. 每个分片各自取出前 **1000** 条
2. 汇总到协调节点重新排序
3. 截取第 990~1000 条

分页数越深，每个分片要返回的数据越多，汇总排序的内存和 CPU 压力越大。因此 **ES 直接禁止 `from + size` 超过 10000**。

| 方案 | 原理 | 状态 |
|---|---|---|
| `search_after` | 分页时需排序，从上一页的排序值开始查下一页 | 官方推荐 |
| `scroll` | 把排序后的文档 id 形成快照，基于快照分页 | 官方已不推荐 |

> 看看百度最多 77 页、京东最多 100 页就知道：**限制分页深度**才是通用做法，没必要真去实现深度分页。

### (七) 高亮

高亮的实现思路：服务端搜索时给关键字加上 HTML 标签，前端提前给该标签写好 CSS 样式。

```json
{
  "query": { "match": { "name": "脱脂牛奶" } },
  "highlight": {
    "fields": {
      "name": {
        "pre_tags": "<em>",
        "post_tags": "</em>"
      }
    }
  }
}
```

三个注意点：

1. 必须有查询条件，且是**全文检索类**查询（如 match）
2. 参与高亮的字段必须是 `text` 类型
3. **默认要求高亮字段与搜索字段一致**，不一致要加 `required_field_match=false`

### (八) 完整 DSL 结构

一个完整的查询 JSON 就是这几块的组合：

| 属性 | 作用 |
|---|---|
| `query` | 查询条件 |
| `from` + `size` | 分页 |
| `sort` | 排序 |
| `highlight` | 高亮 |
| `aggs` | 聚合，与 `query` 同级 |

---

## 九、RestClient 查询

### (一) 四步走

```java
// 1. 创建 SearchRequest
SearchRequest request = new SearchRequest("items");
// 2. 构建 DSL
request.source().query(QueryBuilders.matchAllQuery());
// 3. 发送请求
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
// 4. 解析响应
handleResponse(response);
```

核心是两个 API：

- `request.source()`：对应整个请求 JSON，包含 query、sort、from、size、highlight
- `QueryBuilders`：构建各种叶子查询、复合查询

### (二) 常用查询条件

```java
// match
request.source().query(QueryBuilders.matchQuery("name", "脱脂牛奶"));

// multi_match
request.source().query(QueryBuilders.multiMatchQuery("脱脂牛奶", "name", "category"));

// range
request.source().query(QueryBuilders.rangeQuery("price").gte(10000).lte(30000));

// term
request.source().query(QueryBuilders.termQuery("brand", "华为"));

// bool
BoolQueryBuilder bool = QueryBuilders.boolQuery();
bool.must(QueryBuilders.matchQuery("name", "脱脂牛奶"));
bool.filter(QueryBuilders.termQuery("brand", "德亚"));
bool.filter(QueryBuilders.rangeQuery("price").lte(30000));
request.source().query(bool);
```

### (三) 解析响应

响应 JSON 的结构是固定的，解析就是**从外到内逐层剥**：

```
hits                              → response.getHits()
 ├── total.value                  → getTotalHits().value
 ├── max_score                    → getMaxScore()
 └── hits[]                       → getHits()
      ├── _source                 → hit.getSourceAsString()
      └── highlight               → hit.getHighlightFields()
```

```java
private void handleResponse(SearchResponse response) {
    SearchHits searchHits = response.getHits();
    long total = searchHits.getTotalHits().value;
    for (SearchHit hit : searchHits.getHits()) {
        String source = hit.getSourceAsString();
        ItemDoc item = JSONUtil.toBean(source, ItemDoc.class);

        // 高亮结果需要单独解析
        Map<String, HighlightField> hfs = hit.getHighlightFields();
        if (CollUtils.isNotEmpty(hfs)) {
            HighlightField hf = hfs.get("name");
            if (hf != null) {
                // 取第一个高亮片段，覆盖原始字段值
                item.setName(hf.getFragments()[0].string());
            }
        }
        System.out.println(item);
    }
}
```

> **高亮结果不在 `_source` 里**，必须从 `getHighlightFields()` 单独取，再手动覆盖原字段值。这是最容易漏的一步。

### (四) 排序与分页

```java
request.source().query(QueryBuilders.matchQuery("name", "脱脂牛奶"));
request.source().sort("price", SortOrder.ASC);
request.source().from((pageNo - 1) * pageSize).size(pageSize);
```

### (五) 高亮条件

```java
request.source().highlighter(
        SearchSourceBuilder.highlight()
                .field("name")
                .preTags("<em>")
                .postTags("</em>")
);
```

---

## 十、数据聚合

聚合用于对数据做统计分析，例如"哪些品牌最受欢迎""各品牌手机的平均价格"。它比 SQL 的 group by 更方便，且速度快得多。

三类聚合：

| 类型 | 说明 | 常见实现 |
|---|---|---|
| **桶（Bucket）** | 对文档做分组 | `terms` 按字段值分组、`date_histogram` 按日期阶梯分组 |
| **度量（Metric）** | 计算统计值 | `avg`、`max`、`min`、`stats` |
| **管道（Pipeline）** | 基于其它聚合结果再做运算 | —— |

> **参与聚合的字段必须是 keyword、日期、数值、布尔类型** —— 和排序的限制一样，`text` 不行。

### (一) Bucket 聚合

统计所有商品有哪些分类：

```json
GET /items/_search
{
  "size": 0,
  "aggs": {
    "category_agg": {
      "terms": {
        "field": "category",
        "size": 20
      }
    }
  }
}
```

- `size: 0`：不返回文档，只要聚合结果
- `aggs`：定义聚合，与 `query` **同级**
- `category_agg`：聚合名称，自定义但不能重复
- `terms`：聚合类型；`field` 聚合字段；`size` 返回桶的最大数量

### (二) 带条件聚合

真实场景下聚合必须**限定在搜索结果范围内**。此时 `query` 的作用就是限定聚合的文档范围：

```json
GET /items/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "category": "手机" } },
        { "range": { "price": { "gte": 300000 } } }
      ]
    }
  },
  "size": 0,
  "aggs": {
    "brand_agg": {
      "terms": { "field": "brand", "size": 20 }
    }
  }
}
```

### (三) Metric 聚合（子聚合）

在桶内再做统计，就是**子聚合** —— 在 `aggs` 里再嵌一层 `aggs`：

```json
{
  "size": 0,
  "aggs": {
    "brand_agg": {
      "terms": { "field": "brand", "size": 20 },
      "aggs": {
        "stats_meric": {
          "stats": { "field": "price" }
        }
      }
    }
  }
}
```

`stats` 会一次性给出 min、max、avg、sum。子聚合会对**每个桶分别统计**，所以每个品牌都会得到自己的价格统计。

- **聚合三要素**：聚合名称、聚合类型、聚合字段
- **可配置属性**：`size`（结果数量）、`order`（排序方式）、`field`（聚合字段）

### (四) RestClient 实现聚合

聚合条件用 `AggregationBuilders` 构造，结果与文档同级，需单独解析：

```java
@Test
void testAgg() throws IOException {
    SearchRequest request = new SearchRequest("items");
    // 查询条件
    BoolQueryBuilder bool = QueryBuilders.boolQuery()
            .filter(QueryBuilders.termQuery("category", "手机"))
            .filter(QueryBuilders.rangeQuery("price").gte(300000));
    request.source().query(bool).size(0);
    // 聚合条件
    request.source().aggregation(
            AggregationBuilders.terms("brand_agg").field("brand").size(5)
    );

    SearchResponse response = client.search(request, RequestOptions.DEFAULT);

    // 解析聚合结果
    Aggregations aggregations = response.getAggregations();
    Terms brandTerms = aggregations.get("brand_agg");
    for (Terms.Bucket bucket : brandTerms.getBuckets()) {
        String brand = bucket.getKeyAsString();
        long count = bucket.getDocCount();
        System.out.println(brand + " : " + count);
    }
}
```

---

## 十一、黑马商城搜索实战

这是课程 day08、day09 的作业，也是把上面所有知识串起来的地方。

### (一) 服务拆分

搜索业务并发压力高，与商品服务耦合在一起不利于后期优化。做法是拆出独立的 `search-service`，把搜索相关功能（数据导入、搜索接口、过滤项聚合）全部抽过去。拆分方式参见 {% post_link 微服务 微服务 %}。

### (二) 数据同步

商品服务对商品做增删改时，索引库数据也要同步更新。用 **MQ 异步通知**实现解耦：

- 商品服务完成数据库操作后，发送消息
- 搜索服务监听消息，更新 ES

这样商品服务完全不需要感知 ES 的存在。MQ 相关内容参见 {% post_link 消息队列 消息队列 %}。

### (三) 实现搜索接口

前端搜索页发起的请求：

```
GET /search/list
```

| 参数 | 说明 |
|---|---|
| `key` | 搜索关键字 |
| `pageNo` / `pageSize` | 分页 |
| `sortBy` / `isAsc` | 排序字段与方向 |
| `category` / `brand` | 过滤 |
| `minPrice` / `maxPrice` | 价格区间 |

实现要点：

- `key` 用 `match` 走 `must`，参与算分
- 品牌、分类、价格用 `filter`，不参与算分
- 没有关键字时用 `match_all`

### (四) 过滤条件聚合

搜索页左侧的过滤项不应该写死，而要跟着搜索结果变化 —— 搜"电视"时，分类里就不该出现"手机"。

思路：用**带条件的聚合**，聚合条件与搜索条件完全一致，从而统计出搜索结果中实际存在的分类和品牌。

```
POST /search/filters
```

参数与搜索接口一致，返回值形如：

```json
{
  "category": ["手机", "曲面电视", "拉杆箱"],
  "brand": ["希捷", "小米", "华为", "Apple"]
}
```

### (五) 竞价排名

商品表中有 `isAD` 字段标记广告商品。用 `function_score` 让 `isAD = true` 的商品排到最前面：

- 过滤条件：`isAD = true`
- 算分函数：`weight`，给一个足够大的值
- 运算模式：`multiply`

---

## 十二、总结

### (一) 技术全景

| 环节 | 技术点 |
|---|---|
| 为什么用 ES | 正向索引 vs 倒排索引、MySQL 与 ES 分工 |
| 中文分词 | IK 分词器（ik_smart / ik_max_word）、扩展词典 |
| 数据建模 | Mapping（text / keyword / index / analyzer） |
| 写数据 | 索引库 CRUD、文档 CRUD、`_bulk` 批处理 |
| Java 写操作 | `RestHighLevelClient` + `XxxRequest` |
| 查数据 | DSL：叶子查询、bool、function_score、排序、分页、高亮 |
| Java 读操作 | `SearchRequest` + `QueryBuilders` + 逐层解析 |
| 统计分析 | 桶聚合、度量聚合、子聚合、带条件聚合 |
| 业务落地 | search-service 拆分、MQ 数据同步、搜索接口、过滤聚合、竞价排名 |

### (二) Java API 统一套路

无论是索引库还是文档，无论是写还是读，本质都是：

```
创建 XxxRequest → 准备 DSL 参数 → client.xxx() 发送 → 解析响应
```

记住这张映射表就够了：

| 想做的事 | Request | 发送方法 |
|---|---|---|
| 创建索引库 | `CreateIndexRequest` | `client.indices().create()` |
| 删除 / 判断索引库 | `DeleteIndexRequest` / `GetIndexRequest` | `client.indices().delete()` / `exists()` |
| 新增 / 全量改文档 | `IndexRequest` | `client.index()` |
| 查文档 | `GetRequest` | `client.get()` |
| 局部改文档 | `UpdateRequest` | `client.update()` |
| 删文档 | `DeleteRequest` | `client.delete()` |
| 批量操作 | `BulkRequest` | `client.bulk()` |
| 搜索 | `SearchRequest` | `client.search()` |

### (三) 关键避坑清单

1. **版本要对齐**：SpringBoot 默认 ES 版本是 7.17.10，必须在 `<properties>` 里覆盖成实际使用的版本，否则客户端与服务端版本不一致会各种报错。
2. **mapping 建了就不能改**：只能加字段。改字段类型或分词器必须新建索引库再迁移数据。
3. **text 与 keyword 别选错**：要分词搜索用 text，要精确匹配 / 排序 / 聚合用 keyword。选错了改起来代价极大。
4. **`match_all` 默认只返回 10 条**：别误以为数据只有 10 条。
5. **text 字段不能排序、不能聚合**：这是倒排索引的固有局限。
6. **过滤条件用 filter 而不是 must**：filter 不参与算分，性能更好。
7. **`from + size` 不能超 10000**：ES 直接禁止，深度分页用 `search_after`。
8. **高亮结果不在 `_source` 里**：要从 `getHighlightFields()` 单独取，且默认要求高亮字段 = 搜索字段。
9. **`boost_mode` 拼写是 `multiply`**：不是 `multipy`。
10. **全量修改时 id 不存在会变成新增**：id 是否存在决定了是 update 还是 create。
11. **`_bulk` 的 delete 只有一行**：其他操作是"元数据行 + 数据行"两行。
12. **分词器的两个作用别只记一个**：建索引时对文档分词、搜索时对输入分词，两边规则必须一致。

