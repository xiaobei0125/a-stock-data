# 📊 A股每日复盘存档

## 目录结构

```
reviews/
├── README.md           # 本文件 — 复盘存档总览
├── index.md            # 复盘索引 — 按日期快速检索
└── YYYY-MM-DD/         # 每日复盘文件夹
    ├── README.md        # 当日复盘摘要（快速浏览）
    ├── report.md        # 完整复盘报告（Markdown）
    ├── data/            # 原始数据
    │   ├── stocks.json          # 涨停股原始数据
    │   ├── index.json           # 指数行情
    │   ├── themes.json          # 题材归类结果
    │   └── dragon_tiger.json    # 龙虎榜数据
    └── charts/          # 图表（后续扩展）
```

## 复盘方法论

采用三步复盘法：涨停梳理 → 题材归类 → 龙头/先锋/跟风识别

详见 `~/.claude/skills/stock-review/SKILL.md`

## 快速开始

```bash
# 查看某日复盘
cat reviews/2026-06-23/report.md

# 查看复盘索引
cat reviews/index.md
```
