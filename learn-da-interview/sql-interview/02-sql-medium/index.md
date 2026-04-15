---
title: SQL 中等题精讲
layout: default
description: 数据分析师面试 SQL 中等难度题型精讲——多表 JOIN、子查询、窗口函数，附详细解题思路与代码。
eyebrow: "DA 面试 · SQL 面试"
---

# SQL 中等题精讲

中等难度的 SQL 题是数据分析师面试中**出现频率最高**的题型。它们通常需要你组合使用多表关联、子查询、窗口函数等技巧来解决实际业务问题。掌握这个级别的题目，能让你通过绝大多数 DA 面试的 SQL 环节。

## 题型概览

| 题型 | 考察点 | 出现频率 |
|------|--------|---------|
| 多表 JOIN | INNER/LEFT/RIGHT JOIN 的选择与使用 | 非常高 |
| 子查询与 CTE | 复杂逻辑的分步实现 | 非常高 |
| 窗口函数基础 | ROW_NUMBER、RANK、LAG/LEAD | 非常高 |
| 留存率计算 | 次日/7 日留存的 SQL 实现 | 高 |
| 连续天数问题 | 连续登录、连续增长的判断 | 高 |
| 累计计算 | 累计求和、滚动平均 | 中 |

## 题目 1：用户首次购买分析

> **题目**：订单表 `orders` 包含 `user_id`、`order_date`、`amount`。请查询每个用户的首次购买日期和首次购买金额。

**方法一：窗口函数**

```sql
WITH ranked_orders AS (
    SELECT user_id,
           order_date,
           amount,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date) AS rn
    FROM orders
)
SELECT user_id, order_date AS first_order_date, amount AS first_order_amount
FROM ranked_orders
WHERE rn = 1;
```

**方法二：子查询**

```sql
SELECT o.user_id, o.order_date AS first_order_date, o.amount AS first_order_amount
FROM orders o
INNER JOIN (
    SELECT user_id, MIN(order_date) AS min_date
    FROM orders
    GROUP BY user_id
) first_order ON o.user_id = first_order.user_id
                AND o.order_date = first_order.min_date;
```

> **对比分析**：窗口函数方法更简洁、更直观。子查询方法在 `order_date` 有重复时可能返回多行，需要额外处理。面试中优先使用 CTE + 窗口函数。

## 题目 2：计算次日留存率

> **题目**：用户活动表 `user_activity` 包含 `user_id`、`activity_date`。请计算每天新用户的次日留存率。假设用户首次出现的日期为其注册日期。

```sql
WITH first_day AS (
    -- 找到每个用户的首次活跃日期（即注册日期）
    SELECT user_id,
           MIN(activity_date) AS register_date
    FROM user_activity
    GROUP BY user_id
),
retention AS (
    -- 检查注册次日是否有活跃记录
    SELECT f.register_date,
           f.user_id,
           CASE WHEN a.user_id IS NOT NULL THEN 1 ELSE 0 END AS retained
    FROM first_day f
    LEFT JOIN user_activity a
        ON f.user_id = a.user_id
        AND a.activity_date = DATE_ADD(f.register_date, INTERVAL 1 DAY)
)
SELECT register_date,
       COUNT(*) AS new_users,
       SUM(retained) AS retained_users,
       ROUND(SUM(retained) * 100.0 / COUNT(*), 2) AS day1_retention_rate
FROM retention
GROUP BY register_date
ORDER BY register_date;
```

> **思路拆解**：这道题的关键是分两步——先找到"注册日"（首次活跃日），再检查"注册次日"是否仍然活跃。`LEFT JOIN` 确保没有次日活跃的用户也被包含在计算中。

## 题目 3：同比/环比增长率

> **题目**：月度收入表 `monthly_revenue` 包含 `month`（格式 '2025-01'）、`revenue`。请计算每个月的收入环比增长率（MoM Growth）。

```sql
SELECT month,
       revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
       ROUND(
           (revenue - LAG(revenue) OVER (ORDER BY month))
           * 100.0
           / LAG(revenue) OVER (ORDER BY month),
           2
       ) AS mom_growth_pct
FROM monthly_revenue
ORDER BY month;
```

> **核心知识点**：`LAG(column, offset, default)` 获取当前行之前 offset 行的值。第一个月没有上月数据，`LAG` 返回 NULL，除法结果也为 NULL——这是正确的行为。`LEAD` 是其反义函数，获取后面 N 行的值。

## 题目 4：商品排名与 Top N

> **题目**：销售表 `sales` 包含 `product_id`、`category`、`sale_amount`。请找出每个品类中销售额 Top 3 的商品。

```sql
WITH product_sales AS (
    SELECT product_id,
           category,
           SUM(sale_amount) AS total_sales,
           RANK() OVER (PARTITION BY category ORDER BY SUM(sale_amount) DESC) AS sales_rank
    FROM sales
    GROUP BY product_id, category
)
SELECT category, product_id, total_sales, sales_rank
FROM product_sales
WHERE sales_rank <= 3
ORDER BY category, sales_rank;
```

