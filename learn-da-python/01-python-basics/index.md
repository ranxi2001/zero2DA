---
title: "Python 基础入门"
layout: default
description: "从零开始学习 Python 编程——环境搭建、变量与数据类型、流程控制、函数定义、常用数据结构，为数据分析打下扎实的编程基础。"
eyebrow: "Python · 第 1 篇"
---

# Python 基础入门

> **写给零基础的你**：这篇文章不假设你有任何编程经验。我们会从"什么是 Python"讲起，一步步带你搭建环境、理解核心语法，写出你的第一段数据分析代码。

## 为什么数据分析师要学 Python

在数据分析师的日常工作中，Python 解决的是 Excel 和 SQL 力所不及的问题：

- **处理大规模数据**：百万行的数据在 Excel 中根本打不开，Python 的 Pandas 库可以轻松应对。
- **自动化重复工作**：每周要出的报表、每天要跑的数据校验，写一个脚本自动完成。
- **高级分析与建模**：回归分析、聚类分析、时间序列预测——Python 的生态让这些分析只需几行代码。
- **精美可视化**：Matplotlib、Seaborn、Plotly 提供的图表类型远超 Excel。

## 环境搭建

### 方案一：Anaconda（推荐新手）

Anaconda 是一个 Python 发行版，预装了数据分析需要的绝大部分库。

