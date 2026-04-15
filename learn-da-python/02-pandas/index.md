---
title: "Pandas 数据处理"
layout: default
description: "Pandas 是数据分析师的核心工具——学会 DataFrame 的创建与操作、数据读取与清洗、分组聚合、合并与透视，用 Python 高效处理百万级数据。"
eyebrow: "Python · 第 2 篇"
---

# Pandas 数据处理

> **Pandas 是什么？** 如果说 Python 是数据分析师的瑞士军刀，那 Pandas 就是其中最锋利的那把刀。它提供了 DataFrame（数据表）这一核心数据结构，让你用几行代码完成 Excel 中几十步的操作。名字来自 **Pan**el **Da**ta（面板数据），是金融和计量经济学中的术语。

## 快速入门：5 分钟上手

### 导入 Pandas

```python
import pandas as pd
import numpy as np

# 约定俗成的缩写：pd 代表 pandas，np 代表 numpy
```

### 创建 DataFrame

```python
# 方式一：从字典创建（最常用）
data = {
    "日期": ["2024-01-01", "2024-01-01", "2024-01-02", "2024-01-02", "2024-01-03"],
    "渠道": ["微信", "抖音", "微信", "抖音", "微信"],
    "UV": [12000, 8500, 11500, 9200, 13000],
    "订单数": [360, 255, 345, 276, 390],
    "GMV": [89000, 63000, 86000, 68000, 96000],
}
df = pd.DataFrame(data)
print(df)
```

输出：

```
          日期   渠道     UV  订单数    GMV
0  2024-01-01   微信  12000   360  89000
1  2024-01-01   抖音   8500   255  63000
2  2024-01-02   微信  11500   345  86000
3  2024-01-02   抖音   9200   276  68000
4  2024-01-03   微信  13000   390  96000
```

### 读取外部数据

```python
# 读取 CSV 文件
df = pd.read_csv("daily_report.csv")

# 读取 Excel 文件
df = pd.read_excel("monthly_data.xlsx", sheet_name="Sheet1")

# 读取 SQL 查询结果
import sqlite3
conn = sqlite3.connect("database.db")
df = pd.read_sql("SELECT * FROM orders WHERE date >= '2024-01-01'", conn)

# 常用参数
df = pd.read_csv(
    "data.csv",
    encoding="utf-8",       # 编码格式
    parse_dates=["日期"],    # 自动解析日期列
    dtype={"用户ID": str},   # 指定列类型
    na_values=["N/A", ""],   # 自定义缺失值标记
)
```

## 数据探索：拿到数据第一步

每次拿到一个新数据集，先做这 5 步：

```python
# 1. 查看前几行
df.head()

# 2. 查看数据形状（行数 × 列数）
print(f"数据量: {df.shape[0]} 行, {df.shape[1]} 列")

# 3. 查看列名和数据类型
df.info()

# 4. 查看数值列的统计摘要
df.describe()

# 5. 查看缺失值情况
print(df.isnull().sum())
```

> **业务场景**：产品经理给了你一份用户行为日志，说"帮我看看用户活跃情况"。第一步不是着急做分析，而是先跑上面 5 行代码，搞清楚数据长什么样、有多少行、有没有脏数据。

## 数据选择与筛选

### 选择列

```python
# 选择单列（返回 Series）
uv_series = df["UV"]

# 选择多列（返回 DataFrame）
subset = df[["日期", "渠道", "GMV"]]

# 排除某些列
df_no_uv = df.drop(columns=["UV"])
```

### 条件筛选

```python
# 筛选微信渠道的数据
wechat = df[df["渠道"] == "微信"]

# 多条件筛选：GMV 大于 80000 且渠道为微信
high_gmv_wechat = df[(df["GMV"] > 80000) & (df["渠道"] == "微信")]

# 使用 isin 筛选多个值
target_channels = df[df["渠道"].isin(["微信", "抖音", "小红书"])]

# 字符串包含
df[df["渠道"].str.contains("微")]

# 使用 query 方法（更像 SQL 的写法）
result = df.query("GMV > 80000 and 渠道 == '微信'")
```

### 使用 loc 和 iloc

