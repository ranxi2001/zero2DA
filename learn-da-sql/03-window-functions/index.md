---
title: "窗口函数"
layout: default
description: "全面掌握 SQL 窗口函数：ROW_NUMBER、RANK、DENSE_RANK、LAG、LEAD、SUM OVER、累计计算、滑动平均，解锁高级数据分析能力。"
eyebrow: "SQL · 第 3 篇"
---

# 窗口函数

> **本篇目标**：彻底理解窗口函数的原理和用法。学完本篇，你能用 SQL 完成排名、同比环比、累计计算、滑动平均等高级分析任务。

## 为什么需要窗口函数

假设你有一个需求：**在每条订单旁边显示该用户的历史累计消费金额**。

用 GROUP BY 做聚合后，原始行的明细信息就丢失了——你只能得到"每个用户的总消费"，无法同时看到每条订单的详情。

窗口函数的核心价值就在于：**既能做聚合计算，又不丢失原始行的明细信息**。它在每一行上打开一个"窗口"，在窗口范围内进行计算，然后把结果写回到当前行。

## 窗口函数的基本语法

```sql
函数名(参数) OVER (
    [PARTITION BY 分区列]
    [ORDER BY 排序列 [ASC|DESC]]
    [ROWS/RANGE BETWEEN ... AND ...]
)
```

三个核心组成部分：

| 组成 | 作用 | 是否必须 |
|------|------|:--------:|
| PARTITION BY | 把数据分成多个分区，窗口函数在每个分区内独立计算 | 可选 |
| ORDER BY | 在分区内按指定列排序 | 视函数而定 |
| ROWS/RANGE | 定义窗口的边界范围（窗口框架） | 可选 |

> **PARTITION BY vs GROUP BY**：GROUP BY 把多行合并成一行输出；PARTITION BY 不合并行，只是在逻辑上划分分区，每行都保留在结果集中。

## 排名函数

排名是窗口函数最常用的场景之一。SQL 提供了三个排名函数，它们的区别在于**处理并列值**的方式。

### ROW_NUMBER、RANK、DENSE_RANK 对比

假设有以下成绩数据：

| student | score |
|---------|-------|
| 张三 | 95 |
| 李四 | 95 |
| 王五 | 90 |
| 赵六 | 85 |

```sql
SELECT
    student,
    score,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS row_num,
    RANK()       OVER (ORDER BY score DESC) AS rank_val,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank_val
FROM exam_scores;
```

结果：

| student | score | row_num | rank_val | dense_rank_val |
|---------|:-----:|:-------:|:--------:|:--------------:|
| 张三 | 95 | 1 | 1 | 1 |
| 李四 | 95 | 2 | 1 | 1 |
| 王五 | 90 | 3 | 3 | 2 |
| 赵六 | 85 | 4 | 4 | 3 |

- **ROW_NUMBER**：不管是否并列，编号连续递增（1, 2, 3, 4）
- **RANK**：并列同名次，后续名次跳过（1, 1, 3, 4）
- **DENSE_RANK**：并列同名次，后续名次不跳过（1, 1, 2, 3）

### 实战：每个品类销量 Top 3 的商品

```sql
WITH product_sales AS (
    SELECT
        p.product_id,
        p.product_name,
        p.category,
        SUM(oi.quantity) AS total_qty
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    JOIN orders o   ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY p.product_id, p.product_name, p.category
),
ranked AS (
    SELECT
        *,
        DENSE_RANK() OVER (
            PARTITION BY category
            ORDER BY total_qty DESC
        ) AS rk
    FROM product_sales
)
SELECT category, product_name, total_qty, rk
FROM ranked
WHERE rk <= 3
ORDER BY category, rk;
```

> **面试考点**：Top N 问题几乎是 SQL 面试的必考题型。核心套路就是 `DENSE_RANK() OVER (PARTITION BY ... ORDER BY ...) + WHERE rk <= N`。

### NTILE：等频分桶

`NTILE(n)` 将分区内的行均匀分成 n 个桶。

```sql
-- 将用户按消费金额分成 4 个等级（四分位）
SELECT
    user_id,
    total_spend,
    NTILE(4) OVER (ORDER BY total_spend DESC) AS quartile
FROM (
    SELECT user_id, SUM(total_amount) AS total_spend
    FROM orders
    WHERE status = 'completed'
    GROUP BY user_id
) t;
```

## 偏移函数：LAG 与 LEAD

`LAG` 和 `LEAD` 用来访问当前行**前面或后面的行**的数据，常用于计算同比、环比、变化量。

