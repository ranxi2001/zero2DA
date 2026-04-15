---
title: "多表查询与子查询"
layout: default
description: "深入学习 SQL 多表关联查询（INNER JOIN、LEFT JOIN、RIGHT JOIN、FULL JOIN、自连接）与子查询（标量子查询、行子查询、表子查询、EXISTS），掌握复杂数据提取能力。"
eyebrow: "SQL · 第 2 篇"
---

# 多表查询与子查询

> **本篇目标**：掌握 JOIN 的所有类型和子查询的常见写法。学完本篇，你就能从多张表中提取、关联和整合数据，独立完成大多数业务分析需求。

## 为什么需要多表查询

在真实的数据库中，数据不会全部存在一张表里。为了减少数据冗余、保证数据一致性，数据库设计遵循**范式化原则**——把不同维度的数据拆分到不同的表中。

比如我们的电商数据模型：

- `users` 表存用户信息（user_id, username, city）
- `orders` 表存订单信息（order_id, user_id, total_amount）
- `products` 表存商品信息（product_id, product_name, category）
- `order_items` 表存订单明细（order_id, product_id, quantity）

当你想回答"北京用户最喜欢购买哪个品类的商品"这个问题时，你需要同时用到这四张表。这就是 JOIN 的用武之地。

## JOIN 的核心概念

JOIN 的本质是：**根据两张表之间的某个关联条件，把它们的行组合在一起**。

最常见的关联条件是主键-外键关系。例如 `orders` 表中的 `user_id` 是外键，指向 `users` 表中的 `user_id` 主键。

## INNER JOIN：内连接

INNER JOIN 只返回**两张表都匹配上的行**。如果某个用户没有下过单，或者某个订单的 user_id 在 users 表中不存在，这些行都不会出现在结果中。

```sql
-- 查询所有有订单的用户及其订单信息
SELECT
    u.user_id,
    u.username,
    u.city,
    o.order_id,
    o.order_date,
    o.total_amount
FROM users u
INNER JOIN orders o ON u.user_id = o.user_id;
```

> **别名约定**：通常给表名起短别名（`users` → `u`，`orders` → `o`），让查询更简洁易读。这在多表查询中尤为重要。

### 多表 INNER JOIN

```sql
-- 查询每笔订单的用户名、商品名、购买数量
SELECT
    u.username,
    p.product_name,
    p.category,
    oi.quantity,
    oi.unit_price,
    oi.quantity * oi.unit_price AS 小计
FROM order_items oi
INNER JOIN orders o    ON oi.order_id  = o.order_id
INNER JOIN users u     ON o.user_id    = u.user_id
INNER JOIN products p  ON oi.product_id = p.product_id
WHERE o.status = 'completed';
```

## LEFT JOIN：左连接

LEFT JOIN 返回**左表的所有行**，即使右表中没有匹配的行（没匹配到的部分用 NULL 填充）。

这是数据分析中**最常用的 JOIN 类型**，因为我们经常需要"保留所有用户，不管他们有没有下过单"。

```sql
-- 查询所有用户及其订单数（包括没下过单的用户）
SELECT
    u.user_id,
    u.username,
    u.city,
    COUNT(o.order_id) AS 订单数
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id, u.username, u.city;
```

### 用 LEFT JOIN 找"没有"的数据

一个非常经典的用法：找出从未下过单的用户。

```sql
-- 方法一：LEFT JOIN + IS NULL
SELECT u.user_id, u.username, u.city
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
WHERE o.order_id IS NULL;

-- 方法二：NOT EXISTS（后面会详细讲）
SELECT u.user_id, u.username, u.city
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.user_id
);

-- 方法三：NOT IN
SELECT user_id, username, city
FROM users
WHERE user_id NOT IN (
    SELECT DISTINCT user_id FROM orders
);
```

> **面试高频题**：这三种写法在功能上等价，但性能有差异。一般来说 `LEFT JOIN + IS NULL` 和 `NOT EXISTS` 性能更优，`NOT IN` 在子查询结果包含 NULL 时可能产生意外结果。

## RIGHT JOIN：右连接

RIGHT JOIN 与 LEFT JOIN 相反，返回**右表的所有行**。实际工作中较少使用，因为任何 RIGHT JOIN 都可以通过交换表的顺序改写为 LEFT JOIN。

```sql
-- 以下两条查询完全等价
-- RIGHT JOIN 写法
SELECT o.order_id, u.username
FROM users u
RIGHT JOIN orders o ON u.user_id = o.user_id;

-- 等价的 LEFT JOIN 写法（推荐）
SELECT o.order_id, u.username
FROM orders o
LEFT JOIN users u ON o.user_id = u.user_id;
```