```python
# loc：基于标签选择
df.loc[0:2, ["日期", "GMV"]]  # 前 3 行，指定列

# iloc：基于位置选择（类似数组索引）
df.iloc[0:3, [0, 4]]  # 前 3 行，第 1 和第 5 列

# loc 配合条件筛选
df.loc[df["GMV"] > 80000, "渠道"]
```

## 数据清洗

### 处理缺失值

```python
# 模拟含缺失值的数据
df_dirty = pd.DataFrame({
    "user_id": ["U001", "U002", "U003", "U004", "U005"],
    "age": [25, np.nan, 32, 28, np.nan],
    "city": ["北京", "上海", None, "广州", "深圳"],
    "spend": [1200, 800, np.nan, 2100, 560],
})

# 查看缺失值
print(df_dirty.isnull().sum())
# age      2
# city     1
# spend    1

# 删除含缺失值的行
df_clean = df_dirty.dropna()

# 只看特定列有缺失值的行
df_dirty.dropna(subset=["city"])

# 填充缺失值
df_dirty["age"].fillna(df_dirty["age"].median(), inplace=True)  # 用中位数填充
df_dirty["city"].fillna("未知", inplace=True)                    # 用固定值填充
df_dirty["spend"].fillna(0, inplace=True)                        # 用 0 填充
```

### 处理重复值

```python
# 检查重复
print(f"重复行数: {df.duplicated().sum()}")

# 查看重复的具体行
df[df.duplicated(keep=False)]

# 删除重复行
df_dedup = df.drop_duplicates()

# 基于特定列去重（保留第一条）
df_dedup = df.drop_duplicates(subset=["user_id", "日期"], keep="first")
```

### 数据类型转换

```python
# 字符串转日期
df["日期"] = pd.to_datetime(df["日期"])

# 提取日期的年、月、周
df["年"] = df["日期"].dt.year
df["月"] = df["日期"].dt.month
df["星期"] = df["日期"].dt.day_name()

# 字符串转数值
df["金额"] = pd.to_numeric(df["金额_str"], errors="coerce")  # 无法转换的变为 NaN
```

### 字符串处理

```python
# 去除前后空格
df["city"] = df["city"].str.strip()

# 统一大小写
df["channel"] = df["channel"].str.lower()

# 替换
df["phone"] = df["phone"].str.replace("-", "")

# 分列
df[["省", "市"]] = df["地区"].str.split("-", expand=True)
```

## 新增计算列

```python
# 计算转化率
df["转化率"] = df["订单数"] / df["UV"] * 100

# 计算客单价
df["客单价"] = df["GMV"] / df["订单数"]

# 使用 apply 做复杂转换
def spending_tier(amount):
    if amount >= 5000:
        return "高"
    elif amount >= 1000:
        return "中"
    else:
        return "低"

df["消费等级"] = df["GMV"].apply(spending_tier)

# 使用 np.where 做二元分类（类似 Excel 的 IF）
df["是否达标"] = np.where(df["转化率"] >= 3.0, "达标", "未达标")
```

## 分组聚合（GroupBy）

分组聚合是数据分析中最高频的操作，相当于 SQL 中的 `GROUP BY`。

```python
# 按渠道分组，计算各指标汇总
channel_summary = df.groupby("渠道").agg(
    总UV=("UV", "sum"),
    总订单=("订单数", "sum"),
    总GMV=("GMV", "sum"),
    平均客单价=("客单价", "mean"),
).reset_index()

print(channel_summary)
```

```python
# 多层分组
monthly_channel = df.groupby(["月", "渠道"]).agg(
    GMV=("GMV", "sum"),
    订单数=("订单数", "sum"),
).reset_index()

# 常用聚合函数
# sum, mean, median, min, max, count, nunique, std, first, last
```

> **业务场景**：运营同事问"上个月各渠道的 ROI 怎么样？" 用 `groupby("渠道")` 一行代码就能算出各渠道的 GMV、订单数、客单价。

## 数据合并

### merge（类似 SQL JOIN）

