---
title: "实战：数据清洗与分析"
layout: default
description: "完整的 Python 数据分析实战项目——从拿到原始电商数据，经过清洗、探索、分析、可视化，最终输出一份可放进简历的分析报告。"
eyebrow: "Python · 第 4 篇"
---

# 实战：数据清洗与分析

> **学了这么多，能不能做一个完整项目？** 前三篇我们学了 Python 基础、Pandas 数据处理、可视化。现在把它们全部串起来，模拟一个真实的数据分析工作场景：产品经理给你一份原始数据，要求你产出分析结论和可视化报告。

## 项目背景

你是一家电商平台的数据分析师，产品经理提出以下需求：

> "最近 3 个月的销售数据我导出来了，你帮我看看：
> 1. 整体销售趋势怎么样？
> 2. 哪些品类卖得好？哪些在下滑？
> 3. 不同渠道的获客效率如何？
> 4. 有没有什么值得关注的异常？"

## 第一步：加载与初探

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

# 中文字体设置
plt.rcParams["font.sans-serif"] = ["SimHei", "Arial Unicode MS"]
plt.rcParams["axes.unicode_minus"] = False
sns.set_theme(style="whitegrid", font_scale=1.1)

# 模拟原始数据（实际项目中用 pd.read_csv 读取）
np.random.seed(42)
n = 5000

dates = pd.date_range("2024-01-01", "2024-03-31", freq="h")
sample_dates = np.random.choice(dates, n)

raw_data = pd.DataFrame({
    "order_id": [f"ORD{str(i).zfill(6)}" for i in range(1, n + 1)],
    "order_date": sample_dates,
    "user_id": [f"U{np.random.randint(1000, 3000)}" for _ in range(n)],
    "category": np.random.choice(
        ["服装", "数码", "食品", "美妆", "家居", "图书"], n,
        p=[0.25, 0.15, 0.20, 0.18, 0.12, 0.10]
    ),
    "channel": np.random.choice(
        ["微信", "抖音", "小红书", "百度SEM", "自然搜索"], n,
        p=[0.30, 0.25, 0.20, 0.10, 0.15]
    ),
    "amount": np.round(np.random.lognormal(5, 1, n), 2),
    "quantity": np.random.randint(1, 6, n),
    "city": np.random.choice(
        ["北京", "上海", "广州", "深圳", "杭州", "成都", "武汉", "南京"], n
    ),
})

# 故意注入一些脏数据
raw_data.loc[np.random.choice(n, 150), "amount"] = np.nan
raw_data.loc[np.random.choice(n, 80), "city"] = None
raw_data.loc[np.random.choice(n, 20), "amount"] = -abs(np.random.normal(500, 200, 20))
raw_data.loc[np.random.choice(n, 5), "amount"] = np.random.uniform(50000, 100000, 5)

print(f"原始数据量: {raw_data.shape[0]} 行, {raw_data.shape[1]} 列")
raw_data.head(10)
```

### 数据初探 5 步法

```python
# 1. 基本信息
print("=" * 50)
print("【数据基本信息】")
print(raw_data.info())

# 2. 数值列统计
print("\n【数值列统计】")
print(raw_data.describe())

# 3. 缺失值
print("\n【缺失值统计】")
missing = raw_data.isnull().sum()
missing_pct = (missing / len(raw_data) * 100).round(1)
print(pd.DataFrame({"缺失数": missing, "缺失率%": missing_pct}))

# 4. 各分类字段的唯一值
print("\n【分类字段分布】")
for col in ["category", "channel", "city"]:
    print(f"\n{col} ({raw_data[col].nunique()} 个唯一值):")
    print(raw_data[col].value_counts().head())

# 5. 日期范围
print(f"\n日期范围: {raw_data['order_date'].min()} ~ {raw_data['order_date'].max()}")
```

## 第二步：数据清洗

```python
df = raw_data.copy()

# 2.1 处理缺失值
print(f"清洗前: {len(df)} 行")

# 金额缺失 → 删除（无法合理填充）
df = df.dropna(subset=["amount"])
print(f"删除金额缺失后: {len(df)} 行")

# 城市缺失 → 填充为"未知"
df["city"] = df["city"].fillna("未知")

# 2.2 处理异常值
# 金额为负数 → 可能是退货，标记后排除
df = df[df["amount"] > 0]
print(f"排除负数金额后: {len(df)} 行")

# 金额极端值 → 使用 IQR 方法检测
Q1 = df["amount"].quantile(0.25)
Q3 = df["amount"].quantile(0.75)
IQR = Q3 - Q1
upper_bound = Q3 + 3 * IQR  # 使用 3 倍 IQR，更宽松

