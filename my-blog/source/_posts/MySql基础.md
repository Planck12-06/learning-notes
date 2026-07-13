---
title: MySql基础
date: 2026-07-12 18:53:00

categories:
  - Java
  - MySql 

tags:
  - 数据库
  - SpringBoot

---

## 什么是数据库

> 数据库 (DataBase, DB)：存储数据的仓库，用于管理和存储数据。

> 数据库管理系统 (DBMS)：用于操纵和管理数据库的大型软件。

> SQL (Structured Query Language)：结构化查询语言，定义了一套操作关系型数据库的统一标准。

> 结论：程序员给数据库管理系统 (DBMS) 发送 SQL 语句，再由数据库管理系统操作数据库当中的数据。

### 关系型数据库

概念：关系型数据库 (RDBMS) 使用表格的形式来存储数据。表格由行和列组成，行代表记录，列代表字段。

*代表产品：MySQL、Oracle、PostgreSQL、SQL Server*

### 非关系型数据库

概念：非关系型数据库 (NoSQL) 是一种新的数据存储方式，它不依赖于特定的数据模型，可以存储各种类型的数据（如键值对、文档、图形等）。

*代表产品：Redis、MongoDB、ElasticSearch*

---

## SQL 语句分类

| 分类 | 全称 | 说明 |
|:---:|:---:|:---|
| DDL | Data Definition Language | 数据定义语言，用于定义数据库对象，如创建、修改、删除数据库、表、视图等 |
| DML | Data Manipulation Language | 数据操作语言，用于对数据库中的数据进行增、删、改操作 |
| DCL | Data Control Language | 数据控制语言，用于控制数据库的访问权限，如授权、回收权限等 |
| DQL | Data Query Language | 数据查询语言，用于查询数据库中的数据，如 SELECT 语句 |

---

## DDL - 数据定义语言

### 约束概述

- 约束：作用于表中字段上的规则，用于限制存储在表中的数据
- 目的：保证数据库中数据的正确性、有效性、完整性

> 注意：在同一个数据库服务器中，不能创建两个名称相同的数据库，否则将会报错。可使用 IF NOT EXISTS 来避免报错。

### 常用 DDL 操作

#### 创建 / 删除数据库

```sql
-- 创建数据库
CREATE DATABASE IF EXISTS mydb;

-- 删除数据库
DROP DATABASE IF EXISTS mydb;
```

> 说明：上述语法中的 DATABASE，也可以替换成 SCHEMA

#### 修改表结构

```sql
-- 添加字段
ALTER TABLE 表名 ADD 字段名 类型(长度) [COMMENT 注释] [约束];

-- 修改字段类型
ALTER TABLE 表名 MODIFY 字段名 新数据类型(长度);

-- 修改字段名称及类型
ALTER TABLE 表名 CHANGE 旧字段名 新字段名 类型(长度) [COMMENT 注释] [约束];

-- 删除字段
ALTER TABLE 表名 DROP COLUMN 字段名;

-- 修改表名
ALTER TABLE 表名 RENAME TO 新表名;

-- 删除表结构
DROP TABLE [IF EXISTS] 表名;
```

### 约束类型

| 约束 | 描述 | 关键字 |
|:---:|:---:|:---|
| 主键约束 | 唯一标识一条记录，不允许为空，一般是整数 | `PRIMARY KEY` |
| 非空约束 | 字段不允许为 NULL | `NOT NULL` |
| 唯一约束 | 字段的值必须唯一，允许为空 | `UNIQUE` |
| 默认约束 | 字段有默认值，不设置时使用默认值 | `DEFAULT` |
| 外键约束 | 用于限制两个表的关系，保证数据的参照完整性 | `FOREIGN KEY` |

> 主键自增 AUTO_INCREMENT：
> - 每次插入新的行记录时，数据库自动生成 id 字段（主键）的值
> - 具有 AUTO_INCREMENT 的数据列从 1 开始自增长

