---
title: "数据库基础知识"
layout: default
description: "数据分析师需要了解的数据库基础：关系模型、主键外键、索引原理、数据库范式、事务与 ACID、数据仓库 vs 数据库，建立扎实的底层认知。"
eyebrow: "SQL · 第 5 篇"
---

# 数据库基础知识

> **本篇目标**：理解数据库的核心概念和原理。你不需要成为 DBA（数据库管理员），但作为数据分析师，理解这些基础知识能让你写出更高效的 SQL，在面试中回答得更有深度。

## 什么是关系型数据库

关系型数据库（Relational Database）是目前最主流的数据存储方式。它的核心思想是：**用二维表（行和列）来组织数据，用表与表之间的"关系"来关联数据**。

你可以把关系型数据库想象成一个有多张 Sheet 的 Excel 文件：

- 每张 Sheet 是一张**表（Table）**
- 每一行是一条**记录（Row / Record）**
- 每一列是一个**字段（Column / Field）**
- Sheet 之间可以通过某些列关联起来

### 常见的关系型数据库

| 数据库 | 特点 | 常见场景 |
|--------|------|---------|
| MySQL | 开源、轻量、社区活跃 | Web 应用、中小型业务系统 |
| PostgreSQL | 开源、功能强大、标准兼容性好 | 数据分析、GIS、复杂查询 |
| SQL Server | 微软产品、与 .NET 生态集成好 | 企业级应用、BI |
| Oracle | 商业数据库、性能强、价格高 | 大型企业、金融、电信 |
| SQLite | 嵌入式、无服务器、单文件 | 移动端、小型应用、原型开发 |

> **对数据分析师来说**：你不需要精通每一种数据库的安装和运维，但需要知道它们的 SQL 语法存在细微差异（例如日期函数、分页语法、窗口函数支持程度）。

## 主键与外键

### 主键（Primary Key）

主键是一张表中**唯一标识每条记录**的列（或列组合）。主键有两个约束：

1. **唯一性**：同一张表中不能有两行的主键值相同
2. **非空性**：主键列不允许 NULL 值

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,  -- user_id 是主键
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100)
);
```

### 外键（Foreign Key）

外键是一张表中**引用另一张表主键**的列，用来建立表与表之间的关联关系。

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    order_date DATE,
    total_amount DECIMAL(10, 2),
    -- 声明外键：orders.user_id 引用 users.user_id
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

外键约束确保了**引用完整性**：你不能在 `orders` 表中插入一个不存在于 `users` 表中的 `user_id`。

### ER 图（实体关系图）

表与表之间的关系通常用 ER 图来表示。我们的电商数据模型的关系如下：

```
users (1) ←——→ (N) orders (1) ←——→ (N) order_items (N) ←——→ (1) products
```

- 一个用户可以有多个订单（一对多）
- 一个订单可以有多个订单明细（一对多）
- 一个商品可以出现在多个订单明细中（一对多）

## 数据类型

选择合适的数据类型对查询性能和数据准确性都有影响。以下是最常用的数据类型：

### 数值类型

| 类型 | 范围 | 用途 |
|------|------|------|
| INT | -21 亿 ~ 21 亿 | 主键、计数 |
| BIGINT | 更大的整数范围 | 大表主键、超大计数 |
| DECIMAL(M, D) | 精确小数 | 金额（避免浮点误差） |
| FLOAT / DOUBLE | 近似小数 | 科学计算（有精度损失） |

> **重要**：存储金额**必须使用 DECIMAL**，不要用 FLOAT。FLOAT 存在浮点精度问题，例如 `0.1 + 0.2` 在 FLOAT 下可能不等于 `0.3`。

### 字符串类型

| 类型 | 说明 | 用途 |
|------|------|------|
| CHAR(N) | 固定长度字符串 | 状态码、国家代码 |
| VARCHAR(N) | 可变长度字符串 | 用户名、邮箱、地址 |
| TEXT | 超长文本 | 文章内容、评论 |

### 日期类型

| 类型 | 格式 | 用途 |
|------|------|------|
| DATE | YYYY-MM-DD | 日期（不含时间） |
| DATETIME | YYYY-MM-DD HH:MM:SS | 日期+时间 |
| TIMESTAMP | 同 DATETIME，自动转时区 | 创建时间、更新时间 |

## 索引

索引是数据库中**加速查询**的核心机制。可以把索引想象成一本书的目录——没有目录时，你需要从第一页翻到最后一页才能找到想要的内容；有了目录，你可以直接翻到对应的页码。

### 索引的工作原理

最常见的索引结构是 **B+ 树**。它是一种平衡的树形数据结构，能让数据库在 O(log N) 的时间复杂度内找到目标行。

```sql
-- 创建索引
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_date    ON orders(order_date);