outliers = df[df["amount"] > upper_bound]
print(f"\n极端高额订单（>{upper_bound:.0f}元）: {len(outliers)} 条")
print(outliers[["order_id", "amount", "category"]].head())

# 将极端值截断
df["amount_clean"] = df["amount"].clip(upper=upper_bound)

# 2.3 日期处理
df["order_date"] = pd.to_datetime(df["order_date"])
df["date"] = df["order_date"].dt.date
df["month"] = df["order_date"].dt.to_period("M")
df["weekday"] = df["order_date"].dt.day_name()
df["hour"] = df["order_date"].dt.hour

print(f"\n最终清洗后数据量: {len(df)} 行")
print("新增字段: date, month, weekday, hour, amount_clean")
```

## 第三步：探索性分析

### 3.1 整体销售趋势

```python
daily = df.groupby("date").agg(
    订单量=("order_id", "count"),
    GMV=("amount_clean", "sum"),
    客单价=("amount_clean", "mean"),
    用户数=("user_id", "nunique"),
).reset_index()

fig, axes = plt.subplots(2, 2, figsize=(16, 10))

# 每日订单量趋势
axes[0, 0].plot(daily["date"], daily["订单量"], color="#4F46E5", linewidth=1.5)
axes[0, 0].set_title("每日订单量趋势")
axes[0, 0].set_ylabel("订单量")

# 每日 GMV 趋势
axes[0, 1].plot(daily["date"], daily["GMV"], color="#10B981", linewidth=1.5)
axes[0, 1].set_title("每日 GMV 趋势")
axes[0, 1].set_ylabel("GMV (元)")

# 客单价趋势
axes[1, 0].plot(daily["date"], daily["客单价"], color="#F59E0B", linewidth=1.5)
axes[1, 0].set_title("客单价趋势")
axes[1, 0].set_ylabel("客单价 (元)")

# 每日独立用户数
axes[1, 1].plot(daily["date"], daily["用户数"], color="#EF4444", linewidth=1.5)
axes[1, 1].set_title("每日独立用户数")
axes[1, 1].set_ylabel("用户数")

fig.suptitle("电商平台 Q1 销售概览", fontsize=18, fontweight="bold")
plt.tight_layout()
plt.savefig("01_daily_overview.png", dpi=150, bbox_inches="tight")
plt.show()
```

### 3.2 品类分析

```python
category_stats = df.groupby("category").agg(
    订单量=("order_id", "count"),
    GMV=("amount_clean", "sum"),
    客单价=("amount_clean", "mean"),
    用户数=("user_id", "nunique"),
).sort_values("GMV", ascending=False).reset_index()

category_stats["GMV占比"] = (category_stats["GMV"] / category_stats["GMV"].sum() * 100).round(1)

print("=== 品类销售排名 ===")
print(category_stats.to_string(index=False))

# 可视化
fig, axes = plt.subplots(1, 2, figsize=(14, 6))

# 品类 GMV 柱状图
bars = axes[0].barh(
    category_stats["category"],
    category_stats["GMV"],
    color=plt.cm.Set2(np.linspace(0, 1, len(category_stats)))
)
axes[0].set_xlabel("GMV (元)")
axes[0].set_title("各品类 GMV 排名")
for bar, pct in zip(bars, category_stats["GMV占比"]):
    axes[0].text(bar.get_width() + 1000, bar.get_y() + bar.get_height()/2,
                 f"{pct}%", va="center", fontsize=10)

# 品类占比饼图
axes[1].pie(
    category_stats["GMV"],
    labels=category_stats["category"],
    autopct="%1.1f%%",
    startangle=90,
    colors=plt.cm.Set2(np.linspace(0, 1, len(category_stats)))
)
axes[1].set_title("GMV 品类占比")

plt.tight_layout()
plt.savefig("02_category_analysis.png", dpi=150, bbox_inches="tight")
plt.show()
```

### 3.3 月度品类变化趋势

```python
monthly_cat = df.groupby(["month", "category"]).agg(
    GMV=("amount_clean", "sum")
).reset_index()
monthly_cat["month"] = monthly_cat["month"].astype(str)

