---
title: "数据可视化实战"
layout: default
description: "使用 Matplotlib 和 Seaborn 进行数据可视化——从基础图表到高级统计图，结合真实业务场景，让数据洞察一目了然。"
eyebrow: "Python · 第 3 篇"
---

# 数据可视化实战

> **一图胜千言。** 你用 Pandas 算出的增长率、转化率、留存率，如果只是数字铺在表格里，业务方看了可能一脸茫然。但换成一张趋势图、一张漏斗图，结论瞬间清晰。本篇教你用 Python 制作专业级的数据分析图表。

## Python 可视化库全景

| 库 | 特点 | 适用场景 |
|---|------|---------|
| **Matplotlib** | 底层绑定，灵活度最高 | 自定义需求高的图表、论文配图 |
| **Seaborn** | 基于 Matplotlib，自带统计功能 | 探索性分析、统计图表 |
| **Plotly** | 交互式图表，支持拖拽缩放 | 数据报告、网页展示 |
| **Pyecharts** | 百度 ECharts 的 Python 封装 | 需要中文友好的交互式图表 |

本篇重点讲 **Matplotlib + Seaborn**，它们是数据分析师最常用的组合。

## 环境准备

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

# 设置中文字体（解决中文乱码问题）
plt.rcParams["font.sans-serif"] = ["SimHei", "Arial Unicode MS", "Noto Sans CJK SC"]
plt.rcParams["axes.unicode_minus"] = False  # 解决负号显示问题

# 设置 Seaborn 主题
sns.set_theme(style="whitegrid", font_scale=1.1)
```

> **常见坑**：Matplotlib 默认不支持中文，如果图表出现方框乱码，一定要先设置中文字体。Mac 用户推荐 `Arial Unicode MS`，Windows 用户用 `SimHei`，Linux 用 `Noto Sans CJK SC`。

## Matplotlib 基础

### 第一张图：折线图

```python
# 模拟某电商平台近 7 天的 DAU 数据
dates = pd.date_range("2024-03-01", periods=7, freq="D")
dau = [85000, 82000, 91000, 88000, 95000, 110000, 105000]

plt.figure(figsize=(10, 5))
plt.plot(dates, dau, marker="o", color="#4F46E5", linewidth=2, markersize=6)

plt.title("近 7 日 DAU 趋势", fontsize=16, fontweight="bold")
plt.xlabel("日期")
plt.ylabel("日活跃用户数")
plt.grid(True, alpha=0.3)

# 标注最大值
max_idx = dau.index(max(dau))
plt.annotate(
    f"峰值: {max(dau):,}",
    xy=(dates[max_idx], max(dau)),
    xytext=(dates[max_idx], max(dau) + 3000),
    fontsize=11,
    ha="center",
    arrowprops=dict(arrowstyle="->", color="red"),
    color="red",
)

plt.tight_layout()
plt.savefig("dau_trend.png", dpi=150)
plt.show()
```

### 柱状图：渠道对比

```python
channels = ["微信", "抖音", "小红书", "百度SEM", "自然搜索"]
orders = [4200, 3800, 2100, 1500, 3200]
colors = ["#4F46E5", "#06B6D4", "#F59E0B", "#EF4444", "#10B981"]

plt.figure(figsize=(10, 5))
bars = plt.bar(channels, orders, color=colors, edgecolor="white", linewidth=0.8)

# 在柱子上方标注数值
for bar, val in zip(bars, orders):
    plt.text(
        bar.get_x() + bar.get_width() / 2,
        bar.get_height() + 50,
        f"{val:,}",
        ha="center",
        fontsize=11,
        fontweight="bold",
    )

plt.title("各渠道月订单量对比", fontsize=16, fontweight="bold")
plt.ylabel("订单量")
plt.ylim(0, max(orders) * 1.15)
plt.tight_layout()
plt.show()
```

### 饼图 / 环形图：占比分析

```python
labels = ["微信", "抖音", "小红书", "百度SEM", "自然搜索"]
sizes = [4200, 3800, 2100, 1500, 3200]
colors = ["#4F46E5", "#06B6D4", "#F59E0B", "#EF4444", "#10B981"]
explode = [0.05, 0, 0, 0, 0]  # 突出第一个扇区

fig, ax = plt.subplots(figsize=(8, 8))
wedges, texts, autotexts = ax.pie(
    sizes,
    labels=labels,
    colors=colors,
    explode=explode,
    autopct="%1.1f%%",
    startangle=90,
    pctdistance=0.75,
    wedgeprops=dict(width=0.45),  # 环形图效果
)
ax.set_title("各渠道订单占比", fontsize=16, fontweight="bold")
plt.tight_layout()
plt.show()
```

### 子图组合：一页多图

```python
fig, axes = plt.subplots(1, 3, figsize=(18, 5))