### 建表示例

```sql
CREATE TABLE user (
    id       INT PRIMARY KEY AUTO_INCREMENT COMMENT 'ID,唯一标识',
    username VARCHAR(20) NOT NULL UNIQUE COMMENT '用户名',
    name     VARCHAR(10) NOT NULL COMMENT '姓名',
    age      INT COMMENT '年龄',
    gender   CHAR(1) DEFAULT '男' COMMENT '性别'
) COMMENT '用户表';
```

### 数据类型

#### 数值类型

| 类型 | 大小 | 有符号范围 | 无符号范围 |
|:---:|:---:|:---:|:---:|
| TINYINT | 1 byte | (-128, 127) | (0, 255) |
| SMALLINT | 2 bytes | (-32768, 32767) | (0, 65535) |
| MEDIUMINT | 3 bytes | (-8388608, 8388607) | (0, 16777215) |
| INT / INTEGER | 4 bytes | (-2147483648, 2147483647) | (0, 4294967295) |
| BIGINT | 8 bytes | (-2^63, 2^63-1) | (0, 2^64-1) |
| FLOAT | 4 bytes | (-3.4E38, 3.4E38) | 0 和 (1.175494351 E-38, 3.402823466 E+38) |
| DOUBLE | 8 bytes | (-1.797E308, 1.797E308) | 0 和 (2.2250738585072014 E-308, 1.7976931348623157 E+308) |
| DECIMAL(M,D) | 依赖于 M 和 D 的值 | 依赖于 M 和 D 的值 | 依赖于 M 和 D 的值 |

#### 字符串类型

| 类型 | 大小 | 描述 | 用途 |
|:---:|:---:|:---:|:---|
| CHAR | 0-255 bytes | 定长字符串 | 手机号、身份证号等固定长度 |
| VARCHAR | 0-65535 bytes | 变长字符串 | 用户名、地址等变长数据 |
| TINYBLOB | 0-255 bytes | 二进制数据 | 图片、音频等小型二进制 |
| TINYTEXT | 0-255 bytes | 短文本字符串 | 短文章内容 |
| BLOB | 0-16 MB | 二进制数据 | 二进制大对象 |
| TEXT | 0-16 MB | 长文本字符串 | 文章内容 |
| MEDIUMBLOB | 0-16 MB | 二进制数据 | 中等大小二进制 |
| MEDIUMTEXT | 0-16 MB | 中等长度文本 | 中等文章 |
| LONGBLOB | 0-4 GB | 二进制数据 | 大型二进制对象 |
| LONGTEXT | 0-4 GB | 极大文本 | 超长文本内容 |

> CHAR vs VARCHAR 对比：
> - CHAR：定长字符串，指定长度多长就占用多少个字符，与字段值的实际长度无关
> - VARCHAR：变长字符串，指定长度为最大占用长度，按实际内容分配空间
> - 相对来说，CHAR 的性能会更高些（不需要计算长度）

#### 日期时间类型

| 类型 | 大小 | 范围 | 格式 | 描述 |
|:---:|:---:|:---:|:---:|:---|
| DATE | 3 | 1000-01-01 至 9999-12-31 | YYYY-MM-DD | 日期值 |
| TIME | 3 | -838:59:59 至 838:59:59 | HH:MM:SS | 时间值或持续时间 |
| YEAR | 1 | 1901 至 2155 | YYYY | 年份值 |
| DATETIME | 8 | 1000-01-01 00:00:00 至 9999-12-31 23:59:59 | YYYY-MM-DD HH:MM:SS | 混合日期和时间 |
| TIMESTAMP | 4 | 1970-01-01 00:00:01 至 2038-01-19 03:14:07 | YYYY-MM-DD HH:MM:SS | 时间戳（自动更新） |

---

## DML - 数据操作语言

### INSERT - 插入数据