plt.figure(figsize=(12, 6))
sns.barplot(data=monthly_cat, x="month", y="GMV", hue="category", palette="Set2")
plt.title("各品类月度 GMV 变化", fontsize=16, fontweight="bold")
plt.xlabel("月份")
plt.ylabel("GMV (元)")
plt.legend(title="品类", bbox_to_anchor=(1.02, 1), loc="upper left")
plt.tight_layout()
plt.savefig("03_monthly_category.png", dpi=150, bbox_inches="tight")
plt.show()
```

### 3.4 渠道分析

```python
channel_stats = df.groupby("channel").agg(
    订单量=("order_id", "count"),
    GMV=("amount_clean", "sum"),
    用户数=("user_id", "nunique"),
    客单价=("amount_clean", "mean"),
).reset_index()

# 计算人均贡献
channel_stats["人均GMV"] = channel_stats["GMV"] / channel_stats["用户数"]
channel_stats = channel_stats.sort_values("GMV", ascending=False)

print("=== 渠道效率对比 ===")
print(channel_stats.to_string(index=False, float_format="{:.0f}".format))

# 可视化：渠道效率气泡图
plt.figure(figsize=(10, 7))
scatter = plt.scatter(
    channel_stats["用户数"],
    channel_stats["客单价"],
    s=channel_stats["GMV"] / 500,  # 气泡大小 = GMV
    c=range(len(channel_stats)),
    cmap="Set1",
    alpha=0.7,
    edgecolors="black",
    linewidth=0.5,
)
for _, row in channel_stats.iterrows():
    plt.annotate(
        row["channel"],
        (row["用户数"], row["客单价"]),
        fontsize=12,
        fontweight="bold",
        ha="center",
        va="bottom",
    )
plt.xlabel("用户数", fontsize=12)
plt.ylabel("客单价 (元)", fontsize=12)
plt.title("渠道效率气泡图（气泡大小 = GMV）", fontsize=16, fontweight="bold")
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig("04_channel_bubble.png", dpi=150, bbox_inches="tight")
plt.show()
```

### 3.5 时间维度分析

```python
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# 每小时订单分布
hourly = df.groupby("hour")["order_id"].count()
axes[0].bar(hourly.index, hourly.values, color="#4F46E5", alpha=0.8)
axes[0].set_title("24 小时订单分布")
axes[0].set_xlabel("小时")
axes[0].set_ylabel("订单量")
axes[0].set_xticks(range(0, 24, 2))

# 星期几订单分布
weekday_order = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"]
weekday_cn = ["周一", "周二", "周三", "周四", "周五", "周六", "周日"]
weekly = df.groupby("weekday")["order_id"].count().reindex(weekday_order)
axes[1].bar(weekday_cn, weekly.values, color="#10B981", alpha=0.8)
axes[1].set_title("每周订单分布")
axes[1].set_ylabel("订单量")

plt.tight_layout()
plt.savefig("05_time_analysis.png", dpi=150, bbox_inches="tight")
plt.show()
```

## 第四步：深度分析

### 4.1 RFM 用户分层

```python
# 以 2024-03-31 为基准日期
snapshot_date = pd.Timestamp("2024-03-31")

rfm = df.groupby("user_id").agg(
    R=("order_date", lambda x: (snapshot_date - x.max()).days),  # 最近一次消费距今天数
    F=("order_id", "count"),                                       # 消费频次
    M=("amount_clean", "sum"),                                     # 消费总金额
).reset_index()

# 打分（1-4 分）
rfm["R_score"] = pd.qcut(rfm["R"], 4, labels=[4, 3, 2, 1])  # R 越小越好
rfm["F_score"] = pd.qcut(rfm["F"].rank(method="first"), 4, labels=[1, 2, 3, 4])
rfm["M_score"] = pd.qcut(rfm["M"].rank(method="first"), 4, labels=[1, 2, 3, 4])

# 综合标签
def rfm_label(row):
    r, f, m = int(row["R_score"]), int(row["F_score"]), int(row["M_score"])
    if r >= 3 and f >= 3 and m >= 3:
        return "高价值用户"
    elif r >= 3 and f < 3:
        return "新用户/潜力用户"
    elif r < 3 and f >= 3:
        return "沉睡忠诚用户"
    elif r < 3 and m >= 3:
        return "流失高价值用户"
    else:
        return "低价值用户"

rfm["用户标签"] = rfm.apply(rfm_label, axis=1)

# 统计各群体
segment_stats = rfm.groupby("用户标签").agg(
    用户数=("user_id", "count"),
    平均消费频次=("F", "mean"),
    平均消费金额=("M", "mean"),
).round(1)
segment_stats["占比%"] = (segment_stats["用户数"] / segment_stats["用户数"].sum() * 100).round(1)
print("=== RFM 用户分层 ===")
print(segment_stats)
```

### 4.2 品类交叉购买分析

```python
from itertools import combinations

