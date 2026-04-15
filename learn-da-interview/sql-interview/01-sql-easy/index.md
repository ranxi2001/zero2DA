---
title: SQL 简单题精讲
layout: default
description: 数据分析师面试 SQL 简单题精讲——基础查询、过滤、聚合、排序，附详细解题思路与代码。
eyebrow: "DA 面试 · SQL 面试"
---

# SQL 简单题精讲

SQL 面试中的简单题是**送分题**——它们考察的是最基础的查询能力，如果在这里丢分，面试官会对你的 SQL 能力产生根本性的质疑。本篇精选了面试中出现频率最高的简单题型，每道题都附有详细的解题思路和代码。

## 题型概览

| 题型 | 考察点 | 出现频率 |
|------|--------|---------|
| 基础 SELECT + WHERE | 条件过滤、数据类型处理 | 非常高 |
| GROUP BY + 聚合函数 | 分组统计、COUNT/SUM/AVG | 非常高 |
| ORDER BY + LIMIT | 排序与 Top N 查询 | 高 |
| DISTINCT 去重 | 去重计数 | 高 |
| CASE WHEN | 条件分支 | 高 |
| 日期函数 | 日期过滤、日期计算 | 中 |

## 题目 1：查询活跃用户

> **题目**：有一张用户行为表 `user_actions`，包含字段 `user_id`、`action_type`、`action_date`。请查询 2025 年 1 月有过登录行为（action_type = 'login'）的所有**不重复**用户 ID。

**解题思路**：
1. 过滤条件：`action_type = 'login'` 且 `action_date` 在 2025 年 1 月
2. 对 `user_id` 去重

```sql
SELECT DISTINCT user_id
FROM user_actions
WHERE action_type = 'login'
  AND action_date >= '2025-01-01'
  AND action_date < '2025-02-01';
```

> **注意事项**：日期过滤推荐用 `>= ... AND < ...` 的方式，而不是 `BETWEEN`，因为 `BETWEEN` 在不同数据库对时间戳的处理可能不一致。也不要用 `YEAR(action_date) = 2025 AND MONTH(action_date) = 1`，因为对字段施加函数会导致索引失效。

## 题目 2：统计各类订单数量

> **题目**：订单表 `orders` 包含 `order_id`、`user_id`、`order_status`（取值为 'completed'、'cancelled'、'pending'）、`order_date`。请统计每种订单状态的订单数量，并按数量从大到小排序。

```sql
SELECT order_status,
       COUNT(*) AS order_count
FROM orders
GROUP BY order_status
ORDER BY order_count DESC;
```

**考察点**：`GROUP BY` + `COUNT` + `ORDER BY` 的标准用法。面试中有些人会忘记 `ORDER BY` 或者写成 `ORDER BY COUNT(*)`——两种写法都可以，但用别名更清晰。

## 题目 3：计算每个用户的总消费金额

> **题目**：交易表 `transactions` 包含 `user_id`、`amount`、`transaction_date`。请查询每个用户的总消费金额，只返回总消费超过 1000 元的用户，按总消费降序排列。

```sql
SELECT user_id,
       SUM(amount) AS total_amount
FROM transactions
GROUP BY user_id
HAVING SUM(amount) > 1000
ORDER BY total_amount DESC;
```

> **核心区别**：`WHERE` 在分组前过滤行，`HAVING` 在分组后过滤组。这是面试官经常追问的知识点。比如"只看 2025 年的交易"用 `WHERE`，"只看总额超过 1000 的用户"用 `HAVING`。

## 题目 4：查询从未下单的用户

> **题目**：用户表 `users` 包含 `user_id`、`name`、`register_date`。订单表 `orders` 包含 `order_id`、`user_id`、`order_date`。请找出注册了但从未下过单的用户。

**方法一：LEFT JOIN + IS NULL**

```sql
SELECT u.user_id, u.name
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
WHERE o.user_id IS NULL;
```

**方法二：NOT EXISTS**

```sql
SELECT u.user_id, u.name
FROM users u
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.user_id
);
```

**方法三：NOT IN**

```sql
SELECT user_id, name
FROM users
WHERE user_id NOT IN (
    SELECT DISTINCT user_id
    FROM orders
    WHERE user_id IS NOT NULL
);
```

> **面试加分**：如果你能写出多种方法并比较它们的优劣（LEFT JOIN 直观、NOT EXISTS 在大表上性能通常更好、NOT IN 需要注意 NULL 值陷阱），面试官会对你的 SQL 理解深度印象深刻。

## 题目 5：用 CASE WHEN 做分类统计