```python
# 订单表
orders = pd.DataFrame({
    "order_id": ["A001", "A002", "A003"],
    "user_id": ["U01", "U02", "U03"],
    "amount": [299, 158, 520],
})

# 用户表
users = pd.DataFrame({
    "user_id": ["U01", "U02", "U04"],
    "name": ["张三", "李四", "王五"],
    "city": ["北京", "上海", "广州"],
})

# 内连接（默认）——只保留两边都有的
inner = pd.merge(orders, users, on="user_id", how="inner")

# 左连接——保留左表所有数据
left = pd.merge(orders, users, on="user_id", how="left")

# 全外连接
outer = pd.merge(orders, users, on="user_id", how="outer")
```

### concat（纵向/横向拼接）

```python
# 纵向拼接（追加行）——合并多月数据
jan = pd.read_csv("jan_data.csv")
feb = pd.read_csv("feb_data.csv")
mar = pd.read_csv("mar_data.csv")

all_data = pd.concat([jan, feb, mar], ignore_index=True)
```

## 排序与排名

```python
# 按 GMV 降序排列
df_sorted = df.sort_values("GMV", ascending=False)

# 多列排序
df_sorted = df.sort_values(["渠道", "GMV"], ascending=[True, False])

# 排名
df["GMV排名"] = df["GMV"].rank(ascending=False, method="dense")
```

## 透视表

透视表相当于 Excel 的数据透视表，非常适合做交叉分析：

```python
# 渠道 × 日期 的 GMV 透视表
pivot = df.pivot_table(
    values="GMV",
    index="渠道",
    columns="日期",
    aggfunc="sum",
    fill_value=0,
    margins=True,       # 添加汇总行/列
    margins_name="合计"
)
print(pivot)
```

## 实战示例：电商日报分析

把上面学到的操作串起来，模拟一个真实的数据分析流程：

```python
import pandas as pd
import numpy as np

# 1. 读取数据
df = pd.read_csv("ecommerce_daily.csv", parse_dates=["date"])

# 2. 数据清洗
df = df.dropna(subset=["gmv"])           # 删除 GMV 为空的行
df = df[df["gmv"] > 0]                   # 排除异常值
df["channel"] = df["channel"].str.strip() # 去除空格

# 3. 新增计算列
df["conversion_rate"] = df["orders"] / df["uv"] * 100
df["aov"] = df["gmv"] / df["orders"]     # 客单价

# 4. 分组聚合
daily_summary = df.groupby("date").agg(
    total_uv=("uv", "sum"),
    total_orders=("orders", "sum"),
    total_gmv=("gmv", "sum"),
).reset_index()

daily_summary["overall_conv"] = daily_summary["total_orders"] / daily_summary["total_uv"] * 100

# 5. 计算环比
daily_summary["gmv_change"] = daily_summary["total_gmv"].pct_change() * 100

# 6. 输出结果
print("=== 每日汇总 ===")
print(daily_summary.to_string(index=False))

# 7. 导出到 Excel
daily_summary.to_excel("daily_summary_report.xlsx", index=False)
print("报表已导出！")
```

## 常用技巧速查表

| 操作 | 代码 |
|------|------|
| 读取 CSV | `pd.read_csv("file.csv")` |
| 查看前 N 行 | `df.head(n)` |
| 数据形状 | `df.shape` |
| 列数据类型 | `df.dtypes` |
| 缺失值统计 | `df.isnull().sum()` |
| 条件筛选 | `df[df["col"] > value]` |
| 分组聚合 | `df.groupby("col").agg(...)` |
| 合并表格 | `pd.merge(df1, df2, on="key")` |
| 排序 | `df.sort_values("col", ascending=False)` |
| 去重 | `df.drop_duplicates()` |
| 透视表 | `df.pivot_table(...)` |
| 导出 Excel | `df.to_excel("out.xlsx", index=False)` |

## 小结

Pandas 是数据分析师的核心武器。本篇覆盖了日常工作中 90% 的操作：

- **数据读取**：CSV、Excel、SQL 一行搞定
- **数据探索**：5 步快速了解数据全貌
- **数据清洗**：缺失值、重复值、类型转换
- **筛选与计算**：条件筛选、新增列、apply 函数
- **分组聚合**：GroupBy 实现各维度汇总
- **数据合并**：merge 和 concat 处理多表关联
- **透视表**：交叉分析的利器

掌握了 Pandas，下一篇我们用 [Matplotlib 和 Seaborn](../03-visualization/index.html) 把数据变成图表，让分析结论更直观、更有说服力。