# 子图 1：趋势
axes[0].plot(dates, dau, marker="o", color="#4F46E5")
axes[0].set_title("DAU 趋势")
axes[0].set_ylabel("DAU")

# 子图 2：渠道对比
axes[1].barh(channels, orders, color=colors)
axes[1].set_title("渠道订单量")
axes[1].set_xlabel("订单量")

# 子图 3：占比
axes[2].pie(sizes, labels=labels, colors=colors, autopct="%1.0f%%")
axes[2].set_title("渠道订单占比")

fig.suptitle("电商运营日报 - 2024.03.07", fontsize=18, fontweight="bold", y=1.02)
plt.tight_layout()
plt.show()
```

## Seaborn 进阶可视化

Seaborn 在 Matplotlib 的基础上提供了更高级的统计图表，代码更简洁，默认配色更美观。

### 分布图：了解数据分布

```python
# 模拟用户消费金额数据
np.random.seed(42)
spend_data = pd.DataFrame({
    "消费金额": np.concatenate([
        np.random.normal(200, 80, 500),   # 普通用户
        np.random.normal(800, 200, 100),   # 高消费用户
    ]),
    "用户类型": ["普通用户"] * 500 + ["VIP用户"] * 100,
})

fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# 直方图 + KDE 核密度估计
sns.histplot(data=spend_data, x="消费金额", hue="用户类型", kde=True, ax=axes[0])
axes[0].set_title("用户消费金额分布")

# 箱线图——发现异常值
sns.boxplot(data=spend_data, x="用户类型", y="消费金额", palette="Set2", ax=axes[1])
axes[1].set_title("不同用户类型消费金额箱线图")

plt.tight_layout()
plt.show()
```

### 热力图：相关性分析

```python
# 模拟电商指标数据
np.random.seed(42)
metrics = pd.DataFrame({
    "UV": np.random.randint(5000, 20000, 30),
    "加购数": np.random.randint(500, 3000, 30),
    "订单数": np.random.randint(100, 800, 30),
    "GMV": np.random.randint(30000, 150000, 30),
    "退货数": np.random.randint(10, 100, 30),
    "客单价": np.random.uniform(100, 500, 30),
})

# 计算相关系数矩阵
corr = metrics.corr()

plt.figure(figsize=(8, 6))
sns.heatmap(
    corr,
    annot=True,          # 显示数值
    fmt=".2f",           # 保留 2 位小数
    cmap="RdYlBu_r",    # 配色方案
    center=0,
    square=True,
    linewidths=0.5,
)
plt.title("电商核心指标相关性矩阵", fontsize=14, fontweight="bold")
plt.tight_layout()
plt.show()
```

> **业务洞察**：热力图可以快速发现哪些指标之间强相关。例如 UV 和订单数正相关（流量越大、订单越多），而退货数和客单价可能负相关（高客单价商品用户更谨慎、退货率更低）。

### 散点图 + 回归线：关系分析

```python
plt.figure(figsize=(8, 6))
sns.regplot(
    data=metrics,
    x="UV",
    y="订单数",
    scatter_kws={"alpha": 0.6, "s": 60},
    line_kws={"color": "red"},
)
plt.title("UV 与订单数的关系", fontsize=14, fontweight="bold")
plt.xlabel("UV（访问用户数）")
plt.ylabel("订单数")
plt.tight_layout()
plt.show()
```

### 分面图：多维度对比

```python
# 模拟多渠道多月份数据
np.random.seed(42)
multi_data = pd.DataFrame({
    "月份": np.tile(["1月", "2月", "3月", "4月", "5月", "6月"], 3),
    "渠道": np.repeat(["微信", "抖音", "小红书"], 6),
    "GMV": np.random.randint(50000, 200000, 18),
    "订单数": np.random.randint(200, 1000, 18),
})

g = sns.FacetGrid(multi_data, col="渠道", height=4, aspect=1.2)
g.map_dataframe(sns.barplot, x="月份", y="GMV", palette="viridis")
g.set_titles("{col_name}")
g.set_axis_labels("月份", "GMV (元)")
g.fig.suptitle("各渠道月度 GMV 对比", fontsize=16, fontweight="bold", y=1.03)
plt.tight_layout()
plt.show()
```

## 业务场景实战

### 场景一：漏斗图

电商转化漏斗是最常见的分析需求之一：

```python
# 漏斗数据
stages = ["浏览商品", "加入购物车", "进入结算", "提交订单", "支付成功"]
values = [100000, 35000, 18000, 12000, 9500]
rates = [f"{v/values[0]*100:.1f}%" for v in values]