```sql
-- 向指定字段添加数据
INSERT INTO 表名(字段1, 字段2, ...) VALUES(值1, 值2, ...);

-- 全部字段添加数据
INSERT INTO 表名 VALUES(值1, 值2, ...);

-- 批量添加数据（指定字段）
INSERT INTO 表名(字段名1, 字段名2) VALUES(值1, 值2), (值3, 值4);

-- 批量添加数据（全部字段）
INSERT INTO 表名 VALUES(值1, 值2, ...), (值3, 值4, ...);
```

> INSERT 注意事项：
> 1. 插入数据时，指定的字段顺序需要与值的顺序一一对应
> 2. 字符串和日期型数据应该包含在引号中
> 3. 插入的数据大小，应该在字段的规定范围内

### UPDATE - 更新数据

```sql
UPDATE 表名 SET 字段名1 = 值1, 字段名2 = 值2 WHERE 条件;
```

> UPDATE 注意事项：
> 1. 修改语句的 WHERE 条件可以有也可以没有，如果没有条件，则会修改整张表的所有数据
> 2. 在修改数据时，一般需要同时修改公共字段 update_time，将其修改为当前操作时间 NOW()

### DELETE - 删除数据

```sql
DELETE FROM 表名 WHERE 条件;
```

> DELETE 注意事项：
> 1. DELETE 语句的条件可以有也可以没有，如果没有条件，则会删除整张表的所有数据
> 2. DELETE 语句不能删除某一个字段的值（可使用 UPDATE 将该字段值置为 NULL）
> 3. 删除全部数据时会提示确认，点击 Execute 即可

---

## DQL - 数据查询语言

### 基本语法

```sql
SELECT  字段列表
FROM    表名列表
WHERE   条件列表
GROUP BY 分组字段
HAVING  分组后条件
ORDER BY 排序字段
LIMIT   起始索引, 条数;
```

> 提示：
> - * 号（通配符）代表查询所有字段，在实际开发中尽量少用（不直观、影响效率）
> - 各子句的执行顺序：FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT

### 聚合函数

> 注意：聚合函数会忽略 NULL 值，不对 NULL 进行统计

| 函数 | 功能 | 说明 |
|:---:|:---:|:---|
| `COUNT()` | 统计行数 | 按列统计时，NULL 行不会被统计 |
| `SUM()` | 求和 | 计算指定列的数值和，非数值类型结果为 0 |
| `MAX()` | 最大值 | 返回指定列的最大值 |
| `MIN()` | 最小值 | 返回指定列的最小值 |
| `AVG()` | 平均值 | 计算指定列的平均值 |

### GROUP BY - 分组查询

```sql
SELECT  字段列表
FROM    表名列表
WHERE   条件列表
GROUP BY 分组字段
HAVING  分组后条件
ORDER BY 排序字段
LIMIT   起始索引, 条数;
```

> 分组查询注意事项：
> 1. 分组之后，查询的字段一般为聚合函数和分组字段，查询其他字段无意义
> 2. 执行顺序：WHERE → 聚合函数 → HAVING
> 3. WHERE 在分组前过滤，HAVING 在分组后过滤

### ORDER BY - 排序查询

排序在日常开发中是非常常见的操作，有升序 (ASC) 和降序 (DESC) 两种方式。

```sql
SELECT  字段列表
FROM    表名
[WHERE  条件列表]
[GROUP BY 分组字段]
ORDER BY 字段1 排序方式1 [, 字段2 排序方式2 ...];
```

- ASC：升序（默认值，可省略）
- DESC：降序

### LIMIT - 分页查询

```sql
SELECT  字段列表
FROM    表名
LIMIT   起始索引, 查询记录数;
```

> 分页公式：起始索引 = (查询页码 - 1) × 每页显示记录数

> 注意事项：
> 1. 起始索引从 0 开始
> 2. 分页查询是数据库的方言，不同数据库有不同的实现（MySQL 使用 LIMIT）
> 3. 如果查询的是第一页数据，起始索引可以省略，简写为 LIMIT 条数