-- 组合索引（多列）
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);
```

### 什么时候需要索引

| 场景 | 是否需要索引 |
|------|:----------:|
| WHERE 经常按某列过滤 | 需要 |
| JOIN 的连接列 | 需要 |
| ORDER BY / GROUP BY 的列 | 可考虑 |
| 表数据量很小（< 1000 行） | 不需要 |
| 列的值重复率很高（如性别） | 效果不大 |

### 索引的代价

索引不是越多越好：

- **占用存储空间**：每个索引都是一份额外的数据结构
- **减慢写入速度**：每次 INSERT/UPDATE/DELETE 都需要同步更新索引
- **维护成本**：过多索引会增加数据库优化器的选择难度

> **对数据分析师的建议**：你通常不需要自己创建索引（那是 DBA 的工作），但你需要理解索引的存在，这样在写 SQL 时可以有意识地让查询"命中"索引。例如，避免在 WHERE 条件的列上使用函数：`WHERE YEAR(order_date) = 2024` 无法使用 `order_date` 上的索引，而 `WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'` 则可以。

## 数据库范式

范式（Normal Form）是数据库设计的一套规范，目标是**减少数据冗余、避免更新异常**。

### 第一范式（1NF）：列的原子性

每个字段只存储一个值，不能在一个字段里存多个值。

```
-- 违反 1NF：一个字段存了多个电话号码
| user_id | name | phones              |
|---------|------|---------------------|
| 1       | 张三 | 13800000001,13900000002 |

-- 符合 1NF：拆分为多行或多列
| user_id | name | phone       |
|---------|------|-------------|
| 1       | 张三 | 13800000001 |
| 1       | 张三 | 13900000002 |
```

### 第二范式（2NF）：消除部分依赖

在 1NF 基础上，非主键列必须**完全依赖于主键**（不能只依赖主键的一部分）。

```
-- 违反 2NF：student_name 只依赖 student_id，不依赖 course_id
| student_id | course_id | student_name | score |
|------------|-----------|-------------|-------|

-- 符合 2NF：拆分为两张表
students(student_id, student_name)
scores(student_id, course_id, score)
```

### 第三范式（3NF）：消除传递依赖

在 2NF 基础上，非主键列不能依赖其他非主键列。

```
-- 违反 3NF：city_name 依赖 city_id，而 city_id 依赖 user_id（传递依赖）
| user_id | city_id | city_name |
|---------|---------|-----------|

-- 符合 3NF：拆分
users(user_id, city_id)
cities(city_id, city_name)
```

> **实务中的权衡**：严格遵守范式会导致表被拆得很细，查询时需要大量 JOIN，影响性能。在数据仓库设计中，通常会有意地"反范式化"（冗余一些字段），用空间换查询速度。

## 事务与 ACID

事务（Transaction）是一组 SQL 操作的集合，要么全部成功，要么全部回滚。这对于保证数据一致性至关重要。

### ACID 四大特性

| 特性 | 英文 | 说明 | 例子 |
|------|------|------|------|
| 原子性 | Atomicity | 事务中的操作要么全做，要么全不做 | 转账：扣款和收款必须同时成功 |
| 一致性 | Consistency | 事务前后数据库都处于合法状态 | 转账后总金额不变 |
| 隔离性 | Isolation | 并发事务互不干扰 | 两人同时购买最后一件商品 |
| 持久性 | Durability | 事务提交后数据永久保存 | 即使断电，数据也不丢失 |

```sql
-- 事务示例：转账操作
START TRANSACTION;

UPDATE accounts SET balance = balance - 500 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 500 WHERE user_id = 2;

-- 如果两条都成功，提交
COMMIT;

-- 如果有任何错误，回滚
-- ROLLBACK;
```

> **对数据分析师的意义**：你通常不会直接操作事务，但需要理解它。例如，当你查询时发现数据不一致（某个指标突然跳变），可能是因为某个事务正在执行中还未提交，或者某个事务中途失败导致数据异常。

## 数据仓库 vs 数据库

这是面试中非常高频的概念题。

### 数据库（Database / OLTP）

- **全称**：Online Transaction Processing（联机事务处理）
- **设计目标**：支持高并发的增删改查操作
- **典型特征**：范式化设计、行存储、事务保证
- **典型产品**：MySQL、PostgreSQL、SQL Server
- **使用者**：应用开发者、后端工程师

### 数据仓库（Data Warehouse / OLAP）

- **全称**：Online Analytical Processing（联机分析处理）
- **设计目标**：支持大规模数据的分析查询
- **典型特征**：反范式化设计、列存储、追加写入为主
- **典型产品**：BigQuery、Snowflake、Redshift、Hive、ClickHouse
- **使用者**：数据分析师、数据工程师

