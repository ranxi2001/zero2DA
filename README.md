<div align="center">
<img src="assets/images/logo.svg" alt="zero2DA Logo" width="120" height="120" />

# zero2DA

**从零入门数据分析 · SQL · Python · BI 可视化 · 商业分析 · 求职面试**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Site](https://img.shields.io/badge/Site-onefly.top%2Fzero2DA-10b981)](https://onefly.top/zero2DA)

[在线阅读](https://onefly.top/zero2DA) · [DA 基础](https://onefly.top/zero2DA/learn-da-basic/) · [SQL](https://onefly.top/zero2DA/learn-da-sql/) · [Python](https://onefly.top/zero2DA/learn-da-python/) · [DA 面试](https://onefly.top/zero2DA/learn-da-interview/)

</div>

---

## 这是什么

**zero2DA** 是一个面向零基础转行者和初学者的数据分析 / 商业分析 / 商业智能系统化入门教程，目标是帮助想做数据但不知道从哪里开始的人，真正建立起数据分析师的核心能力。

内容不停留在工具操作和语法记忆，而是从数据思维出发，覆盖 SQL、Excel、Python、BI 可视化、商业分析，最终落到真实分析项目与求职实战——每篇文章都有真实业务场景、代码示例和面试真题。

**在线阅读（GitHub Pages）**：[https://onefly.top/zero2DA](https://onefly.top/zero2DA)

---

## 模块概览

| # | 模块 | 篇数 | 状态 | 核心内容 |
|---|------|------|------|---------|
| 01 | DA 基础 | 5 篇 | ✅ 完成 | 岗位认知、数据思维、BA/DA/BI 对比、行业差异 |
| 02 | 数据思维 | 5 篇 | ✅ 完成 | 假设驱动、指标体系、漏斗分析、A/B 测试、数据驱动决策 |
| 03 | SQL | 5 篇 | ✅ 完成 | 基础语法、JOIN/子查询、窗口函数、面试题型、数据库原理 |
| 04 | Excel | 4 篇 | ✅ 完成 | 数据清洗、透视表、常用公式、图表制作 |
| 05 | Python | 4 篇 | ✅ 完成 | Python 基础、Pandas、Matplotlib/Seaborn、实战项目 |
| 06 | 可视化 & BI | 4 篇 | ✅ 完成 | 可视化原则、Tableau、Power BI、数据看板设计 |
| 07 | 商业分析 | 4 篇 | ✅ 完成 | 商业模式、用户行为、营收分析、竞品分析 |
| 08 | DA 面试 | 10 篇 | ✅ 完成 | SQL 面试题（Easy/Medium/Hard）、Case Study、BA/DA 方向 |
| 09 | 工具箱 | — | 🔲 待开始 | 工具全景图、学习资源推荐 |
| 10 | 实战项目 | — | 🔲 待开始 | 完整分析项目，能放进简历的 Portfolio |

> **41 篇已发布**，总内容量约 470KB，持续更新中。

---

## 适合谁

- 想转行做数据分析，但不知道从哪里开始
- 应届生想入行 DA/BA/BI，需要系统化学习路线
- 运营/产品/市场背景，想掌握数据分析技能
- 准备数据分析师面试，需要 SQL 题库和 Case 练习
- 会写 SQL 但不会"用数据解决业务问题"

---

## 学习路线图

```
                    ┌─────────────┐
                    │  DA 基础     │  ← 岗位认知，建立正确预期
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  数据思维    │  ← 假设驱动、指标体系、A/B 测试
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
    │    SQL      │ │   Excel     │ │   Python    │  ← 三大硬技能
    └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
           │               │               │
           └───────────────┼───────────────┘
                           │
                    ┌──────▼──────┐
                    │ 可视化 & BI  │  ← Tableau / Power BI / 看板设计
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  商业分析    │  ← 从数据洞察到商业价值
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  DA 面试     │  ← SQL 题 + Case Study + 真题
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  实战项目    │  ← Portfolio 级别的完整项目
                    └─────────────┘
```

---

## 仓库结构

```
zero2DA/
├── index.html              # 首页
├── _config.yml             # Jekyll 配置
├── _layouts/
│   └── default.html        # 文章页模板
├── _data/
│   └── nav.yml             # 导航结构（10 模块 41 篇文章）
├── assets/
│   ├── css/
│   │   ├── style.css       # 首页样式
│   │   └── docs.css        # 文章页样式
│   ├── js/
│   │   └── app.js          # 侧边栏、目录、主题切换
│   └── images/
│       └── logo.svg        # 品牌 Logo
├── learn-da-basic/         # 01 DA 基础
├── learn-da-metrics/       # 02 数据思维
├── learn-da-sql/           # 03 SQL
├── learn-da-excel/         # 04 Excel
├── learn-da-python/        # 05 Python
├── learn-da-bi/            # 06 可视化 & BI
├── learn-da-business/      # 07 商业分析
├── learn-da-interview/     # 08 DA 面试
├── learn-da-tools/         # 09 工具箱
├── final-project/          # 10 实战项目
├── .github/workflows/      # GitHub Pages CI/CD
├── CNAME                   # 自定义域名
└── robots.txt              # SEO
```

---

## 本地运行

```bash
git clone https://github.com/ranxi2001/zero2DA
cd zero2DA

# 安装 Jekyll（需要 Ruby）
gem install bundler jekyll
bundle install

# 本地预览
bundle exec jekyll serve
# 访问 http://localhost:4000/zero2DA
```

---

## 技术栈

| 技术 | 用途 |
|------|------|
| Jekyll | 静态站点生成 |
| GitHub Pages | 托管与部署 |
| Kramdown | Markdown 渲染 |
| highlight.js | 代码语法高亮 |
| Mermaid | 流程图与图表 |
| Busuanzi | 访问量统计 |
| 纯 CSS + Vanilla JS | 样式与交互，零框架依赖 |

---

## Part of the Zero Series

| 项目 | 方向 | 链接 |
|------|------|------|
| **zero2DA** | 数据分析入门 | [onefly.top/zero2DA](https://onefly.top/zero2DA) |
| zero2PM | 产品经理入门 | [onefly.top/zero2PM](https://onefly.top/zero2PM) |
| zero2Agent | AI Agent 工程 | [onefly.top/zero2Agent](https://onefly.top/zero2Agent) |
| zero2Leetcode | 算法刷题 | [onefly.top/zero2Leetcode](https://onefly.top/zero2Leetcode) |

---

## Contributing

欢迎 PR 和 Issue。内容补充、错误修正、新模块建议均可。

---

## License

[MIT](LICENSE)
