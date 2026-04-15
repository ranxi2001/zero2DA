---
title: "SQL 面试高频题型"
layout: default
description: "精讲 SQL 面试中最常考的题型：连续登录、Top N、留存率、同比环比、行转列、累计计算，附完整题目与解题思路。"
eyebrow: "SQL · 第 4 篇"
---

# SQL 面试高频题型

> **本篇目标**：通过 8 道经典 SQL 面试题，掌握数据分析师面试中最常见的题型套路。每道题都包含题目描述、解题思路和完整代码。

## 面试 SQL 考什么

数据分析师面试中的 SQL 考核通常有两种形式：

1. **在线笔试**：在 HackerRank、LeetCode 或公司自研平台上限时写 SQL，通常 3-5 道题，45-60 分钟
2. **现场白板**：面试官口述需求，你在白板或文档上写 SQL，边写边讲思路

不管哪种形式，考察的核心知识点高度集中在以下几类：

| 题型 | 考察知识点 | 出现频率 |
|------|-----------|:--------:|
| 连续 N 天登录/活跃 | 窗口函数、日期运算、自连接 | 极高 |
| Top N 问题 | DENSE_RANK、PARTITION BY | 极高 |
| 留存率计算 | LEFT JOIN、日期差、CASE WHEN | 高 |
| 同比环比 | LAG/LEAD、日期函数 | 高 |
| 行转列 / 列转行 | CASE WHEN + 聚合、UNION ALL | 中高 |
| 累计计算 | SUM OVER、窗口框架 | 中高 |
| 去重与排重 | ROW_NUMBER、GROUP BY | 中 |
| 中位数 | ROW_NUMBER、NTILE 或 PERCENTILE | 中 |

## 题目一：连续登录 N 天的用户

### 题目

有一张用户登录记录表 `user_logins(user_id, login_date)`，同一用户同一天可能有多条记录。找出**连续登录至少 3 天**的用户及其最长连续登录天数。

### 解题思路

1. 先对每个用户的登录日期去重
2. 用 `ROW_NUMBER()` 给每个用户的登录日期编号
3. 用 `login_date - ROW_NUMBER` 得到一个"分组标记"——连续日期减去连续编号会得到相同的值
4. 按分组标记统计连续天数

### 完整代码

```sql
WITH distinct_logins AS (
    -- 第一步：去重
    SELECT DISTINCT user_id, login_date
    FROM user_logins
),
numbered AS (
    -- 第二步：编号
    SELECT
        user_id,
        login_date,
        ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY login_date
        ) AS rn
    FROM distinct_logins
),
grouped AS (
    -- 第三步：计算分组标记
    SELECT
        user_id,
        login_date,
        DATE_SUB(login_date, INTERVAL rn DAY) AS grp
    FROM numbered
),
streaks AS (
    -- 第四步：统计每段连续登录的天数
    SELECT
        user_id,
        grp,
        COUNT(*)       AS consecutive_days,
        MIN(login_date) AS start_date,
        MAX(login_date) AS end_date
    FROM grouped
    GROUP BY user_id, grp
    HAVING COUNT(*) >= 3
)
-- 最终结果：每个用户的最长连续登录天数
SELECT
    user_id,
    MAX(consecutive_days) AS 最长连续登录天数
FROM streaks
GROUP BY user_id
ORDER BY 最长连续登录天数 DESC;
```

> **核心技巧**：`login_date - ROW_NUMBER() = 常数` 这个公式是连续日期类问题的万能钥匙。如果日期连续，那么日期减去行号得到的值一定相同。

## 题目二：每个部门工资 Top 3 的员工

### 题目

有一张员工表 `employees(id, name, department, salary)`。找出每个部门工资最高的前 3 名员工。如果并列第 3，则全部输出。

### 完整代码

```sql
WITH ranked AS (
    SELECT
        id,
        name,
        department,
        salary,
        DENSE_RANK() OVER (
            PARTITION BY department
            ORDER BY salary DESC
        ) AS rk
    FROM employees
)
SELECT department, name, salary, rk
FROM ranked
WHERE rk <= 3
ORDER BY department, rk, name;
```

> **为什么用 DENSE_RANK 而不是 ROW_NUMBER**？题目要求"如果并列第 3 则全部输出"。ROW_NUMBER 会给并列的人不同编号（可能一个排第 3、另一个排第 4），导致漏选。DENSE_RANK 则会给并列的人相同排名。

## 题目三：计算用户次日留存率

### 题目

有一张用户行为表 `user_actions(user_id, action_date)`。计算每天新用户的**次日留存率**。新用户定义为该用户第一次出现在表中的那天。

### 解题思路

1. 找出每个用户的首次活跃日期（即"新用户日期"）
2. 看该用户在首次活跃日期的第二天是否再次出现
3. 按首次活跃日期分组，计算留存率

