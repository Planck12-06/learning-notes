---
title: Feed流
date: 2026-08-18 16:59:36
categories:
- Feed流
tags:
- Feed流
---

# Feed 流

Feed 流是内容型产品（朋友圈、微博、点评）的核心信息流形态：系统按一定规则把"用户可能关心的内容"持续推送到一个滚动列表里。如何高效、稳定地实现这个列表，是 Feed 流产品的核心工程问题。

## 一、两种产品模式

内容怎么排序，决定了 Feed 流的"性格"：

- **Timeline（时间线）**：按发布时间倒序排列，如朋友圈。信息全面、不遗漏，实现简单，但内容噪音较多、用户获取感兴趣信息的效率较低。
- **智能排序**：按兴趣算法过滤并排序，如抖音、小红书推荐流。用户粘性高、容易沉浸，但强依赖算法精度，推送不准时反而劝退用户。

> 本文基于好友关注场景（关注 = 兴趣），采用 **Timeline 模式**。该模式下，用户看到的 Feed 流 = 所有关注的人发布的内容按时间倒序聚合，其工程实现有三种经典方案。

## 二、拉模式（读扩散）

也称 **Read Fan-out**：用户读取 Feed 时，**临时**去拉取他关注的所有人的内容，聚合后按时间排序返回。

- **写入**：极简，发布内容只需写自己的发件箱，不读就不写
- **读取**：每次都要聚合多个人的数据，**读延迟高、计算开销大**；关注很多（关注数千人）时性能急剧下降

> 适合**关注关系稀疏、用户量较小、读少写多**的场景，如早期 Twitter、封闭小社区。

## 三、推模式（写扩散）

也称 **Write Fan-out**：用户发布内容时，**主动推送**到每个粉丝的收件箱；读 Feed 时只需读自己的收件箱。

- **写入**：开销与粉丝数成正比，**大 V 一次发布要写百万份数据**，存储冗余度高
- **读取**：极佳，一次查询即可拿到自己的 Feed，时间排序天然有序

> 适合**关注关系密集、用户活跃度均匀、作者粉丝数可控**的场景，如中小型社交平台、即时聊天群的已读列表。

## 四、推拉结合

吸取两种方案的优点，**按用户身份区分**采用不同策略：

- **普通用户（粉丝少）→ 推模式**：发布时直接把 Feed 写入粉丝收件箱
- **大 V / 头部用户（粉丝多）→ 拉模式**：发布时只写自己发件箱，粉丝读 Feed 时再聚合拉取
- 读端：先读自己收件箱中的"推"部分，再按需拉取"大 V"的发件箱合并

> 读写兼顾，是**工业界主流方案**（微博、Twitter、Instagram 等亿级用户产品）。

## 五、对比

| 对比项 | 拉模式（读扩散） | 推模式（写扩散） | 推拉结合 |
|---|---|---|---|
| 写入开销 | 低 | 极高（与粉丝数成正比） | 中（仅大 V 走拉模式） |
| 读取性能 | 差（每次聚合） | 极佳（读自己收件箱即可） | 较好（合并读） |
| 存储冗余 | 低 | 高 | 中 |
| 实现复杂度 | 低 | 中 | 高 |
| 适用规模 | 小 | 中 | 大 |
| 典型应用 | 早期 Twitter、封闭社区 | 小型社交、群组应用 | 微博、Twitter、Instagram |

## 六、选型

- **项目初期、用户量小**：优先选 **拉模式**，实现简单、上线快
- **中等规模、关注关系均匀**：选 **推模式**，读体验最佳
- **海量用户、存在头部大 V**：选 **推拉结合**，平衡写入与读取

> 选型的本质是 **"写入开销 × 读取开销 × 存储成本 × 工程复杂度"** 的权衡，没有银弹，只有最适合当前业务规模与用户结构的方案。

---

## 七、实战：黑马点评中的 Feed 流实现

黑马点评的"关注的人的探店笔记"就是典型 Feed 流：**用户发布的笔记要出现在粉丝的收件箱里，按时间倒序分页展示**。由于是好友关注场景（无大 V 问题），采用 **推模式** 实现。

### 1. 整体流程

```
作者发布笔记
    │
    ├─ 保存笔记到数据库
    ├─ 查询所有粉丝（follow 表）
    └─ 逐个推送笔记 id 到粉丝收件箱（Redis ZSet）
         │
         └──────────────┐
                        ▼
                   粉丝查看 Feed 流
                        └─ 从自己收件箱滚动分页读取笔记 id，再查库组装笔记详情
```

### 2. 收件箱为什么用 ZSet 而不是 List

| 方案 | 分页方式 | 问题 |
|---|---|---|
| List（`LPUSH` + `LRANGE`） | 按**角标**分页 | 收件箱实时有新数据插入，角标随之变化，会**读取重复或遗漏** |
| **ZSet（推荐）** | 按 **score 范围 + 游标** 分页 | 天然按时间戳排序，支持滚动分页，不受新增数据影响 |

- 收件箱 key：`feed:{userId}`
- member：笔记 id
- score：发布时间戳（`System.currentTimeMillis()`）

### 3. 发布笔记时推送到粉丝收件箱