### 基本语法

```sql
LAG(列名, 偏移行数, 默认值)  OVER (PARTITION BY ... ORDER BY ...)
LEAD(列名, 偏移行数, 默认值) OVER (PARTITION BY ... ORDER BY ...)
```

- `LAG`：向前看（取前 N 行的值）
- `LEAD`：向后看（取后 N 行的值）
- 偏移行数默认为 1
- 默认值：当前面/后面没有足够的行时使用，默认为 NULL

### 实战：计算月环比增长率

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_FORMAT(order_date, '%Y-%m') AS month,
        SUM(total_amount) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY DATE_FORMAT(order_date, '%Y-%m')
)
SELECT
    month,
    revenue                                  AS 当月营收,
    LAG(revenue, 1) OVER (ORDER BY month)    AS 上月营收,
    ROUND(
        (revenue - LAG(revenue, 1) OVER (ORDER BY month))
        / LAG(revenue, 1) OVER (ORDER BY month) * 100
    , 2) AS 环比增长率百分比
FROM monthly_revenue
ORDER BY month;
```

### 实战：计算同比增长率

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_FORMAT(order_date, '%Y-%m') AS month,
        YEAR(order_date)  AS yr,
        MONTH(order_date) AS mn,
        SUM(total_amount) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY DATE_FORMAT(order_date, '%Y-%m'),
             YEAR(order_date), MONTH(order_date)
)
SELECT
    month,
    revenue                                            AS 当月营收,
    LAG(revenue, 12) OVER (ORDER BY yr, mn)            AS 去年同月营收,
    ROUND(
        (revenue - LAG(revenue, 12) OVER (ORDER BY yr, mn))
        / LAG(revenue, 12) OVER (ORDER BY yr, mn) * 100
    , 2) AS 同比增长率百分比
FROM monthly_revenue
ORDER BY month;
```

### 实战：计算用户两次购买的间隔天数

```sql
SELECT
    user_id,
    order_date,
    LAG(order_date, 1) OVER (
        PARTITION BY user_id ORDER BY order_date
    ) AS 上次购买日期,
    DATEDIFF(
        order_date,
        LAG(order_date, 1) OVER (
            PARTITION BY user_id ORDER BY order_date
        )
    ) AS 间隔天数
FROM orders
WHERE status = 'completed'
ORDER BY user_id, order_date;
```

## 聚合窗口函数：SUM / AVG / COUNT OVER

所有聚合函数（SUM、AVG、COUNT、MAX、MIN）都可以作为窗口函数使用。

### 累计求和（Running Total）

```sql
-- 计算每个用户的订单累计消费
SELECT
    user_id,
    order_date,
    total_amount,
    SUM(total_amount) OVER (
        PARTITION BY user_id
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS 累计消费
FROM orders
WHERE status = 'completed'
ORDER BY user_id, order_date;
```

> `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` 的含义是：从分区的第一行到当前行。这是 `ORDER BY` 存在时的默认窗口框架，可以省略不写。

### 全局聚合 vs 分区聚合

```sql
SELECT
    user_id,
    order_id,
    total_amount,
    -- 全表总额（没有 PARTITION BY）
    SUM(total_amount) OVER ()                              AS 全平台总额,
    -- 每个用户的总额
    SUM(total_amount) OVER (PARTITION BY user_id)          AS 用户总消费,
    -- 每条订单占该用户总消费的比例
    ROUND(
        total_amount / SUM(total_amount) OVER (PARTITION BY user_id) * 100
    , 2) AS 订单占比百分比
FROM orders
WHERE status = 'completed';
```

### 滑动平均（Moving Average）

```sql
-- 计算 7 天滑动平均日营收
WITH daily_revenue AS (
    SELECT
        order_date,
        SUM(total_amount) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY order_date
)
SELECT
    order_date,
    revenue,
    ROUND(
        AVG(revenue) OVER (
            ORDER BY order_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        )
    , 2) AS 七日滑动平均
FROM daily_revenue
ORDER BY order_date;
```

### 滑动窗口的边界定义

| 边界写法 | 含义 |
|---------|------|
| `UNBOUNDED PRECEDING` | 分区的第一行 |
| `N PRECEDING` | 当前行之前的第 N 行 |
| `CURRENT ROW` | 当前行 |
| `N FOLLOWING` | 当前行之后的第 N 行 |
| `UNBOUNDED FOLLOWING` | 分区的最后一行 |

常见的窗口框架组合：