### 完整代码

```sql
WITH first_day AS (
    -- 每个用户的首次活跃日
    SELECT
        user_id,
        MIN(action_date) AS first_date
    FROM user_actions
    GROUP BY user_id
),
retention AS (
    SELECT
        f.user_id,
        f.first_date,
        CASE
            WHEN EXISTS (
                SELECT 1 FROM user_actions a
                WHERE a.user_id = f.user_id
                  AND a.action_date = DATE_ADD(f.first_date, INTERVAL 1 DAY)
            ) THEN 1
            ELSE 0
        END AS is_retained
    FROM first_day f
)
SELECT
    first_date              AS 日期,
    COUNT(*)                AS 新用户数,
    SUM(is_retained)        AS 次日留存用户数,
    ROUND(
        SUM(is_retained) / COUNT(*) * 100, 2
    )                       AS 次日留存率百分比
FROM retention
GROUP BY first_date
ORDER BY first_date;
```

### 扩展：N 日留存率

```sql
WITH first_day AS (
    SELECT user_id, MIN(action_date) AS first_date
    FROM user_actions
    GROUP BY user_id
)
SELECT
    f.first_date,
    COUNT(DISTINCT f.user_id) AS 新用户数,
    COUNT(DISTINCT CASE
        WHEN DATEDIFF(a.action_date, f.first_date) = 1 THEN f.user_id
    END) AS 次日留存,
    COUNT(DISTINCT CASE
        WHEN DATEDIFF(a.action_date, f.first_date) = 3 THEN f.user_id
    END) AS 三日留存,
    COUNT(DISTINCT CASE
        WHEN DATEDIFF(a.action_date, f.first_date) = 7 THEN f.user_id
    END) AS 七日留存,
    COUNT(DISTINCT CASE
        WHEN DATEDIFF(a.action_date, f.first_date) = 30 THEN f.user_id
    END) AS 三十日留存
FROM first_day f
LEFT JOIN user_actions a
    ON f.user_id = a.user_id
    AND a.action_date >= f.first_date
GROUP BY f.first_date
ORDER BY f.first_date;
```

## 题目四：计算月环比与同比

### 题目

有一张销售表 `sales(sale_date, amount)`。计算每月的总销售额、环比增长率（与上月相比）、同比增长率（与去年同月相比）。

### 完整代码

```sql
WITH monthly AS (
    SELECT
        DATE_FORMAT(sale_date, '%Y-%m') AS month,
        YEAR(sale_date)                 AS yr,
        MONTH(sale_date)                AS mn,
        SUM(amount)                     AS revenue
    FROM sales
    GROUP BY DATE_FORMAT(sale_date, '%Y-%m'),
             YEAR(sale_date), MONTH(sale_date)
)
SELECT
    month                                                       AS 月份,
    revenue                                                     AS 当月销售额,
    LAG(revenue, 1)  OVER (ORDER BY yr, mn)                     AS 上月销售额,
    LAG(revenue, 12) OVER (ORDER BY yr, mn)                     AS 去年同月销售额,
    CASE
        WHEN LAG(revenue, 1) OVER (ORDER BY yr, mn) IS NOT NULL
        THEN ROUND(
            (revenue - LAG(revenue, 1) OVER (ORDER BY yr, mn))
            / LAG(revenue, 1) OVER (ORDER BY yr, mn) * 100, 2)
    END AS 环比增长率,
    CASE
        WHEN LAG(revenue, 12) OVER (ORDER BY yr, mn) IS NOT NULL
        THEN ROUND(
            (revenue - LAG(revenue, 12) OVER (ORDER BY yr, mn))
            / LAG(revenue, 12) OVER (ORDER BY yr, mn) * 100, 2)
    END AS 同比增长率
FROM monthly
ORDER BY month;
```

## 题目五：行转列——统计每个学生各科成绩

### 题目

有一张成绩表 `scores(student_name, subject, score)`，每行是一个学生一门课的成绩。将其转换为每行一个学生、每列一门课的宽表。

### 完整代码

```sql
SELECT
    student_name                                        AS 学生,
    MAX(CASE WHEN subject = '语文' THEN score END)      AS 语文,
    MAX(CASE WHEN subject = '数学' THEN score END)      AS 数学,
    MAX(CASE WHEN subject = '英语' THEN score END)      AS 英语,
    SUM(score)                                          AS 总分,
    ROUND(AVG(score), 1)                                AS 平均分
FROM scores
GROUP BY student_name
ORDER BY 总分 DESC;
```

> **行转列套路**：`MAX(CASE WHEN 条件 THEN 值 END)` + `GROUP BY`。这是面试和实际工作中最常用的行转列方式。

### 反向操作：列转行