> **题目**：学生成绩表 `scores` 包含 `student_id`、`subject`、`score`。请将成绩分为三档：90 分及以上为"优秀"、60-89 分为"及格"、60 分以下为"不及格"，并统计每个档次的人数。

```sql
SELECT
    CASE
        WHEN score >= 90 THEN '优秀'
        WHEN score >= 60 THEN '及格'
        ELSE '不及格'
    END AS grade_level,
    COUNT(*) AS student_count
FROM scores
GROUP BY
    CASE
        WHEN score >= 90 THEN '优秀'
        WHEN score >= 60 THEN '及格'
        ELSE '不及格'
    END
ORDER BY student_count DESC;
```

> **技巧**：`CASE WHEN` 按顺序匹配，命中第一个条件就停止。所以 `WHEN score >= 60` 实际上匹配的是 60-89 分（因为 >= 90 的已经被前一个条件捕获了）。

## 题目 6：日期相关查询

> **题目**：登录日志表 `login_log` 包含 `user_id`、`login_time`（datetime 类型）。请查询每天的登录用户数（去重），并找出登录用户数最多的那一天。

```sql
SELECT DATE(login_time) AS login_date,
       COUNT(DISTINCT user_id) AS unique_users
FROM login_log
GROUP BY DATE(login_time)
ORDER BY unique_users DESC
LIMIT 1;
```

> **注意**：`COUNT(DISTINCT user_id)` 表示去重后的用户数，如果一个用户一天登录多次只算一个。`DATE()` 函数将 datetime 截取为日期部分（MySQL 语法，其他数据库可能用 `CAST(login_time AS DATE)` 或 `login_time::date`）。

## 题目 7：百分比计算

> **题目**：商品表 `products` 包含 `product_id`、`category`、`price`。请计算每个品类的商品数量占总商品数量的百分比。

```sql
SELECT category,
       COUNT(*) AS product_count,
       ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM products), 2) AS percentage
FROM products
GROUP BY category
ORDER BY percentage DESC;
```

**考察点**：子查询用于获取总数、`ROUND` 函数控制小数位、`100.0`（而非 `100`）避免整数除法导致结果为 0。

## 题目 8：查询第 N 高的值

> **题目**：员工表 `employees` 包含 `emp_id`、`name`、`salary`。请查询薪资第二高的员工信息。如果有多人并列第二高，全部返回。

```sql
SELECT emp_id, name, salary
FROM employees
WHERE salary = (
    SELECT DISTINCT salary
    FROM employees
    ORDER BY salary DESC
    LIMIT 1 OFFSET 1
);
```

> **思路解析**：先用子查询找到第二高的薪资值（去重后取第二个），再用外层查询找到所有拿这个薪资的员工。`OFFSET 1` 跳过第一条记录（最高薪资），取第二条。

## 面试现场的答题技巧

1. **先理清题意**：不要拿到题就写。花 30 秒确认表结构、字段含义、预期输出格式
2. **说出思路再写代码**："我先从 XX 表筛选 YY 条件的数据，然后按 ZZ 分组统计"
3. **检查边界情况**：
   - NULL 值会不会影响结果？
   - 如果某个分组没有数据会怎样？
   - 日期范围的边界是否正确？
4. **注意代码风格**：关键字大写、合理缩进、使用别名——这些细节反映你的专业度
5. **完成后做验证**：用一个小例子在脑中跑一遍，确认逻辑正确

## 练习建议

| 平台 | 推荐题目数量 | 特点 |
|------|------------|------|
| LeetCode（数据库专题） | Easy 全部 | 题目质量高、有在线运行环境 |
| HackerRank（SQL） | Easy 全部 | 分类清晰、适合系统练习 |
| StrataScratch | 标注为 Easy 的题目 | 真实公司面试题 |

> **刷题节奏**：面试前 2-4 周，每天做 2-3 道简单题保持手感。简单题不需要花太多时间，重点是**准确率和速度**——面试中简单题应该在 5 分钟内完成。

## 本篇小结

- 简单题是面试中的送分题，必须确保 100% 正确率
- 核心语法：`SELECT`、`WHERE`、`GROUP BY`、`HAVING`、`ORDER BY`、`LIMIT`
- 常用技巧：`DISTINCT` 去重、`CASE WHEN` 分类、`COUNT`/`SUM`/`AVG` 聚合
- 注意 NULL 值、日期边界、整数除法等常见陷阱
- 养成"先说思路再写代码"的习惯

准备好挑战更高难度了吗？进入 [SQL 中等题精讲](../02-sql-medium/)。