```sql
-- 从分区第一行到当前行（累计）
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- 前 6 行到当前行（7 天滑动）
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW

-- 前 3 行到后 3 行（居中滑动）
ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING

-- 整个分区（全局计算）
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```

## ROWS vs RANGE

`ROWS` 按照物理行数来定义窗口边界，`RANGE` 按照值的范围来定义窗口边界。

```sql
-- ROWS：严格按行数计算，前 2 行 + 当前行 = 共 3 行
SUM(amount) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)

-- RANGE：按值范围计算，如果有日期相同的行，它们会被视为同一范围
SUM(amount) OVER (ORDER BY order_date RANGE BETWEEN INTERVAL 2 DAY PRECEDING AND CURRENT ROW)
```

> **实务建议**：大多数情况下使用 `ROWS` 即可。`RANGE` 在处理有重复值的排序列时行为不同，容易产生意外结果。

## FIRST_VALUE 与 LAST_VALUE

```sql
-- 每个用户的第一笔和最近一笔订单金额
SELECT
    user_id,
    order_date,
    total_amount,
    FIRST_VALUE(total_amount) OVER (
        PARTITION BY user_id ORDER BY order_date
    ) AS 首单金额,
    LAST_VALUE(total_amount) OVER (
        PARTITION BY user_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS 最近一单金额
FROM orders
WHERE status = 'completed';
```

> **LAST_VALUE 的坑**：默认窗口框架是 `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`，所以 `LAST_VALUE` 默认返回的是当前行而不是分区最后一行。必须显式指定 `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`。

## 综合实战：用户留存分析

用窗口函数分析用户首次购买后是否在 30 天内复购：

```sql
WITH user_orders AS (
    SELECT
        user_id,
        order_date,
        ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY order_date
        ) AS order_seq,
        FIRST_VALUE(order_date) OVER (
            PARTITION BY user_id ORDER BY order_date
        ) AS first_order_date
    FROM orders
    WHERE status = 'completed'
),
first_and_second AS (
    SELECT
        user_id,
        first_order_date,
        MIN(CASE WHEN order_seq = 2 THEN order_date END) AS second_order_date
    FROM user_orders
    GROUP BY user_id, first_order_date
)
SELECT
    DATE_FORMAT(first_order_date, '%Y-%m') AS 首购月份,
    COUNT(*)                               AS 新用户数,
    COUNT(second_order_date)               AS 有复购用户数,
    ROUND(
        COUNT(second_order_date) / COUNT(*) * 100, 2
    ) AS 复购率百分比,
    ROUND(
        AVG(DATEDIFF(second_order_date, first_order_date)), 1
    ) AS 平均复购间隔天数
FROM first_and_second
GROUP BY DATE_FORMAT(first_order_date, '%Y-%m')
ORDER BY 首购月份;
```

## 窗口函数速查表

| 函数 | 类型 | 用途 | 是否需要 ORDER BY |
|------|------|------|:-----------------:|
| ROW_NUMBER() | 排名 | 连续编号，无并列 | 是 |
| RANK() | 排名 | 有并列，名次跳跃 | 是 |
| DENSE_RANK() | 排名 | 有并列，名次不跳跃 | 是 |
| NTILE(n) | 排名 | 均匀分成 n 桶 | 是 |
| LAG(col, n) | 偏移 | 取前第 n 行的值 | 是 |
| LEAD(col, n) | 偏移 | 取后第 n 行的值 | 是 |
| FIRST_VALUE(col) | 偏移 | 取窗口第一行的值 | 是 |
| LAST_VALUE(col) | 偏移 | 取窗口最后一行的值 | 是 |
| SUM() OVER | 聚合 | 累计求和 / 分组求和 | 可选 |
| AVG() OVER | 聚合 | 滑动平均 / 分组平均 | 可选 |
| COUNT() OVER | 聚合 | 累计计数 / 分组计数 | 可选 |
| MAX() / MIN() OVER | 聚合 | 分区内最大最小值 | 可选 |

## 本篇小结

- 窗口函数的核心优势是**不丢失明细行的同时进行聚合计算**
- **PARTITION BY** 定义分区，**ORDER BY** 定义排序，**ROWS/RANGE** 定义窗口框架
- 排名函数（ROW_NUMBER、RANK、DENSE_RANK）是面试最高频考点
- LAG/LEAD 是计算环比、同比、间隔的利器
- SUM OVER 配合窗口框架可以实现累计求和、滑动平均

> **下一篇预告**：[SQL 面试高频题型](../04-sql-interview/index.html) —— 把前三篇学到的知识融会贯通，用实际面试题检验和巩固你的 SQL 能力。
