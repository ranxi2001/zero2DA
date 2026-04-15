---
title: SQL 困难题精讲
layout: default
description: 数据分析师面试 SQL 困难题精讲——复杂窗口函数、递归 CTE、间隔分析，挑战高级岗位面试。
eyebrow: "DA 面试 · SQL 面试"
---

# SQL 困难题精讲

困难题是 Senior DA 或偏技术方向的数据分析师面试中的**区分题**。它们通常涉及复杂的窗口函数组合、递归查询、多步骤业务逻辑等。能解出这些题，说明你的 SQL 能力达到了可以独立处理任何业务数据需求的水平。

## 题型概览

| 题型 | 考察点 | 难度特征 |
|------|--------|---------|
| 复杂窗口函数组合 | 多个窗口函数嵌套使用 | 需要清晰的逻辑拆分 |
| 递归 CTE | 层级数据、序列生成 | 需要理解递归终止条件 |
| 间隔合并问题 | 重叠时间区间的合并 | 需要巧妙的排序和比较 |
| 漏斗转化分析 | 多步骤有序事件序列 | 需要处理时间顺序和条件过滤 |
| 中位数计算 | 在 SQL 中实现中位数 | 不同数据库实现方式差异大 |
| 会话划分 | 基于时间间隔划分用户会话 | 窗口函数 + 条件累计 |

## 题目 1：中位数薪资

> **题目**：员工表 `employees` 包含 `department`、`salary`。请计算每个部门的薪资中位数。

```sql
WITH ranked AS (
    SELECT department,
           salary,
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary) AS rn,
           COUNT(*) OVER (PARTITION BY department) AS cnt
    FROM employees
)
SELECT department,
       AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (FLOOR((cnt + 1) / 2.0), CEIL((cnt + 1) / 2.0))
GROUP BY department;
```

> **思路解析**：中位数的位置取决于总数是奇数还是偶数。奇数时中位数在第 (n+1)/2 位；偶数时中位数是第 n/2 和第 n/2+1 位的平均值。`FLOOR` 和 `CEIL` 统一处理了这两种情况——奇数时两者相等，偶数时分别取两个中间位置。

## 题目 2：漏斗转化分析

> **题目**：用户事件表 `events` 包含 `user_id`、`event_type`（取值为 'view'、'cart'、'checkout'、'purchase'）、`event_time`。请计算从浏览到购买的四步漏斗转化率，要求每个用户在同一天内按顺序完成（后一步的时间必须晚于前一步）。

```sql
WITH funnel AS (
    SELECT
        e1.user_id,
        DATE(e1.event_time) AS funnel_date,
        MIN(e1.event_time) AS view_time,
        MIN(e2.event_time) AS cart_time,
        MIN(e3.event_time) AS checkout_time,
        MIN(e4.event_time) AS purchase_time
    FROM events e1
    LEFT JOIN events e2
        ON e1.user_id = e2.user_id
        AND DATE(e1.event_time) = DATE(e2.event_time)
        AND e2.event_type = 'cart'
        AND e2.event_time > e1.event_time
    LEFT JOIN events e3
        ON e1.user_id = e3.user_id
        AND DATE(e1.event_time) = DATE(e3.event_time)
        AND e3.event_type = 'checkout'
        AND e3.event_time > e2.event_time
    LEFT JOIN events e4
        ON e1.user_id = e4.user_id
        AND DATE(e1.event_time) = DATE(e4.event_time)
        AND e4.event_type = 'purchase'
        AND e4.event_time > e3.event_time
    WHERE e1.event_type = 'view'
    GROUP BY e1.user_id, DATE(e1.event_time)
)
SELECT
    COUNT(*) AS total_view_users,
    SUM(CASE WHEN cart_time IS NOT NULL THEN 1 ELSE 0 END) AS cart_users,
    SUM(CASE WHEN checkout_time IS NOT NULL THEN 1 ELSE 0 END) AS checkout_users,
    SUM(CASE WHEN purchase_time IS NOT NULL THEN 1 ELSE 0 END) AS purchase_users,
    ROUND(SUM(CASE WHEN cart_time IS NOT NULL THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS view_to_cart_pct,
    ROUND(SUM(CASE WHEN checkout_time IS NOT NULL THEN 1 ELSE 0 END) * 100.0
          / NULLIF(SUM(CASE WHEN cart_time IS NOT NULL THEN 1 ELSE 0 END), 0), 2) AS cart_to_checkout_pct,
    ROUND(SUM(CASE WHEN purchase_time IS NOT NULL THEN 1 ELSE 0 END) * 100.0
          / NULLIF(SUM(CASE WHEN checkout_time IS NOT NULL THEN 1 ELSE 0 END), 0), 2) AS checkout_to_purchase_pct
FROM funnel;
```

