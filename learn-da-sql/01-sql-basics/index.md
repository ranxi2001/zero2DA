---
title: "SQL 基础语法"
layout: default
description: "系统学习 SQL 基础语法：SELECT、WHERE、GROUP BY、HAVING、ORDER BY、LIMIT，掌握数据查询的核心能力。"
eyebrow: "SQL · 第 1 篇"
---

# SQL 基础语法

> **本篇目标**：掌握 SQL 查询的六大核心子句 —— SELECT、WHERE、GROUP BY、HAVING、ORDER BY、LIMIT。学完本篇，你就能独立完成大多数日常数据提取工作。

## 什么是 SQL

SQL（Structured Query Language，结构化查询语言）是用来与关系型数据库"对话"的标准语言。你可以把数据库想象成一个巨大的 Excel 文件，而 SQL 就是你对这个文件发出的指令 —— "帮我查一下上个月北京用户的订单总额"。

SQL 的核心能力分为四类：

| 类型 | 关键字 | 说明 |
|------|-------|------|
| 查询（Query） | SELECT | 从表中读取数据 |
| 插入（Insert） | INSERT | 向表中添加新数据 |
| 更新（Update） | UPDATE | 修改表中已有的数据 |
| 删除（Delete） | DELETE | 从表中删除数据 |

作为数据分析师，你 **95% 的工作** 都围绕 `SELECT` 查询展开。本篇将聚焦于此。

## SQL 查询的执行顺序

在写 SQL 之前，必须先理解一个关键概念：**SQL 的书写顺序和执行顺序是不同的**。

**书写顺序**：

```sql
SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
```

**执行顺序**：

```sql
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

> 理解执行顺序至关重要。例如，你不能在 `WHERE` 子句中使用 `SELECT` 里定义的别名，因为 `WHERE` 在 `SELECT` 之前执行。

## SELECT：选取列

`SELECT` 是每条 SQL 查询的起点，用来指定你想要查看哪些列。

### 基本用法

```sql
-- 查看所有列
SELECT *
FROM users;

-- 查看指定列
SELECT user_id, username, city
FROM users;

-- 使用别名（AS）让列名更易读
SELECT
    user_id       AS 用户ID,
    username      AS 用户名,
    created_at    AS 注册日期
FROM users;
```

### 使用表达式和函数

```sql
-- 计算列
SELECT
    order_id,
    quantity,
    unit_price,
    quantity * unit_price AS 小计金额
FROM order_items;

-- 使用内置函数
SELECT
    user_id,
    UPPER(username)            AS 大写用户名,
    YEAR(created_at)           AS 注册年份,
    DATEDIFF(CURDATE(), created_at) AS 注册天数
FROM users;
```

### DISTINCT：去重

```sql
-- 查看所有不重复的城市
SELECT DISTINCT city
FROM users;

-- 多列去重：城市 + 注册年份的唯一组合
SELECT DISTINCT city, YEAR(created_at) AS reg_year
FROM users;
```

## WHERE：条件过滤

`WHERE` 子句在 `FROM` 之后执行，用于筛选满足条件的行。

### 比较运算符

```sql
-- 等于
SELECT * FROM orders WHERE status = 'completed';

-- 不等于
SELECT * FROM orders WHERE status != 'cancelled';

-- 大于、小于
SELECT * FROM orders WHERE total_amount > 500;
SELECT * FROM orders WHERE order_date < '2024-06-01';

-- BETWEEN：范围查询（包含两端）
SELECT * FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31';

-- IN：匹配多个值
SELECT * FROM users
WHERE city IN ('北京', '上海', '深圳');
```

### 逻辑运算符

```sql
-- AND：同时满足多个条件
SELECT * FROM orders
WHERE status = 'completed'
  AND total_amount > 200
  AND order_date >= '2024-01-01';

-- OR：满足任一条件
SELECT * FROM users
WHERE city = '北京' OR city = '上海';

-- NOT：取反
SELECT * FROM orders
WHERE NOT status = 'cancelled';
```

### LIKE：模糊匹配

```sql
-- % 匹配任意多个字符
SELECT * FROM users WHERE username LIKE '张%';    -- 以"张"开头
SELECT * FROM users WHERE email LIKE '%@gmail.com'; -- Gmail 邮箱