fig, ax = plt.subplots(figsize=(10, 6))
colors = plt.cm.Blues(np.linspace(0.3, 0.9, len(stages)))

for i, (stage, val, rate) in enumerate(zip(stages, values, rates)):
    width = val / values[0]
    left = (1 - width) / 2
    ax.barh(len(stages) - 1 - i, width, left=left, height=0.6, color=colors[i])
    ax.text(0.5, len(stages) - 1 - i, f"{stage}\n{val:,} ({rate})",
            ha="center", va="center", fontsize=11, fontweight="bold")

ax.set_xlim(0, 1)
ax.set_yticks([])
ax.set_xticks([])
ax.set_title("电商转化漏斗", fontsize=16, fontweight="bold")
ax.spines["top"].set_visible(False)
ax.spines["right"].set_visible(False)
ax.spines["bottom"].set_visible(False)
ax.spines["left"].set_visible(False)
plt.tight_layout()
plt.show()
```

### 场景二：留存率曲线

```python
# 模拟 7 日留存率数据
days = list(range(1, 8))
retention_organic = [45, 32, 28, 25, 23, 22, 21]
retention_paid = [38, 22, 16, 13, 11, 10, 9]

plt.figure(figsize=(10, 5))
plt.plot(days, retention_organic, marker="o", linewidth=2, label="自然流量", color="#10B981")
plt.plot(days, retention_paid, marker="s", linewidth=2, label="付费流量", color="#EF4444")

plt.fill_between(days, retention_organic, alpha=0.1, color="#10B981")
plt.fill_between(days, retention_paid, alpha=0.1, color="#EF4444")

plt.title("新用户 7 日留存率对比", fontsize=16, fontweight="bold")
plt.xlabel("注册后第 N 天")
plt.ylabel("留存率 (%)")
plt.legend()
plt.xticks(days)
plt.ylim(0, 55)
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

> **业务洞察**：自然流量的留存率显著高于付费流量，说明自然流量的用户质量更高。付费流量在第 3 天后留存快速衰减，可以考虑在第 2-3 天推送优惠券来提升留存。

### 场景三：同比环比分析

```python
months = ["1月", "2月", "3月", "4月", "5月", "6月"]
this_year = [120, 135, 118, 142, 156, 149]
last_year = [98, 105, 110, 115, 128, 135]

x = np.arange(len(months))
width = 0.35

fig, ax1 = plt.subplots(figsize=(12, 6))

# 柱状图：绝对值
bars1 = ax1.bar(x - width/2, last_year, width, label="2023年", color="#94A3B8", alpha=0.8)
bars2 = ax1.bar(x + width/2, this_year, width, label="2024年", color="#4F46E5", alpha=0.9)
ax1.set_ylabel("GMV (万元)", fontsize=12)
ax1.set_xticks(x)
ax1.set_xticklabels(months)
ax1.legend(loc="upper left")

# 折线图：同比增长率（双轴）
ax2 = ax1.twinx()
yoy_growth = [(t - l) / l * 100 for t, l in zip(this_year, last_year)]
ax2.plot(x, yoy_growth, color="#EF4444", marker="D", linewidth=2, label="同比增长率")
ax2.set_ylabel("同比增长率 (%)", fontsize=12, color="#EF4444")
ax2.legend(loc="upper right")

plt.title("月度 GMV 同比分析", fontsize=16, fontweight="bold")
plt.tight_layout()
plt.show()
```

## 图表美化清单

制作正式的分析报告时，注意以下细节：

| 要素 | 建议 |
|------|------|
| **标题** | 简洁明了，包含指标名称和时间范围 |
| **坐标轴标签** | 说明单位（元/万元/%/人） |
| **图例** | 有多组数据时必须添加 |
| **数据标签** | 关键数值直接标注在图上 |
| **配色** | 同系列用同一色系，对比用互补色，不超过 5 种颜色 |
| **网格线** | 降低透明度，辅助读数但不抢眼 |
| **导出分辨率** | `dpi=150` 以上，用于报告或 PPT |
| **图片比例** | 趋势图宽扁（16:9），对比图适当宽（4:3） |

```python
# 导出高质量图片
plt.savefig("chart.png", dpi=200, bbox_inches="tight", facecolor="white")
```

## 小结

本篇覆盖了数据分析师最常用的可视化技能：

- **Matplotlib 基础**：折线图、柱状图、饼图、子图组合
- **Seaborn 进阶**：分布图、热力图、散点回归图、分面图
- **业务场景**：转化漏斗、留存曲线、同比环比分析
- **美化技巧**：字体、配色、标注、导出

学会了数据处理和可视化，下一篇我们将在 [实战：数据清洗与分析](../04-project/index.html) 中把所有技能串起来，完成一个从原始数据到分析报告的完整项目。