> **关键点**：
> - 用 `LEFT JOIN` 而非 `INNER JOIN`，确保即使用户没有完成后续步骤也被统计
> - `event_time` 的先后顺序用 `>` 保证逻辑正确
> - `NULLIF(..., 0)` 防止分母为零导致除法报错

## 题目 3：会话划分（Sessionization）

> **题目**：用户点击流表 `clickstream` 包含 `user_id`、`click_time`（datetime）。如果两次点击间隔超过 30 分钟，则认为是一个新的会话。请给每次点击标记所属的会话编号，并统计每个用户的会话数量和平均会话时长。

```sql
WITH click_with_gap AS (
    SELECT user_id,
           click_time,
           LAG(click_time) OVER (PARTITION BY user_id ORDER BY click_time) AS prev_click_time,
           CASE
               WHEN TIMESTAMPDIFF(MINUTE, LAG(click_time) OVER (PARTITION BY user_id ORDER BY click_time), click_time) > 30
                    OR LAG(click_time) OVER (PARTITION BY user_id ORDER BY click_time) IS NULL
               THEN 1
               ELSE 0
           END AS is_new_session
    FROM clickstream
),
sessions AS (
    SELECT user_id,
           click_time,
           SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY click_time) AS session_id
    FROM click_with_gap
),
session_stats AS (
    SELECT user_id,
           session_id,
           MIN(click_time) AS session_start,
           MAX(click_time) AS session_end,
           TIMESTAMPDIFF(MINUTE, MIN(click_time), MAX(click_time)) AS session_duration_min,
           COUNT(*) AS clicks_in_session
    FROM sessions
    GROUP BY user_id, session_id
)
SELECT user_id,
       COUNT(*) AS total_sessions,
       ROUND(AVG(session_duration_min), 1) AS avg_session_duration_min,
       ROUND(AVG(clicks_in_session), 1) AS avg_clicks_per_session
FROM session_stats
GROUP BY user_id
ORDER BY total_sessions DESC;
```

> **经典模式**：会话划分是一个三步走模式：1) 用 `LAG` 计算时间间隔；2) 用 `CASE WHEN` 标记新会话开始；3) 用 `SUM OVER` 累计求和得到会话编号。这个模式在用户行为分析中非常常用。

## 题目 4：重叠时间区间合并

> **题目**：项目表 `projects` 包含 `project_id`、`start_date`、`end_date`。有些项目的时间区间有重叠。请将所有重叠的区间合并，输出合并后的时间区间。

```sql
WITH ordered_projects AS (
    SELECT project_id,
           start_date,
           end_date,
           MAX(end_date) OVER (
               ORDER BY start_date, end_date
               ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING
           ) AS max_prev_end
    FROM projects
),
grouped AS (
    SELECT *,
           SUM(CASE WHEN start_date > max_prev_end OR max_prev_end IS NULL THEN 1 ELSE 0 END)
               OVER (ORDER BY start_date, end_date) AS grp
    FROM ordered_projects
)
SELECT grp,
       MIN(start_date) AS merged_start,
       MAX(end_date) AS merged_end
FROM grouped
GROUP BY grp
ORDER BY merged_start;
```

> **思路**：按开始日期排序后，如果当前项目的开始日期晚于之前所有项目的最大结束日期，则开启一个新的合并组。否则归入当前组。

## 题目 5：递归 CTE——组织架构层级查询

> **题目**：员工表 `employees` 包含 `emp_id`、`name`、`manager_id`（上级的 emp_id，CEO 的 manager_id 为 NULL）。请查询每个员工的层级（CEO 为第 1 级）和完整的汇报链。

```sql
WITH RECURSIVE org_tree AS (
    -- 基础情况：CEO（没有上级）
    SELECT emp_id,
           name,
           manager_id,
           1 AS level,
           CAST(name AS CHAR(1000)) AS report_chain
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- 递归步骤：每一级的下属
    SELECT e.emp_id,
           e.name,
           e.manager_id,
           t.level + 1 AS level,
           CONCAT(t.report_chain, ' > ', e.name) AS report_chain
    FROM employees e
    INNER JOIN org_tree t ON e.manager_id = t.emp_id
)
SELECT emp_id, name, level, report_chain
FROM org_tree
ORDER BY level, name;
```

> **递归 CTE 的结构**：
> 1. **基础部分**（Anchor）：定义递归的起点（通常是根节点或初始条件）
> 2. **递归部分**：引用 CTE 自身，每次迭代扩展一层
> 3. **终止条件**：当递归部分不再产生新行时自动停止
>
> 注意：大多数数据库对递归深度有限制（MySQL 默认 1000 层），处理深层嵌套数据时需要留意。