### 核心区别对比

| 维度 | OLTP（数据库） | OLAP（数据仓库） |
|------|---------------|-----------------|
| 操作类型 | 增删改查 | 以查询为主 |
| 数据量 | GB 级 | TB ~ PB 级 |
| 查询模式 | 单条记录操作 | 大范围聚合分析 |
| 表设计 | 高度范式化 | 星型/雪花型（反范式） |
| 存储方式 | 行存储 | 列存储 |
| 并发需求 | 高并发读写 | 低并发、复杂查询 |
| 响应时间 | 毫秒级 | 秒级到分钟级 |

### 星型模型与雪花模型

在数据仓库中，最常见的表设计模式是**星型模型（Star Schema）**：

- **事实表（Fact Table）**：存储业务事件数据（如订单、点击、交易），数据量极大
- **维度表（Dimension Table）**：存储描述信息（如用户维度、时间维度、商品维度），数据量较小

```sql
-- 星型模型示例
-- 事实表：订单事实
CREATE TABLE fact_orders (
    order_id    BIGINT,
    user_key    INT,       -- 关联用户维度
    product_key INT,       -- 关联商品维度
    date_key    INT,       -- 关联时间维度
    quantity    INT,
    amount      DECIMAL(12, 2)
);

-- 维度表：时间维度
CREATE TABLE dim_date (
    date_key    INT PRIMARY KEY,
    full_date   DATE,
    year        INT,
    quarter     INT,
    month       INT,
    week        INT,
    day_of_week VARCHAR(10),
    is_weekend  BOOLEAN
);

-- 维度表：用户维度
CREATE TABLE dim_user (
    user_key    INT PRIMARY KEY,
    user_id     INT,
    username    VARCHAR(50),
    city        VARCHAR(50),
    age_group   VARCHAR(20),
    register_date DATE
);
```

> **雪花模型**是星型模型的变体——维度表进一步范式化拆分。例如把 `dim_user` 中的 `city` 拆成单独的 `dim_city` 表。雪花模型更规范，但查询需要更多 JOIN。

## ETL 流程

数据从业务系统到数据仓库的过程叫做 **ETL（Extract-Transform-Load）**：

1. **Extract（提取）**：从各个业务数据库、日志系统、API 中提取原始数据
2. **Transform（转换）**：数据清洗、格式统一、指标计算、维度关联
3. **Load（加载）**：将处理后的数据写入数据仓库

> 近年来也出现了 **ELT** 模式——先加载原始数据到数据仓库，再在仓库内部做转换（利用仓库强大的计算能力）。BigQuery、Snowflake 等云数据仓库更适合 ELT 模式。

## SQL 方言差异

不同数据库的 SQL 语法存在差异。以下是数据分析师最常遇到的几个差异点：

### 分页

```sql
-- MySQL / PostgreSQL
SELECT * FROM users LIMIT 10 OFFSET 20;

-- SQL Server
SELECT * FROM users ORDER BY user_id OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

-- Oracle (12c+)
SELECT * FROM users ORDER BY user_id OFFSET 20 ROWS FETCH FIRST 10 ROWS ONLY;
```

### 字符串拼接

```sql
-- MySQL
SELECT CONCAT(first_name, ' ', last_name) FROM users;

-- PostgreSQL / Oracle
SELECT first_name || ' ' || last_name FROM users;

-- SQL Server
SELECT first_name + ' ' + last_name FROM users;
```

### 获取当前日期

```sql
-- MySQL
SELECT CURDATE(), NOW();

-- PostgreSQL
SELECT CURRENT_DATE, CURRENT_TIMESTAMP;

-- SQL Server
SELECT GETDATE(), CAST(GETDATE() AS DATE);
```

## 本篇小结

| 概念 | 核心要点 |
|------|---------|
| 主键 | 唯一标识每条记录，不允许重复和 NULL |
| 外键 | 建立表间关联，保证引用完整性 |
| 索引 | 加速查询，但增加写入开销 |
| 范式 | 减少冗余，1NF → 2NF → 3NF 逐步规范 |
| 事务 | ACID 四大特性保证数据一致性 |
| OLTP vs OLAP | 事务处理 vs 分析处理，面试高频考点 |
| 星型模型 | 数据仓库的核心设计模式：事实表 + 维度表 |

> **SQL 模块到此完结**。掌握了基础语法、多表查询、窗口函数、面试题型和数据库基础知识，你已经具备了数据分析师所需的核心 SQL 能力。接下来建议在 LeetCode 上持续刷题保持手感，同时进入下一个模块继续学习。