# 找出同一用户购买了哪些品类
user_cats = df.groupby("user_id")["category"].apply(set).reset_index()
user_cats = user_cats[user_cats["category"].apply(len) >= 2]  # 至少购买 2 个品类

# 统计品类对的共同购买次数
pair_count = {}
for cats in user_cats["category"]:
    for pair in combinations(sorted(cats), 2):
        pair_count[pair] = pair_count.get(pair, 0) + 1

cross_buy = pd.DataFrame(
    [(p[0], p[1], c) for p, c in pair_count.items()],
    columns=["品类A", "品类B", "共同购买用户数"]
).sort_values("共同购买用户数", ascending=False)

print("=== 品类交叉购买 Top 10 ===")
print(cross_buy.head(10).to_string(index=False))
```

> **业务建议**：交叉购买频率高的品类对可以做联合营销。例如"服装+美妆"共同购买率高，可以在服装详情页推荐美妆产品，或设计跨品类满减活动。

## 第五步：输出分析报告

### 关键指标汇总

```python
total_orders = len(df)
total_gmv = df["amount_clean"].sum()
total_users = df["user_id"].nunique()
avg_aov = df["amount_clean"].mean()
avg_orders_per_user = total_orders / total_users

print("=" * 50)
print("电商平台 2024 Q1 销售分析报告")
print("=" * 50)
print(f"分析周期: 2024-01-01 ~ 2024-03-31")
print(f"总订单量: {total_orders:,}")
print(f"总 GMV:   ¥{total_gmv:,.0f}")
print(f"独立用户: {total_users:,}")
print(f"客单价:   ¥{avg_aov:,.0f}")
print(f"人均下单: {avg_orders_per_user:.1f} 次")
print("=" * 50)
```

### 导出完整报告

```python
# 导出到 Excel（多 Sheet）
with pd.ExcelWriter("Q1_analysis_report.xlsx", engine="openpyxl") as writer:
    daily.to_excel(writer, sheet_name="每日汇总", index=False)
    category_stats.to_excel(writer, sheet_name="品类分析", index=False)
    channel_stats.to_excel(writer, sheet_name="渠道分析", index=False)
    segment_stats.to_excel(writer, sheet_name="RFM分层")
    cross_buy.head(20).to_excel(writer, sheet_name="交叉购买", index=False)

print("分析报告已导出: Q1_analysis_report.xlsx")
```

## 分析结论模板

一份完整的数据分析结论应该包含以下要素：

> **结论一：整体销售稳中有升**
> Q1 总 GMV 达到 XXX 万元，环比 Q4 增长 X%。其中 3 月表现最好，日均 GMV 达到 XXX 元，主要由"XXX"品类拉动。
>
> **结论二：品类表现分化明显**
> 服装品类贡献了 XX% 的 GMV，是绝对主力。数码品类客单价最高（¥XXX），但订单量偏低，建议加大流量投入。图书品类 GMV 占比仅 X%，且环比下滑，需关注。
>
> **结论三：渠道效率差异大**
> 微信渠道贡献了最多用户和 GMV，但客单价较低（¥XXX）；小红书渠道客单价最高（¥XXX），用户质量好但流量偏小，建议加大投放预算。
>
> **建议**：
> 1. 加大小红书渠道投放，提升高质量用户占比
> 2. 服装 + 美妆做联合营销，提升交叉购买率
> 3. 关注 20-22 点订单高峰期，在此时段推送促销活动

## 项目总结

本项目完整演示了数据分析师的日常工作流程：

| 步骤 | 核心操作 | 使用的工具 |
|------|---------|-----------|
| 加载与初探 | 读取数据、5 步探索法 | Pandas |
| 数据清洗 | 缺失值、异常值、类型转换 | Pandas / NumPy |
| 探索性分析 | 趋势、品类、渠道、时间维度 | Pandas + Matplotlib |
| 深度分析 | RFM 分层、交叉购买 | Pandas |
| 输出报告 | 关键指标、图表、结论建议 | Pandas + Excel |

> **放进简历的建议**：将这个项目稍作修改（使用真实公开数据集如 Kaggle 的 E-Commerce 数据），上传到 GitHub，在简历中描述为"独立完成电商平台 Q1 销售分析，产出品类优化和渠道策略建议，协助运营团队提升 GMV"。

恭喜你完成了 Python 数据分析模块的全部学习！接下来进入 [可视化 & BI](../../learn-da-bi/index.html) 模块，学习用 Tableau 和 Power BI 制作交互式数据看板。