```sql
-- 假设有宽表 wide_scores(student_name, chinese, math, english)
SELECT student_name, '语文' AS subject, chinese AS score FROM wide_scores
UNION ALL
SELECT student_name, '数学', math FROM wide_scores
UNION ALL
SELECT student_name, '英语', english FROM wide_scores
ORDER BY student_name, subject;
```

## 题目六：累计销售额与达标日期

### 题目

有一张日销售表 `daily_sales(sale_date, amount)`。找出累计销售额首次达到 100 万的日期。

### 完整代码

```sql
WITH running AS (
    SELECT
        sale_date,
        amount,
        SUM(amount) OVER (ORDER BY sale_date) AS cumulative_amount
    FROM daily_sales
)
SELECT sale_date AS 首次达标日期, cumulative_amount AS 累计销售额
FROM running
WHERE cumulative_amount >= 1000000
ORDER BY sale_date
LIMIT 1;
```

## 题目七：删除重复数据，保留最新的一条

### 题目

表 `contacts(id, email, created_at)` 中有重复的 email。删除重复记录，每个 email 只保留 `created_at` 最新的那条。

### 完整代码

```sql
-- 先找出要保留的行
WITH keep AS (
    SELECT id
    FROM (
        SELECT
            id,
            ROW_NUMBER() OVER (
                PARTITION BY email
                ORDER BY created_at DESC
            ) AS rn
        FROM contacts
    ) t
    WHERE rn = 1
)
-- 删除不在保留名单中的行
DELETE FROM contacts
WHERE id NOT IN (SELECT id FROM keep);
```

> **面试变体**：如果题目只是让你"查出每个 email 最新的记录"（不需要真正删除），那就把 DELETE 换成 SELECT，WHERE rn = 1 即可。

## 题目八：中位数计算

### 题目

计算 `orders` 表中已完成订单金额的中位数。

### 解题思路

中位数的定义：将数据从小到大排序后，位于正中间的值（偶数个时取中间两个的平均值）。

### 完整代码

```sql
-- 方法一：利用 ROW_NUMBER 和总行数
WITH ordered AS (
    SELECT
        total_amount,
        ROW_NUMBER() OVER (ORDER BY total_amount) AS rn,
        COUNT(*) OVER () AS total_count
    FROM orders
    WHERE status = 'completed'
)
SELECT
    ROUND(AVG(total_amount), 2) AS 中位数
FROM ordered
WHERE rn IN (
    FLOOR((total_count + 1) / 2),
    CEIL((total_count + 1) / 2)
);

-- 方法二：利用 NTILE（更简洁但不够精确）
-- 适用于快速估算
WITH bucketed AS (
    SELECT
        total_amount,
        NTILE(2) OVER (ORDER BY total_amount) AS half
    FROM orders
    WHERE status = 'completed'
)
SELECT
    ROUND(
        (MAX(CASE WHEN half = 1 THEN total_amount END) +
         MIN(CASE WHEN half = 2 THEN total_amount END)) / 2
    , 2) AS 中位数近似值
FROM bucketed;
```

## 面试答题框架

在面试中写 SQL 时，建议遵循以下步骤：

1. **确认需求**：复述题目，确认边界条件（是否需要去重？NULL 怎么处理？时间范围？）
2. **说思路**：先用中文描述你打算分几步解决，不要直接开始写代码
3. **分步写 CTE**：每个 CTE 完成一个小任务，边写边解释
4. **验证结果**：写完后口头检查边界情况（空表？只有一行？有 NULL？）
5. **讨论优化**：如果面试官问性能，讨论索引、执行计划、大数据量下的替代方案

### 常见追问及应对

| 追问 | 应对方向 |
|------|---------|
| "如果数据量有 10 亿行怎么办？" | 讨论分区表、索引、提前聚合、用 Spark SQL 替代 |
| "RANK 和 DENSE_RANK 有什么区别？" | 画表说明并列时的编号差异 |
| "NOT IN 和 NOT EXISTS 有什么区别？" | NOT IN 遇到 NULL 会导致结果为空；NOT EXISTS 不受此影响 |
| "CTE 和临时表有什么区别？" | CTE 是查询级别的，不物化；临时表会写入 tempdb |
| "你的查询能不能用 JOIN 替代子查询？" | 展示等价的 JOIN 写法并比较可读性 |

## 刷题建议

| 平台 | 推荐题量 | 说明 |
|------|:--------:|------|
| LeetCode Database | 50-80 题 | 面试最接近的题库，按 Easy → Medium → Hard 刷 |
| HackerRank SQL | 30-40 题 | 基础练习，适合巩固语法 |
| StrataScratch | 20-30 题 | 包含真实公司面试题（Google、Amazon、Meta 等） |

> **下一篇预告**：[数据库基础知识](../05-database-fundamentals/index.html) —— 了解关系模型、主键外键、索引、范式等数据库核心概念，让你在面试中展现更深厚的技术功底。
