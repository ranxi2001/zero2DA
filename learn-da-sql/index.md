---
title: "SQL —— 数据分析师最核心的硬技能"
layout: default
description: "SQL 模块总览：从基础语法到多表查询、窗口函数、面试高频题型、数据库基础知识，系统掌握数据分析师必备的 SQL 技能。"
eyebrow: "SQL"
---

# SQL —— 数据分析师最核心的硬技能

> **"如果数据分析师只能学一门技能，那一定是 SQL。"**
>
> 无论你面试哪家公司、做什么行业的数据分析，SQL 都是绕不开的第一关。它不只是"写查询"，更是你与数据库对话、提取洞察、验证假设的核心语言。

## 为什么 SQL 对数据分析师如此重要

在实际工作中，数据分析师 **80% 以上的时间** 都在和数据打交道：从数据仓库里提取数据、做数据清洗、跑指标报表、做 Ad-hoc 分析。而完成这些工作最常用的工具就是 SQL。

与 Python 或 Excel 不同，SQL 是直接在数据库层面操作数据的语言。它的优势在于：

- **速度快**：直接在数据库引擎上运算，处理百万甚至上亿行数据毫无压力
- **通用性强**：MySQL、PostgreSQL、BigQuery、Snowflake、Hive 等主流平台都使用 SQL
- **面试必考**：几乎所有 DA/BA 岗位的技术面试都会考 SQL，通常占面试权重的 30%-50%
- **上手快**：语法接近自然语言，入门门槛低，但深入后有窗口函数等高级话题

## 本模块包含的文章

本模块共 5 篇文章，按照从基础到进阶的顺序编排。建议按顺序学习，每篇文章都包含大量可运行的 SQL 代码示例。

| 序号 | 文章 | 核心内容 | 建议用时 |
|:---:|------|---------|:-------:|
| 01 | [SQL 基础语法](./01-sql-basics/index.html) | SELECT、WHERE、GROUP BY、HAVING、ORDER BY、LIMIT | 2-3 小时 |
| 02 | [多表查询与子查询](./02-join-subquery/index.html) | INNER JOIN、LEFT JOIN、RIGHT JOIN、FULL JOIN、自连接、子查询 | 2-3 小时 |
| 03 | [窗口函数](./03-window-functions/index.html) | ROW_NUMBER、RANK、DENSE_RANK、LAG、LEAD、SUM OVER、累计计算 | 2-3 小时 |
| 04 | [SQL 面试高频题型](./04-sql-interview/index.html) | 连续登录、Top N、留存率、同比环比等面试经典题 | 3-4 小时 |
| 05 | [数据库基础知识](./05-database-fundamentals/index.html) | 关系模型、主键外键、索引、范式、事务、数据仓库 vs 数据库 | 1-2 小时 |

## 学习建议

### 动手练习是唯一的学习方式

SQL 不是"看会"的，而是"写会"的。推荐以下免费练习平台：

- **[SQLZoo](https://sqlzoo.net/)**：交互式 SQL 练习，按知识点分类
- **[LeetCode Database](https://leetcode.com/problemset/database/)**：面试级 SQL 题库
- **[HackerRank SQL](https://www.hackerrank.com/domains/sql)**：从 Easy 到 Hard 的分级练习
- **[Mode Analytics SQL Tutorial](https://mode.com/sql-tutorial/)**：结合真实数据集的教程

### 推荐的学习路径

```
第 1 周：SQL 基础语法 + 数据库基础知识
         → 能独立写 SELECT / WHERE / GROUP BY 查询

第 2 周：多表查询与子查询
         → 能用 JOIN 关联多张表，用子查询解决复杂问题

第 3 周：窗口函数
         → 掌握排名、偏移、累计等高级分析能力

第 4 周：SQL 面试高频题型 + 刷题
         → 每天 2-3 道 LeetCode SQL 题，保持手感
```

### 示例数据说明

本模块所有文章使用统一的电商数据模型作为示例，包含以下核心表：

```sql
-- 用户表
CREATE TABLE users (
    user_id     INT PRIMARY KEY,
    username    VARCHAR(50),
    email       VARCHAR(100),
    city        VARCHAR(50),
    created_at  DATE
);

-- 订单表
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    user_id     INT,
    order_date  DATE,
    total_amount DECIMAL(10, 2),
    status      VARCHAR(20)   -- 'completed', 'cancelled', 'pending'
);

-- 商品表
CREATE TABLE products (
    product_id   INT PRIMARY KEY,
    product_name VARCHAR(100),
    category     VARCHAR(50),
    price        DECIMAL(10, 2)
);

-- 订单明细表
CREATE TABLE order_items (
    item_id     INT PRIMARY KEY,
    order_id    INT,
    product_id  INT,
    quantity    INT,
    unit_price  DECIMAL(10, 2)
);
```

> 这套数据模型覆盖了用户、订单、商品三个维度，足以演示数据分析中 90% 的 SQL 场景。后续每篇文章都会基于这些表来写查询示例。

## 准备好了吗？

如果你是零基础，建议从第一篇 [SQL 基础语法](./01-sql-basics/index.html) 开始，一步步打好基础。如果你已经有一定 SQL 基础，可以直接跳到 [窗口函数](./03-window-functions/index.html) 或 [SQL 面试高频题型](./04-sql-interview/index.html) 查漏补缺。

开始学习吧 ------ SQL 是你成为数据分析师的第一把钥匙。