> **实务建议**：统一使用 LEFT JOIN，避免 RIGHT JOIN。这样代码风格更一致，可读性更好。

## FULL OUTER JOIN：全外连接

FULL OUTER JOIN 返回**两张表中所有的行**。左表有但右表没有的，右表列填 NULL；右表有但左表没有的，左表列填 NULL。

```sql
-- PostgreSQL / SQL Server 支持 FULL OUTER JOIN
SELECT
    u.user_id,
    u.username,
    o.order_id,
    o.total_amount
FROM users u
FULL OUTER JOIN orders o ON u.user_id = o.user_id;
```

> **注意**：MySQL 不支持 `FULL OUTER JOIN`，需要用 `LEFT JOIN UNION RIGHT JOIN` 来模拟：

```sql
-- MySQL 模拟 FULL OUTER JOIN
SELECT u.user_id, u.username, o.order_id, o.total_amount
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id

UNION

SELECT u.user_id, u.username, o.order_id, o.total_amount
FROM users u
RIGHT JOIN orders o ON u.user_id = o.user_id;
```

## 自连接（Self Join）

自连接是指**一张表与自己连接**。听起来奇怪，但在实际业务中非常有用。

### 场景一：员工表——找每个员工的上级

```sql
-- employees 表：employee_id, name, manager_id
SELECT
    e.name   AS 员工,
    m.name   AS 上级
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;
```

### 场景二：找同城用户

```sql
-- 找出和 user_id = 1 住在同一城市的所有其他用户
SELECT u2.user_id, u2.username, u2.city
FROM users u1
INNER JOIN users u2 ON u1.city = u2.city
WHERE u1.user_id = 1
  AND u2.user_id != 1;
```

### 场景三：连续日期分析

```sql
-- 找出连续两天都有订单的用户
SELECT DISTINCT a.user_id
FROM orders a
INNER JOIN orders b
  ON a.user_id = b.user_id
  AND DATEDIFF(b.order_date, a.order_date) = 1;
```

## CROSS JOIN：交叉连接

CROSS JOIN 产生两张表的**笛卡尔积**——左表的每一行与右表的每一行都组合一次。看起来用处不大，但在生成"日历维度表"或"补全缺失日期"时非常实用。

```sql
-- 生成所有城市 × 所有月份的组合（用于补全无数据的月份）
SELECT
    c.city,
    m.month
FROM (SELECT DISTINCT city FROM users) c
CROSS JOIN (
    SELECT '2024-01' AS month
    UNION ALL SELECT '2024-02'
    UNION ALL SELECT '2024-03'
) m;
```

## 子查询（Subquery）

子查询是嵌套在另一条 SQL 语句内部的查询。根据返回结果的形式，可以分为以下几类。

### 标量子查询：返回单个值

```sql
-- 查询消费金额高于平均值的订单
SELECT order_id, user_id, total_amount
FROM orders
WHERE total_amount > (
    SELECT AVG(total_amount) FROM orders WHERE status = 'completed'
);
```

### 行子查询 / 列子查询：返回一列值

```sql
-- 查询下过单的用户信息（用 IN）
SELECT * FROM users
WHERE user_id IN (
    SELECT DISTINCT user_id FROM orders
);
```

### 表子查询（派生表）：在 FROM 中使用

```sql
-- 先计算每个用户的总消费，再筛选出高消费用户
SELECT *
FROM (
    SELECT
        user_id,
        COUNT(*)          AS 订单数,
        SUM(total_amount) AS 总消费
    FROM orders
    WHERE status = 'completed'
    GROUP BY user_id
) user_summary
WHERE 总消费 > 5000
ORDER BY 总消费 DESC;
```

> **派生表必须有别名**：`FROM (子查询) 别名` 中的别名是强制要求的，否则会报错。

### 关联子查询（Correlated Subquery）

关联子查询引用了外部查询的列，因此对外部查询的每一行都重新执行一次。

```sql
-- 找出每个用户的最新订单
SELECT o.*
FROM orders o
WHERE o.order_date = (
    SELECT MAX(o2.order_date)
    FROM orders o2
    WHERE o2.user_id = o.user_id
);
```

### EXISTS 与 NOT EXISTS

`EXISTS` 检查子查询是否返回至少一行结果，常用于"是否存在"类判断。