-- _ 匹配恰好一个字符
SELECT * FROM users WHERE username LIKE '张_';     -- "张"后跟一个字
```

### NULL 值处理

```sql
-- 判断 NULL 不能用 = ，必须用 IS NULL / IS NOT NULL
SELECT * FROM orders WHERE total_amount IS NULL;
SELECT * FROM orders WHERE total_amount IS NOT NULL;
```

> **常见踩坑**：`WHERE total_amount = NULL` 永远不会返回任何行，因为 NULL 不等于任何值，包括它自身。必须使用 `IS NULL`。

## GROUP BY：分组聚合

`GROUP BY` 是数据分析中最重要的子句之一。它将数据按照指定的列分组，然后对每个组应用聚合函数。

### 常用聚合函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `COUNT(*)` | 计算行数 | 统计订单数 |
| `COUNT(DISTINCT col)` | 计算去重后的值数 | 统计独立用户数 |
| `SUM(col)` | 求和 | 计算总金额 |
| `AVG(col)` | 求平均 | 计算平均客单价 |
| `MAX(col)` | 最大值 | 最高消费金额 |
| `MIN(col)` | 最小值 | 最早下单时间 |

### 基本分组

```sql
-- 每个城市有多少用户
SELECT
    city,
    COUNT(*) AS 用户数
FROM users
GROUP BY city;

-- 每个月的订单数和总金额
SELECT
    DATE_FORMAT(order_date, '%Y-%m') AS 月份,
    COUNT(*)                         AS 订单数,
    SUM(total_amount)                AS 总金额,
    AVG(total_amount)                AS 平均客单价
FROM orders
WHERE status = 'completed'
GROUP BY DATE_FORMAT(order_date, '%Y-%m');
```

### 多列分组

```sql
-- 每个城市每个月的订单统计
SELECT
    u.city,
    DATE_FORMAT(o.order_date, '%Y-%m') AS 月份,
    COUNT(DISTINCT o.user_id)          AS 下单用户数,
    COUNT(o.order_id)                  AS 订单数,
    SUM(o.total_amount)                AS 总金额
FROM orders o
JOIN users u ON o.user_id = u.user_id
WHERE o.status = 'completed'
GROUP BY u.city, DATE_FORMAT(o.order_date, '%Y-%m');
```

### 分组后的注意事项

> **规则**：`SELECT` 中出现的非聚合列，必须出现在 `GROUP BY` 中。否则数据库不知道该显示分组中的哪一行的值。

```sql
-- 错误写法：username 没有出现在 GROUP BY 中
SELECT city, username, COUNT(*)
FROM users
GROUP BY city;  -- 会报错或返回不确定结果

-- 正确写法
SELECT city, COUNT(*) AS 用户数
FROM users
GROUP BY city;
```

## HAVING：过滤分组结果

`HAVING` 和 `WHERE` 的区别在于：`WHERE` 过滤的是原始行，`HAVING` 过滤的是分组聚合后的结果。

```sql
-- 找出订单数超过 10 的用户
SELECT
    user_id,
    COUNT(*) AS 订单数,
    SUM(total_amount) AS 总消费
FROM orders
WHERE status = 'completed'
GROUP BY user_id
HAVING COUNT(*) > 10;
```

一个更复杂的例子 —— 找出月均消费超过 1000 元的城市：

```sql
SELECT
    u.city,
    DATE_FORMAT(o.order_date, '%Y-%m') AS 月份,
    SUM(o.total_amount) AS 月消费额
FROM orders o
JOIN users u ON o.user_id = u.user_id
WHERE o.status = 'completed'
GROUP BY u.city, DATE_FORMAT(o.order_date, '%Y-%m')
HAVING SUM(o.total_amount) > 1000
ORDER BY 月消费额 DESC;
```

> **WHERE vs HAVING 速记**：WHERE 在 GROUP BY 之前执行，过滤原始数据行；HAVING 在 GROUP BY 之后执行，过滤分组结果。如果条件不涉及聚合函数，优先用 WHERE（性能更好）。

## ORDER BY：排序

`ORDER BY` 用于对查询结果排序，默认升序（ASC），可以指定降序（DESC）。

```sql
-- 按总金额降序排列
SELECT
    user_id,
    SUM(total_amount) AS 总消费
FROM orders
WHERE status = 'completed'
GROUP BY user_id
ORDER BY 总消费 DESC;

-- 多列排序：先按城市升序，再按注册时间降序
SELECT user_id, city, created_at
FROM users
ORDER BY city ASC, created_at DESC;

-- 按列序号排序（不推荐，但面试时可能见到）
SELECT user_id, username, created_at
FROM users
ORDER BY 3 DESC;  -- 按第 3 列即 created_at 降序
```

## LIMIT：限制返回行数

`LIMIT` 用于只返回前 N 行结果，在数据量大时非常实用。

```sql
-- 消费最高的前 10 名用户
SELECT
    user_id,
    SUM(total_amount) AS 总消费