## 题目 6：年度用户留存矩阵

> **题目**：用户活动表 `user_activity` 包含 `user_id`、`activity_date`。请生成一张用户留存矩阵：按注册月份分群（Cohort），计算注册后每个月的留存率。

```sql
WITH user_cohort AS (
    -- 确定每个用户的注册月份
    SELECT user_id,
           DATE_FORMAT(MIN(activity_date), '%Y-%m') AS cohort_month
    FROM user_activity
    GROUP BY user_id
),
monthly_activity AS (
    -- 计算每个用户每月是否活跃
    SELECT DISTINCT
           a.user_id,
           c.cohort_month,
           DATE_FORMAT(a.activity_date, '%Y-%m') AS activity_month,
           PERIOD_DIFF(
               DATE_FORMAT(a.activity_date, '%Y%m'),
               DATE_FORMAT(STR_TO_DATE(CONCAT(c.cohort_month, '-01'), '%Y-%m-%d'), '%Y%m')
           ) AS months_since_register
    FROM user_activity a
    INNER JOIN user_cohort c ON a.user_id = c.user_id
),
cohort_size AS (
    SELECT cohort_month, COUNT(DISTINCT user_id) AS total_users
    FROM user_cohort
    GROUP BY cohort_month
)
SELECT m.cohort_month,
       cs.total_users,
       m.months_since_register,
       COUNT(DISTINCT m.user_id) AS active_users,
       ROUND(COUNT(DISTINCT m.user_id) * 100.0 / cs.total_users, 2) AS retention_rate
FROM monthly_activity m
INNER JOIN cohort_size cs ON m.cohort_month = cs.cohort_month
GROUP BY m.cohort_month, cs.total_users, m.months_since_register
ORDER BY m.cohort_month, m.months_since_register;
```

> **实际应用**：这道题的 SQL 输出可以直接导入 Excel 或 BI 工具生成留存热力图。在面试中如果能提到"这个结果可以用 Tableau 做成 Cohort 热力图来可视化"，会展示你的完整分析思维。

## 题目 7：移动平均异常检测

> **题目**：每日指标表 `daily_metrics` 包含 `metric_date`、`metric_value`。请找出指标值偏离过去 7 天移动平均超过 2 个标准差的异常日期。

```sql
WITH stats AS (
    SELECT metric_date,
           metric_value,
           AVG(metric_value) OVER (
               ORDER BY metric_date
               ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING
           ) AS rolling_avg,
           STDDEV(metric_value) OVER (
               ORDER BY metric_date
               ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING
           ) AS rolling_std
    FROM daily_metrics
)
SELECT metric_date,
       metric_value,
       ROUND(rolling_avg, 2) AS rolling_avg,
       ROUND(rolling_std, 2) AS rolling_std,
       ROUND((metric_value - rolling_avg) / NULLIF(rolling_std, 0), 2) AS z_score,
       CASE
           WHEN ABS(metric_value - rolling_avg) > 2 * rolling_std AND rolling_std > 0
           THEN '异常'
           ELSE '正常'
       END AS status
FROM stats
WHERE rolling_avg IS NOT NULL
ORDER BY metric_date;
```

> **统计知识**：Z-score（标准分数）= (观测值 - 均值) / 标准差。超过 2 个标准差的数据点在正态分布中出现的概率不到 5%，可以标记为异常。注意用 `ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING` 排除当天数据，避免"用自己检测自己"。

## 困难题的答题策略

1. **不要急于写代码**：花 2-3 分钟在纸上画出数据转换的每一步
2. **用 CTE 拆分步骤**：每个 CTE 只做一件事，命名清晰，面试官能跟上你的思路
3. **主动说出复杂度**：提到"这个自关联的复杂度是 O(n^2)，在大表上可能需要优化"，展示你的工程意识
4. **处理边界情况**：NULL 值、空结果、第一行/最后一行的窗口函数行为
5. **时间管理**：困难题控制在 15-20 分钟。如果超时，先说出完整思路，再写核心部分

## 本篇小结

- 困难题考察的不仅是语法，更是**分析问题和拆解步骤的能力**
- 递归 CTE 解决层级数据问题，关键是定义好基础部分和终止条件
- 会话划分、区间合并、漏斗分析都有经典的 SQL 模式，值得反复练习
- 窗口帧（`ROWS BETWEEN`）的精确控制是困难题的常见考点
- 面试中即使没写出完整代码，清晰的思路表达也能拿到大部分分数

SQL 面试系列到此结束。接下来学习 [Case 分析框架](../../case-study/01-case-framework/)，掌握数据分析师面试中另一个重要环节。