```sql
-- 找出买过"电子产品"类别商品的用户
SELECT u.user_id, u.username
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p     ON oi.product_id = p.product_id
    WHERE o.user_id = u.user_id
      AND p.category = '电子产品'
);
```

## WITH（CTE）：公用表表达式

CTE（Common Table Expression）用 `WITH` 关键字定义，可以把复杂查询拆分成多个命名步骤，极大地提升可读性。

```sql
-- 分析思路：先算每个用户的月消费，再按消费等级统计人数
WITH monthly_spend AS (
    SELECT
        user_id,
        DATE_FORMAT(order_date, '%Y-%m') AS month,
        SUM(total_amount)                AS spend
    FROM orders
    WHERE status = 'completed'
    GROUP BY user_id, DATE_FORMAT(order_date, '%Y-%m')
),
user_level AS (
    SELECT
        user_id,
        AVG(spend) AS avg_monthly_spend,
        CASE
            WHEN AVG(spend) >= 3000 THEN '高价值'
            WHEN AVG(spend) >= 1000 THEN '中等'
            ELSE '低消费'
        END AS level
    FROM monthly_spend
    GROUP BY user_id
)
SELECT
    level       AS 用户等级,
    COUNT(*)    AS 用户数,
    ROUND(AVG(avg_monthly_spend), 2) AS 平均月消费
FROM user_level
GROUP BY level
ORDER BY 平均月消费 DESC;
```

> **CTE vs 子查询**：功能上两者等价，但 CTE 更易读、可复用（同一个 CTE 可以在后续查询中引用多次）。面试和实际工作中都推荐使用 CTE。

## UNION 与 UNION ALL

`UNION` 将两个查询结果**上下拼接**（合并行），要求两个查询的列数和数据类型一致。

```sql
-- UNION：去重合并
SELECT city FROM users WHERE city LIKE '北%'
UNION
SELECT city FROM users WHERE city LIKE '上%';

-- UNION ALL：不去重合并（性能更好）
SELECT user_id, '已完成' AS 来源 FROM orders WHERE status = 'completed'
UNION ALL
SELECT user_id, '已取消' AS 来源 FROM orders WHERE status = 'cancelled';
```

> **性能提示**：如果你确定两个结果集没有重复，或者不需要去重，优先用 `UNION ALL`，它不需要执行去重操作，速度更快。

## 综合实战：北京用户最喜欢的商品品类

```sql
WITH bj_orders AS (
    SELECT o.order_id
    FROM orders o
    JOIN users u ON o.user_id = u.user_id
    WHERE u.city = '北京'
      AND o.status = 'completed'
),
category_stats AS (
    SELECT
        p.category,
        COUNT(DISTINCT bo.order_id) AS 订单数,
        SUM(oi.quantity)            AS 购买件数,
        SUM(oi.quantity * oi.unit_price) AS 消费金额
    FROM bj_orders bo
    JOIN order_items oi ON bo.order_id = oi.order_id
    JOIN products p     ON oi.product_id = p.product_id
    GROUP BY p.category
)
SELECT
    category  AS 品类,
    订单数,
    购买件数,
    ROUND(消费金额, 2) AS 消费金额,
    ROUND(消费金额 / SUM(消费金额) OVER (), 4) AS 消费占比
FROM category_stats
ORDER BY 消费金额 DESC;
```

## JOIN 类型速查表

| JOIN 类型 | 返回结果 | 典型场景 |
|-----------|---------|---------|
| INNER JOIN | 两表都匹配的行 | 查询有订单的用户 |
| LEFT JOIN | 左表全部 + 右表匹配的行 | 保留所有用户，包括未下单的 |
| RIGHT JOIN | 右表全部 + 左表匹配的行 | 很少使用，改写为 LEFT JOIN |
| FULL OUTER JOIN | 两表全部行 | 合并两个数据源 |
| CROSS JOIN | 笛卡尔积 | 生成日历维度、补全缺失组合 |
| Self JOIN | 同一张表连接自身 | 上下级关系、连续日期分析 |

## 本篇小结

- **JOIN** 解决的是"如何从多张表中关联数据"的问题，其中 INNER JOIN 和 LEFT JOIN 是最常用的两种
- **子查询** 解决的是"如何把一个查询的结果作为另一个查询的输入"的问题
- **CTE** 是子查询的可读性升级版，推荐在复杂分析中使用
- 面试中 JOIN 题目占比极高，务必理解每种 JOIN 的行为差异

> **下一篇预告**：[窗口函数](../03-window-functions/index.html) —— SQL 中最强大也最容易让人困惑的特性。掌握窗口函数，你的 SQL 分析能力会产生质变。