1. 访问 [Anaconda 官网](https://www.anaconda.com/download) 下载安装包
2. 安装时勾选"Add to PATH"（Windows）或使用默认设置（Mac）
3. 打开终端（Mac）或 Anaconda Prompt（Windows），输入：

```python
# 验证安装是否成功
python --version
# 输出类似：Python 3.11.5

# 启动 Jupyter Notebook
jupyter notebook
```

### 方案二：Google Colab（零安装）

如果不想在本地安装任何软件，可以直接使用 [Google Colab](https://colab.research.google.com/)。它在浏览器中运行，免费提供 GPU，而且预装了所有常用库。

### 方案三：VS Code + venv

适合有一定经验的用户，更灵活但需要手动配置：

```python
# 创建虚拟环境
python -m venv da_env

# 激活虚拟环境（Mac/Linux）
source da_env/bin/activate

# 激活虚拟环境（Windows）
da_env\Scripts\activate

# 安装数据分析核心包
pip install pandas numpy matplotlib seaborn jupyter
```

## 变量与数据类型

Python 是动态类型语言，不需要提前声明变量类型，直接赋值即可。

### 基本数据类型

```python
# 整数（int）
user_count = 15000
print(type(user_count))  # <class 'int'>

# 浮点数（float）
conversion_rate = 0.032
print(type(conversion_rate))  # <class 'float'>

# 字符串（str）
channel = "微信小程序"
print(type(channel))  # <class 'str'>

# 布尔值（bool）
is_active = True
print(type(is_active))  # <class 'bool'>

# None 类型——表示"没有值"
missing_value = None
```

### 类型转换

在数据分析中经常需要做类型转换，例如从 CSV 文件读入的数字可能是字符串格式：

```python
# 字符串转数字
revenue_str = "128500"
revenue = int(revenue_str)
print(revenue + 1000)  # 129500

# 数字转字符串（用于拼接文本）
month = 3
report_title = "2024年第" + str(month) + "季度报告"
print(report_title)  # 2024年第3季度报告

# 更推荐使用 f-string 格式化
report_title = f"2024年第{month}季度报告"
print(report_title)  # 2024年第3季度报告
```

## 运算符

### 算术运算符

```python
# 数据分析中常用的计算
total_revenue = 500000
total_users = 12000

# 人均收入
arpu = total_revenue / total_users
print(f"ARPU: {arpu:.2f}")  # ARPU: 41.67

# 整除和取余——常用于分组
user_id = 10237
group = user_id % 2  # 用于 A/B 分组：0 为对照组，1 为实验组
print(f"用户 {user_id} 属于{'实验组' if group else '对照组'}")

# 幂运算——常用于增长率计算
monthly_growth = 1.05
annual_growth = monthly_growth ** 12
print(f"年化增长率: {(annual_growth - 1) * 100:.1f}%")  # 年化增长率: 79.6%
```

### 比较与逻辑运算符

```python
# 比较运算
daily_active_users = 8500
target = 10000
print(daily_active_users >= target)  # False

# 逻辑运算——数据筛选的基础
age = 25
city = "北京"

# and：两个条件都满足
is_target_user = (age >= 18) and (age <= 35) and (city == "北京")
print(is_target_user)  # True

# or：满足任一条件
is_tier1_city = (city == "北京") or (city == "上海") or (city == "广州") or (city == "深圳")
print(is_tier1_city)  # True
```

## 流程控制

### 条件判断（if / elif / else）

```python
def classify_user(monthly_spend):
    """根据月消费金额对用户进行分层"""
    if monthly_spend >= 5000:
        return "高价值用户"
    elif monthly_spend >= 1000:
        return "中等价值用户"
    elif monthly_spend >= 100:
        return "普通用户"
    else:
        return "低活跃用户"

# 测试
print(classify_user(8000))  # 高价值用户
print(classify_user(500))   # 普通用户
print(classify_user(30))    # 低活跃用户
```

### 循环（for / while）

```python
# for 循环——遍历数据
monthly_revenue = [120000, 135000, 118000, 142000, 156000, 149000]

# 计算总收入和平均收入
total = 0
for revenue in monthly_revenue:
    total += revenue

average = total / len(monthly_revenue)
print(f"半年总收入: {total:,}")       # 半年总收入: 820,000
print(f"月均收入: {average:,.0f}")    # 月均收入: 136,667

# 使用 enumerate 同时获取索引和值
for i, revenue in enumerate(monthly_revenue, start=1):
    mom_change = ""
    if i > 1:
        change = (revenue - monthly_revenue[i-2]) / monthly_revenue[i-2] * 100
        mom_change = f"  环比: {change:+.1f}%"
    print(f"第{i}月: ¥{revenue:,}{mom_change}")
```

输出：

```
第1月: ¥120,000
第2月: ¥135,000  环比: +12.5%
第3月: ¥118,000  环比: -12.6%
第4月: ¥142,000  环比: +20.3%
第5月: ¥156,000  环比: +9.9%
第6月: ¥149,000  环比: -4.5%
```

### 列表推导式

列表推导式是 Python 的一大特色，数据分析中非常常用：

```python
# 传统写法
high_revenue_months = []
for i, rev in enumerate(monthly_revenue, 1):
    if rev > 140000:
        high_revenue_months.append(i)

# 列表推导式——一行搞定
high_revenue_months = [i for i, rev in enumerate(monthly_revenue, 1) if rev > 140000]
print(f"收入超过14万的月份: {high_revenue_months}")  # [4, 5, 6]

# 对列表中的每个元素做转换
revenue_in_wan = [rev / 10000 for rev in monthly_revenue]
print(revenue_in_wan)  # [12.0, 13.5, 11.8, 14.2, 15.6, 14.9]
```

## 常用数据结构

### 列表（List）

列表是有序、可变的数据集合，数据分析中用来存储一组数据：

```python
# 创建与操作
channels = ["微信", "抖音", "小红书", "百度SEM", "自然搜索"]

# 添加元素
channels.append("B站")

# 切片
top3 = channels[:3]
print(top3)  # ['微信', '抖音', '小红书']

# 排序
daily_sales = [230, 180, 310, 275, 190, 420, 350]
daily_sales_sorted = sorted(daily_sales, reverse=True)
print(f"销量排名: {daily_sales_sorted}")
print(f"最高销量: {max(daily_sales)}, 最低销量: {min(daily_sales)}")
```

### 字典（Dict）

字典用键值对存储数据，非常适合表示结构化信息：

```python
# 用户画像
user_profile = {
    "user_id": "U10086",
    "age": 28,
    "city": "上海",
    "channel": "抖音",
    "total_orders": 15,
    "total_spend": 4230.5,
    "is_vip": True
}

# 访问和修改
print(f"用户来源: {user_profile['channel']}")
user_profile["total_orders"] += 1  # 新增一单

# 安全地访问可能不存在的键
email = user_profile.get("email", "未填写")
print(f"邮箱: {email}")  # 邮箱: 未填写

# 遍历字典
for key, value in user_profile.items():
    print(f"  {key}: {value}")
```

### 用字典列表模拟数据表

在学 Pandas 之前，字典列表是最接近"数据表"的结构：

```python
orders = [
    {"order_id": "ORD001", "user_id": "U101", "amount": 299, "channel": "微信"},
    {"order_id": "ORD002", "user_id": "U102", "amount": 158, "channel": "抖音"},
    {"order_id": "ORD003", "user_id": "U101", "amount": 89,  "channel": "微信"},
    {"order_id": "ORD004", "user_id": "U103", "amount": 520, "channel": "小红书"},
    {"order_id": "ORD005", "user_id": "U102", "amount": 199, "channel": "抖音"},
]

# 统计各渠道订单量和总金额
channel_stats = {}
for order in orders:
    ch = order["channel"]
    if ch not in channel_stats:
        channel_stats[ch] = {"count": 0, "total": 0}
    channel_stats[ch]["count"] += 1
    channel_stats[ch]["total"] += order["amount"]

for ch, stats in channel_stats.items():
    avg = stats["total"] / stats["count"]
    print(f"{ch}: {stats['count']}单, 总额¥{stats['total']}, 客单价¥{avg:.0f}")
```

输出：

```
微信: 2单, 总额¥388, 客单价¥194
抖音: 2单, 总额¥357, 客单价¥179
小红书: 1单, 总额¥520, 客单价¥520
```

## 函数

函数是代码复用的核心。数据分析中经常需要把重复的计算逻辑封装成函数。

### 定义和调用函数

```python
def calculate_growth_rate(current, previous):
    """计算环比增长率"""
    if previous == 0:
        return None
    return (current - previous) / previous * 100

# 使用
q1_revenue = 350000
q2_revenue = 420000
growth = calculate_growth_rate(q2_revenue, q1_revenue)
print(f"Q2 环比增长: {growth:.1f}%")  # Q2 环比增长: 20.0%
```

### 带默认参数的函数

```python
def format_currency(amount, currency="¥", decimals=0):
    """格式化金额显示"""
    return f"{currency}{amount:,.{decimals}f}"

print(format_currency(1234567))        # ¥1,234,567
print(format_currency(1234.5, "$", 2)) # $1,234.50
```

### 返回多个值

```python
def analyze_dataset(data):
    """对数据集进行基础统计"""
    n = len(data)
    total = sum(data)
    average = total / n
    minimum = min(data)
    maximum = max(data)
    return n, total, average, minimum, maximum

sales = [230, 180, 310, 275, 190, 420, 350]
count, total, avg, low, high = analyze_dataset(sales)
print(f"数据量: {count}, 总计: {total}, 均值: {avg:.1f}, 最小: {low}, 最大: {high}")
```

## 文件读写

数据分析经常需要读取 CSV、文本文件等：

```python
# 写入 CSV 文件（不依赖第三方库）
import csv

data = [
    ["日期", "渠道", "UV", "订单数", "GMV"],
    ["2024-01-01", "微信", 12000, 360, 89000],
    ["2024-01-01", "抖音", 8500, 255, 63000],
    ["2024-01-02", "微信", 11500, 345, 86000],
    ["2024-01-02", "抖音", 9200, 276, 68000],
]

with open("daily_report.csv", "w", newline="", encoding="utf-8-sig") as f:
    writer = csv.writer(f)
    writer.writerows(data)

# 读取 CSV 文件
with open("daily_report.csv", "r", encoding="utf-8-sig") as f:
    reader = csv.DictReader(f)
    for row in reader:
        conv_rate = int(row["订单数"]) / int(row["UV"]) * 100
        print(f"{row['日期']} {row['渠道']}: 转化率 {conv_rate:.2f}%")
```

> **预告**：在下一篇 [Pandas 数据处理](../02-pandas/index.html) 中，你会发现用 `pd.read_csv()` 一行代码就能完成上面所有工作，而且功能强大百倍。

## 小结

本篇覆盖了 Python 数据分析所需的基础语法：

| 知识点 | 数据分析场景 |
|-------|------------|
| 变量与数据类型 | 存储指标值、用户属性 |
| 运算符 | 计算增长率、ARPU、转化率 |
| 条件判断 | 用户分层、数据标记 |
| 循环 | 遍历数据、批量计算 |
| 列表推导式 | 数据筛选、转换 |
| 字典 | 结构化数据存储 |
| 函数 | 封装计算逻辑、复用代码 |
| 文件读写 | 读取/写出 CSV 数据 |

掌握这些基础之后，下一篇我们进入数据分析师的核心武器—— [Pandas 数据处理](../02-pandas/index.html)。