FROM orders
WHERE status = 'completed'
GROUP BY user_id
ORDER BY 总消费 DESC
LIMIT 10;

-- LIMIT + OFFSET：分页查询（跳过前 20 行，取 10 行）
SELECT * FROM products
ORDER BY product_id
LIMIT 10 OFFSET 20;
```

> 不同数据库的分页语法略有差异：MySQL 用 `LIMIT 10 OFFSET 20`，SQL Server 用 `OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY`，Oracle 用 `ROWNUM` 或 `FETCH FIRST`。

## 综合实战：一个完整的分析查询

**需求**：找出 2024 年第一季度，消费金额排名前 5 的城市，要求每个城市至少有 50 笔已完成订单。

```sql
SELECT
    u.city                         AS 城市,
    COUNT(o.order_id)              AS 订单数,
    COUNT(DISTINCT o.user_id)      AS 下单用户数,
    SUM(o.total_amount)            AS 总消费额,
    ROUND(AVG(o.total_amount), 2)  AS 平均客单价
FROM orders o
JOIN users u ON o.user_id = u.user_id
WHERE o.status = 'completed'
  AND o.order_date BETWEEN '2024-01-01' AND '2024-03-31'
GROUP BY u.city
HAVING COUNT(o.order_id) >= 50
ORDER BY 总消费额 DESC
LIMIT 5;
```

让我们逐行拆解这条查询：

1. **FROM + JOIN**：先将 `orders` 表和 `users` 表通过 `user_id` 关联
2. **WHERE**：筛选已完成的订单，并限定时间范围为 2024 年 Q1
3. **GROUP BY**：按城市分组
4. **HAVING**：过滤掉订单数不足 50 的城市
5. **SELECT**：计算每个城市的订单数、用户数、总消费额、平均客单价
6. **ORDER BY**：按总消费额从高到低排序
7. **LIMIT**：只取前 5 名

这就是 SQL 基础语法的完整链条。掌握这六大子句，你就能应对日常工作中 80% 的数据提取需求。

## CASE WHEN：条件表达式

`CASE WHEN` 类似于编程中的 if-else，在 `SELECT` 中用来根据条件生成新的列值。

```sql
-- 根据消费金额划分用户等级
SELECT
    user_id,
    SUM(total_amount) AS 总消费,
    CASE
        WHEN SUM(total_amount) >= 5000 THEN '高价值用户'
        WHEN SUM(total_amount) >= 1000 THEN '中等用户'
        ELSE '低消费用户'
    END AS 用户等级
FROM orders
WHERE status = 'completed'
GROUP BY user_id
ORDER BY 总消费 DESC;
```

`CASE WHEN` 也常用于做**透视统计**（把行转列）：

```sql
-- 统计每个城市不同状态的订单数
SELECT
    u.city,
    COUNT(CASE WHEN o.status = 'completed' THEN 1 END) AS 已完成,
    COUNT(CASE WHEN o.status = 'cancelled' THEN 1 END) AS 已取消,
    COUNT(CASE WHEN o.status = 'pending'   THEN 1 END) AS 待处理
FROM orders o
JOIN users u ON o.user_id = u.user_id
GROUP BY u.city;
```

## 常用日期函数

数据分析离不开时间维度。以下是 MySQL 中最常用的日期函数：

```sql
-- 获取当前日期
SELECT CURDATE();                           -- 2024-07-15

-- 提取年、月、日
SELECT YEAR(order_date)  FROM orders;       -- 2024
SELECT MONTH(order_date) FROM orders;       -- 7
SELECT DAY(order_date)   FROM orders;       -- 15

-- 日期格式化
SELECT DATE_FORMAT(order_date, '%Y-%m') FROM orders;  -- 2024-07

-- 日期加减
SELECT DATE_ADD('2024-01-01', INTERVAL 30 DAY);       -- 2024-01-31
SELECT DATEDIFF('2024-03-01', '2024-01-01');           -- 60 天
```

## 本篇小结

| 子句 | 作用 | 执行顺序 |
|------|------|:--------:|
| FROM | 指定数据来源 | 1 |
| WHERE | 过滤原始数据行 | 2 |
| GROUP BY | 按列分组 | 3 |
| HAVING | 过滤分组后的结果 | 4 |
| SELECT | 选取列、计算、聚合 | 5 |
| ORDER BY | 排序 | 6 |
| LIMIT | 限制返回行数 | 7 |

> **下一篇预告**：[多表查询与子查询](../02-join-subquery/index.html) —— 当数据分散在多张表时，如何用 JOIN 和子查询把它们关联起来？这是从"能写简单查询"到"能独立做分析"的关键一步。