```java
@Override
public Result saveBlog(Blog blog) {
    // 1. 获取登录用户
    Long userId = UserHolder.getUser().getId();
    blog.setUserId(userId);
    // 2. 保存探店笔记
    boolean isSuccess = save(blog);
    if (!isSuccess) {
        return Result.fail("新增笔记失败!");
    }
    // 3. 查询笔记作者的所有粉丝 select * from tb_follow where follow_user_id = ?
    List<Follow> follows = followService.query().eq("follow_user_id", userId).list();
    // 4. 推送笔记id到所有粉丝的收件箱
    for (Follow follow : follows) {
        // 4.1 获取粉丝id
        Long followUserId = follow.getUserId();
        // 4.2 推送：ZADD feed:{粉丝id} 时间戳 笔记id
        String key = "feed:" + followUserId;
        stringRedisTemplate.opsForZSet().add(key, blog.getId().toString(), System.currentTimeMillis());
    }
    // 5. 返回笔记id
    return Result.ok(blog.getId());
}
```

### 4. 滚动分页（核心难点）

**为什么不能用普通分页**：Feed 流数据**实时增长**，若按角标/固定页码查询，翻页期间新笔记插入会导致角标偏移，出现**数据重复或遗漏**。

**滚动分页思想**：记录上次查询的"位置"作为**游标**，每次从游标处继续往后查，与数据总量无关。对应 ZSet 命令：

```bash
ZREVRANGEBYSCORE key max min [WITHSCORES] LIMIT offset count
```

| 参数 | 含义 |
|---|---|
| `max` | 上次查询结果中**最小的时间戳**（首页传当前时间） |
| `min` | 0 |
| `offset` | 上次查询结果中，**score 与 minTime 相同的元素个数**（跳过重复时间戳） |
| `count` | 本页条数 |

> **offset 的作用**：多条笔记可能时间戳完全相同。若直接用 `max = minTime` 再查，同分值的笔记会被**重复查出**；用 offset 跳过已查过的同分值条数，保证不重不漏。计算方式：遍历本次结果时，记录最小时间戳 `minTime` 与"分数等于 `minTime` 的元素个数"，后者即为下次请求的 offset。

### 5. 滚动分页查询实现

```java
@Override
public Result queryBlogOfFollow(Long max, Integer offset) {
    // 1. 获取当前用户
    Long userId = UserHolder.getUser().getId();
    // 2. 查询收件箱 ZREVRANGEBYSCORE key max min LIMIT offset count
    String key = "feed:" + userId;
    Set<TypedTuple<String>> typedTuples = stringRedisTemplate.opsForZSet()
            .reverseRangeByScoreWithScores(key, 0, max, offset, 2);
    // 3. 非空判断
    if (typedTuples == null || typedTuples.isEmpty()) {
        return Result.ok();
    }
    // 4. 解析数据：blogId、minTime（本次查询最小时间戳）、offset（与最小时间戳相同的个数）
    List<Long> ids = new ArrayList<>(typedTuples.size());
    long minTime = 0;
    int os = 1;
    for (TypedTuple<String> tuple : typedTuples) {
        // 4.1 获取笔记id
        ids.add(Long.valueOf(tuple.getValue()));
        // 4.2 获取score（时间戳）
        long time = tuple.getScore().longValue();
        if (time == minTime) {
            // 与当前最小时间戳相同，offset + 1
            os++;
        } else {
            // 遇到更小的时间戳，重置 minTime 与 offset
            minTime = time;
            os = 1;
        }
    }
    // 5. 根据id查询笔记，并按传入id顺序排序（保证 Feed 流时间倒序）
    String idStr = StrUtil.join(",", ids);
    List<Blog> blogs = query().in("id", ids).last("ORDER BY FIELD(id," + idStr + ")").list();
    // 6. 封装返回（前端下次请求携带 minTime 作为 max、offset）
    ScrollResult r = new ScrollResult();
    r.setList(blogs);
    r.setMinTime(minTime);
    r.setOffset(os);
    return Result.ok(r);
}
```

前端每次请求携带上次返回的 `minTime`（作为 `max`）和 `offset`，实现"滚动加载下一页"。

### 6. 小结

- **推模式落地**：发布笔记 → 查粉丝 → `ZADD` 到每个粉丝的 `feed:{粉丝id}`
- **收件箱选型**：ZSet 按时间戳排序 + 支持 score 范围查询，天然适合滚动分页
- **滚动分页**：`ZREVRANGEBYSCORE` + 游标（`max`/`offset`），解决实时数据流中普通分页的重复/遗漏问题
- 该方案的代价与推模式固有缺陷一致：粉丝多时写入开销大；若出现大 V，需演进为**推拉结合**

---

## 八、总结

| 维度 | 内容 |
|---|---|
| 产品模式 | Timeline（按时间排序，适合关注关系）/ 智能排序（按兴趣算法，适合推荐流） |
| 三种方案 | 拉模式（读扩散）/ 推模式（写扩散）/ 推拉结合 |
| 方案选型 | 小用户量 → 拉；均匀中等 → 推；海量含大 V → 推拉结合 |
| 落地要点 | 推模式用 ZSet 收件箱 + 时间戳 score + `ZREVRANGEBYSCORE` 滚动分页 |

> Feed 流的工程本质：**用空间换时间**（推模式把读开销转移到写时）或**用时间换空间**（拉模式读时聚合），推拉结合则在两者间寻找平衡。理解了这层取舍，再看微博、Twitter 的架构演进就会有"原来如此"的通透感。