> **RANK vs DENSE_RANK vs ROW_NUMBER 的区别**：
>
> | 函数 | 并列处理 | 示例排名 |
> |------|---------|---------|
> | `ROW_NUMBER` | 不允许并列，强制唯一排名 | 1, 2, 3, 4 |
> | `RANK` | 允许并列，跳过后续排名 | 1, 2, 2, 4 |
> | `DENSE_RANK` | 允许并列，不跳过后续排名 | 1, 2, 2, 3 |
>
> 面试中问"Top 3"时，用 `RANK` 最安全——并列第 3 名的全部包含。

## 题目 5：连续登录天数

> **题目**：登录表 `logins` 包含 `user_id`、`login_date`（已去重，每天只有一条记录）。请找出连续登录 3 天及以上的用户和他们的最长连续登录天数。

```sql
WITH login_groups AS (
    SELECT user_id,
           login_date,
           -- 用日期减去行号得到的值，连续日期会得到相同的值
           DATE_SUB(login_date, INTERVAL ROW_NUMBER() OVER (
               PARTITION BY user_id ORDER BY login_date
           ) DAY) AS grp
    FROM logins
),
consecutive AS (
    SELECT user_id,
           grp,
           COUNT(*) AS consecutive_days,
           MIN(login_date) AS start_date,
           MAX(login_date) AS end_date
    FROM login_groups
    GROUP BY user_id, grp
)
SELECT user_id,
       MAX(consecutive_days) AS max_consecutive_days
FROM consecutive
WHERE consecutive_days >= 3
GROUP BY user_id
ORDER BY max_consecutive_days DESC;
```

> **经典技巧解析**：这道题使用了"日期减去行号"的经典方法。如果用户在 1/1、1/2、1/3 连续登录，减去行号 1、2、3 后分别得到 12/31、12/31、12/31——相同的值说明是连续的。这是 SQL 面试中判断"连续性"的标准技巧。

## 题目 6：累计求和

> **题目**：每日销售表 `daily_sales` 包含 `sale_date`、`revenue`。请计算每天的累计收入和 7 日滚动平均收入。

```sql
SELECT sale_date,
       revenue,
       SUM(revenue) OVER (ORDER BY sale_date) AS cumulative_revenue,
       ROUND(
           AVG(revenue) OVER (
               ORDER BY sale_date
               ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
           ), 2
       ) AS rolling_7day_avg
FROM daily_sales
ORDER BY sale_date;
```

> **窗口帧详解**：
> - `SUM(...) OVER (ORDER BY date)` 默认窗口帧是从第一行到当前行，即累计求和
> - `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` 明确指定当前行及前 6 行（共 7 行），实现 7 日滚动平均
> - 注意：前几天不足 7 行时会用实际行数计算平均值

## 题目 7：同时在线人数

> **题目**：会议表 `meetings` 包含 `meeting_id`、`start_time`、`end_time`（datetime 类型）。请找出同时进行的会议最多有多少个。

```sql
WITH time_events AS (
    SELECT start_time AS event_time, 1 AS delta FROM meetings
    UNION ALL
    SELECT end_time AS event_time, -1 AS delta FROM meetings
)
SELECT MAX(concurrent) AS max_concurrent
FROM (
    SELECT event_time,
           SUM(delta) OVER (ORDER BY event_time, delta) AS concurrent
    FROM time_events
) t;
```

> **思路解析**：将每个会议拆成两个事件——开始（+1）和结束（-1）。按时间排序后做累计求和，就得到了每个时间点的同时在线数。`ORDER BY event_time, delta` 确保同一时间点先处理结束（-1）再处理开始（+1），避免高估。

## 题目 8：用户购买间隔分析

> **题目**：订单表 `orders` 包含 `user_id`、`order_date`。请计算每个用户相邻两次购买的平均间隔天数。

```sql
WITH order_with_prev AS (
    SELECT user_id,
           order_date,
           LAG(order_date) OVER (PARTITION BY user_id ORDER BY order_date) AS prev_order_date
    FROM orders
)
SELECT user_id,
       ROUND(AVG(DATEDIFF(order_date, prev_order_date)), 1) AS avg_days_between_orders
FROM order_with_prev
WHERE prev_order_date IS NOT NULL
GROUP BY user_id
ORDER BY avg_days_between_orders;
```

> **考察点**：`LAG` 窗口函数 + 日期差计算 + 过滤首次订单（没有上一次）。面试中这类"用户行为间隔"的分析非常常见，因为它直接关联到用户活跃度和复购分析。

## 中等题的答题策略

1. **先画出数据流**：在草稿纸上画出每一步的中间结果长什么样
2. **善用 CTE**：把复杂查询拆成多个 CTE 步骤，每步只做一件事
3. **窗口函数优先**：能用窗口函数解决的问题不要用自关联（self-join），可读性和性能都更好
4. **说出假设**：面试时主动说"我假设日期已经去重/数据没有 NULL"，展示你的严谨性
5. **时间控制**：中等题目标在 10-15 分钟内完成

## 本篇小结

- 中等题是 DA 面试 SQL 环节的核心，需要熟练掌握
- 多表 JOIN 要区分 INNER/LEFT/RIGHT 的使用场景
- 窗口函数（`ROW_NUMBER`、`RANK`、`LAG`/`LEAD`、`SUM OVER`）是解题利器
- 留存率计算、连续天数、累计求和是高频题型
- 用 CTE 拆分复杂逻辑，每步只做一件事

想挑战更高难度？进入 [SQL 困难题精讲](../03-sql-hard/)。
