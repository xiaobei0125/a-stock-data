---
name: stock-review
description: A股每日复盘 — 三步法梳理涨停个股、题材归类、龙头/跟风/先锋识别，培养市场敏感度。基于 a-stock-data skill 的数据层。
origin: custom
version: 1.0.0
---

> 📦 项目主页：https://github.com/simonlin1212/a-stock-data — 依赖 a-stock-data skill
>
> 作者：Simon 林 · 抖音「Simon林」· 公众号「硅基世纪」

# A股每日复盘工具包 V1.0

三步复盘法：涨停梳理 → 题材归类 → 龙头/先锋/跟风识别。

## 复盘三步法概述

```
Step 1: 梳理全市场涨停个股
  ├── 获取当日所有涨停/触及涨停的股票
  ├── 挖掘每只股票的上涨逻辑（reason tags）
  └── 标注封板特征（一字板 / T字板 / 换手板）

Step 2: 题材归类
  ├── 对涨停股按上涨逻辑进行归类
  ├── 归并相似原因 → 形成题材组
  ├── 按涨停数 + 总成交额 + 平均涨幅 → 题材强度排序
  └── 清晰区分当日强势题材 vs 弱势题材

Step 3: 题材内角色定位
  ├── 🐉 龙头：题材内涨幅最高 + 封板最紧（低换手 / 一字板）
  ├── 🚀 先锋：题材内最先涨停（一字板优先 → 低换手 → 高量比）
  └── 📈 跟风：其余同题材个股

核心目的：培养对市场行情的敏锐判断能力。
```

---

## When to Activate

- 用户要做 A 股每日复盘
- 用户要分析涨停板 / 涨停原因
- 用户要研究当日题材热点 / 主线
- 用户要识别龙头股 / 跟风股
- 用户要看市场情绪 / 赚钱效应
- 关键词：**复盘、涨停、题材、龙头、跟风、先锋、涨停复盘、每日复盘、涨停分析、题材分析、市场复盘、强势股、封板、一字板、换手板、市场情绪、赚钱效应**

---

## 数据源优先级 & 东财防封

| 优先级 | 数据源 | 用途 | 风控 |
|--------|--------|------|------|
| **1** | **同花顺热点** | 涨停股 + reason tags（主力数据源） | 极低（零鉴权） |
| **2** | **腾讯财经** | 指数行情 + 个股实时价 | 低 |
| **3** | **东财 datacenter** | 龙虎榜 + 行业排名 | 低 |
| **4** | **东财 push2** | 概念板块归属（slist）+ 个股信息 | 有风控 |
| **5** | **东财 push2 clist** | 全市场涨跌幅扫描（兜底补充） | ⚠️ 易触发 502 |

> **主力数据源是同花顺热点**（`ths_hot_reason`），它一次返回 ~120 只当日强势股，其中 ~115 只是涨停股，且每只都带上涨原因 tags。这在绝大多数复盘场景已经足够。东财 clist 全市场扫描仅在需要确保"一只不漏"时启用。

### 东财防封铁律

所有 eastmoney.com 请求走 `em_get()`（定义见下方 Prerequisites），串行限流（间隔 ≥1s + 随机抖动），复用 session。

---

## Prerequisites

依赖 a-stock-data skill 中定义的数据获取函数。核心依赖已在 a-stock-data 中安装：

```bash
pip install mootdx requests pandas
```

### 共用 Helper：东财节流入口（从 a-stock-data 继承）

```python
import time
import random
import requests

UA = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36"

EM_SESSION = requests.Session()
EM_SESSION.headers.update({"User-Agent": UA})
EM_MIN_INTERVAL = 1.0
_em_last_call = [0.0]

def em_get(url: str, params: dict | None = None, headers: dict | None = None,
           timeout: int = 15, **kwargs):
    """东财统一请求入口：自动节流 + 复用 session"""
    wait = EM_MIN_INTERVAL - (time.time() - _em_last_call[0])
    if wait > 0:
        time.sleep(wait + random.uniform(0.1, 0.5))
    try:
        return EM_SESSION.get(url, params=params, headers=headers, timeout=timeout, **kwargs)
    finally:
        _em_last_call[0] = time.time()

def eastmoney_datacenter(report_name: str, columns: str = "ALL",
                          filter_str: str = "", page_size: int = 50,
                          sort_columns: str = "", sort_types: str = "-1") -> list[dict]:
    """东财数据中心统一查询"""
    params = {
        "reportName": report_name, "columns": columns,
        "filter": filter_str, "pageNumber": "1", "pageSize": str(page_size),
        "sortColumns": sort_columns, "sortTypes": sort_types,
        "source": "WEB", "client": "WEB",
    }
    r = em_get("https://datacenter-web.eastmoney.com/api/data/v1/get",
               params=params, timeout=15)
    d = r.json()
    if d.get("result") and d["result"].get("data"):
        return d["result"]["data"]
    return []
```

---

## Step 1: 全市场涨停股获取 + 上涨逻辑挖掘

### 1.1 主力数据源：同花顺热点（涨停股 + reason tags）

**核心价值：** 一次 HTTP GET 返回 ~120 只当日强势股（其中 ~115 只涨停），每只带人工运营的上涨原因 tags，是复盘的核心数据源。

```python
import requests
from datetime import date as _date

def fetch_limit_up_stocks(date_str: str = None) -> list[dict]:
    """
    Step 1 核心: 获取当日所有涨停股及上涨逻辑。
    主力: 同花顺热点 API（~115 只涨停 + reason tags）。
    返回: [{code, name, change_pct, turnover, amount, reason, limit_type, board, ...}]
    """
    if date_str is None:
        date_str = _date.today().strftime("%Y-%m-%d")

    # ── 1. 同花顺热点（主力数据源）──
    url = (
        f"http://zx.10jqka.com.cn/event/api/getharden/"
        f"date/{date_str}/orderby/date/orderway/desc/charset/GBK/"
    )
    headers = {
        "User-Agent": (
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
            "Chrome/117.0.0.0 Safari/537.36"
        )
    }
    r = requests.get(url, headers=headers, timeout=10)
    data = r.json()
    if data.get("errocode", 0) != 0:
        raise RuntimeError(f"同花顺热点错误: {data.get('errormsg', '')}")

    rows = data.get("data") or []

    # ── 2. 涨停阈值判断 ──
    def _limit_threshold(code: str, name: str) -> float:
        """返回涨停所需的最小涨跌幅（考虑各板块差异）"""
        if "ST" in name or "*ST" in name:
            return 4.9
        if code.startswith("688"):                      # 科创板 20%
            return 19.9
        if code.startswith(("300", "301")):             # 创业板 20%
            return 19.9
        if code.startswith(("8", "4")):                 # 北交所 30%
            return 29.9
        return 9.9                                       # 主板 10%

    # ── 3. 涨停分类 + 封板特征推断 ──
    stocks = []
    for row in rows:
        code = str(row.get("code", ""))
        name = str(row.get("name", ""))
        chg = float(row.get("zhangfu", 0))
        threshold = _limit_threshold(code, name)

        if chg < threshold:
            continue  # 非涨停股跳过

        reason = row.get("reason", "") or "未标注"
        turnover = float(row.get("huanshou", 0) or 0)      # 换手率%
        amount = float(row.get("chengjiaoe", 0) or 0)       # 成交额(元)
        close_p = float(row.get("close", 0) or 0)           # 收盘价
        dde = float(row.get("ddejingliang", 0) or 0)        # 大单净量

        # ── 封板特征推断 ──
        # 因为没有盘中的开/高/低数据（同花顺热点只给收盘价/涨跌幅/换手），
        # 封板特征基于换手率 + reason 中的"一字板"等关键词推断：
        seal_type = _infer_seal_type(name, reason, chg, turnover)

        stocks.append({
            "code": code,
            "name": name,
            "change_pct": chg,
            "turnover": turnover,
            "amount": amount,
            "close": close_p,
            "dde_net": dde,
            "reason_raw": reason,
            "reason_tags": [t.strip() for t in reason.split("+") if t.strip()],
            "limit_type": _limit_board_type(code),
            "seal_type": seal_type,               # 一字板 / T字板 / 换手板 / 烂板
            "board": _classify_board(code),
        })

    # ── 4. 补充腾讯行情数据（今开/最高/最低，用于精确封板判断）──
    _enrich_with_tencent(stocks)

    return stocks


def _limit_board_type(code: str) -> str:
    """涨停板类型"""
    if code.startswith("688"):
        return "科创板(20%)"
    if code.startswith(("300", "301")):
        return "创业板(20%)"
    if code.startswith(("8", "4")):
        return "北交所(30%)"
    return "主板(10%)"


def _classify_board(code: str) -> str:
    """所属交易板块"""
    if code.startswith("688"):
        return "科创板"
    if code.startswith(("300", "301")):
        return "创业板"
    if code.startswith(("8", "4")):
        return "北交所"
    if code.startswith("6"):
        return "沪市主板"
    return "深市主板"


def _infer_seal_type(name: str, reason: str, chg: float, turnover: float) -> str:
    """
    基于可用数据推断封板特征。
    同花顺热点不提供盘中开/高/低，这里用换手率做近似推断：
      - 换手率 < 3%  → 一字板 / 强封板（封板极紧）
      - 换手率 3-8%  → T字板 / 中等封板
      - 换手率 8-15% → 换手板（正常封板）
      - 换手率 > 15% → 烂板 / 分歧板（封板有分歧）
    """
    # reason 中有时会说明封板类型
    if "一字" in reason:
        return "一字板"
    if "T字" in reason or "t字" in reason:
        return "T字板"

    if turnover < 3:
        return "一字板/强封"
    elif turnover < 8:
        return "T字板/中等"
    elif turnover < 15:
        return "换手板"
    else:
        return "烂板/分歧"


def _enrich_with_tencent(stocks: list[dict]):
    """用腾讯财经补充今开/最高/最低，精确判断封板特征"""
    if not stocks:
        return

    import urllib.request

    # 分批查询（腾讯一次支持多只）
    batch_size = 50
    for i in range(0, len(stocks), batch_size):
        batch = stocks[i:i+batch_size]
        prefixed = []
        code_map = {}
        for s in batch:
            code = s["code"]
            prefix = "sh" if code.startswith(("6", "9")) else (
                "bj" if code.startswith(("8", "4")) else "sz"
            )
            key = f"{prefix}{code}"
            prefixed.append(key)
            code_map[key] = s

        url = "https://qt.gtimg.cn/q=" + ",".join(prefixed)
        req = urllib.request.Request(url)
        req.add_header("User-Agent", "Mozilla/5.0")
        try:
            resp = urllib.request.urlopen(req, timeout=10)
            data = resp.read().decode("gbk")
        except Exception:
            continue

        for line in data.strip().split(";"):
            if "=" not in line or '"' not in line:
                continue
            key = line.split("=")[0].split("_")[-1]
            s = code_map.get(key)
            if not s:
                continue
            vals = line.split('"')[1].split("~")
            if len(vals) < 35:
                continue
            try:
                s["open"] = float(vals[5]) if vals[5] else 0
                s["high"] = float(vals[33]) if vals[33] else 0
                s["low"] = float(vals[34]) if vals[34] else 0
                s["prev_close"] = float(vals[4]) if vals[4] else 0
                s["volume_ratio"] = float(vals[49]) if vals[49] else 0

                # ── 精确封板判断 ──
                open_p = s.get("open", 0)
                high = s.get("high", 0)
                low = s.get("low", 0)
                close_p = s.get("close", 0)
                prev = s.get("prev_close", 0)

                if prev > 0 and close_p > 0:
                    limit_price = _calc_limit_price(s["code"], s["name"], prev)
                    # 一字板：开=高=低=收=涨停价
                    if (abs(open_p - limit_price) < 0.01 and
                        abs(high - limit_price) < 0.01 and
                        abs(low - limit_price) < 0.01):
                        s["seal_type"] = "一字板 🔒"
                    # T字板：开=涨停 但低 < 涨停
                    elif (abs(open_p - limit_price) < 0.01 and
                          abs(close_p - limit_price) < 0.01 and
                          low < limit_price - 0.01):
                        s["seal_type"] = "T字板 ⚡"
                    # 实体板：开 < 涨停 收 = 涨停
                    elif (abs(close_p - limit_price) < 0.01 and
                          open_p < limit_price - 0.01):
                        s["seal_type"] = "实体板 📈"
                    # 触及但未封死
                    elif high >= limit_price - 0.01 and close_p < limit_price - 0.01:
                        s["seal_type"] = "炸板 💥"
            except (ValueError, IndexError):
                pass


def _calc_limit_price(code: str, name: str, prev_close: float) -> float:
    """计算涨停价（四舍五入到分）"""
    if "ST" in name or "*ST" in name:
        rate = 0.05
    elif code.startswith("688") or code.startswith(("300", "301")):
        rate = 0.20
    elif code.startswith(("8", "4")):
        rate = 0.30
    else:
        rate = 0.10
    return round(prev_close * (1 + rate), 2)


# 用法
stocks = fetch_limit_up_stocks("2026-06-23")
print(f"当日涨停股: {len(stocks)} 只")

# 按封板类型统计
from collections import Counter
seal_counts = Counter(s["seal_type"] for s in stocks)
print("封板类型分布:", seal_counts)

# 打印涨停股列表
for s in stocks[:10]:
    print(f"  {s['code']} {s['name']}: {s['change_pct']}% | {s['seal_type']} | {s['reason_raw'][:60]}")
```

### 1.2 兜底数据源：东财 clist 全市场扫涨跌幅

当需要"一只不漏"时启用（东财 clist 服务器不稳定，容易 502）。

```python
def fetch_limit_up_clist_fallback(date_str: str = None) -> list[dict]:
    """
    兜底: 东财 push2 clist 全市场涨跌幅扫描。
    ⚠️ 注意: clist 服务器不稳定，容易返回 502。优先使用 fetch_limit_up_stocks()。
    返回: [{code, name, change_pct, turnover, ...}]（无 reason tags）
    """
    import urllib.request
    import json

    if date_str is None:
        date_str = _date.today().strftime("%Y-%m-%d")

    # 东财 clist 五大板块分页扫（深A + 创业板 + 沪A + 科创板 + 北交所）
    all_items = []
    for fs_code in ["m:0+t:6", "m:0+t:80", "m:1+t:2", "m:1+t:23", "m:0+t:81"]:
        for page in [1, 2]:  # 每板块前 200 只（pz=100 × 2页）
            url = (
                f"http://push2.eastmoney.com/api/qt/clist/get"
                f"?pn={page}&pz=100&po=1&np=1&fltt=2&invt=2"
                f"&fs={fs_code}"
                f"&fields=f2,f3,f4,f5,f6,f8,f10,f12,f14,f15,f16,f17,f18"
            )
            req = urllib.request.Request(url)
            req.add_header("User-Agent", UA)
            try:
                resp = urllib.request.urlopen(req, timeout=15)
                d = json.loads(resp.read().decode("utf-8"))
                items = d.get("data", {}).get("diff", [])
                all_items.extend(items)
            except Exception as e:
                print(f"[WARN] clist {fs_code} page {page} 失败: {e}")
            time.sleep(1.5)  # 严格限流

    # 涨停过滤（同 Step 1 逻辑）
    stocks = []
    for it in all_items:
        code = str(it.get("f12", ""))
        name = str(it.get("f14", ""))
        chg = float(it.get("f3", 0) or 0)
        threshold = 9.9
        if "ST" in name:
            threshold = 4.9
        elif code.startswith("688") or code.startswith(("300", "301")):
            threshold = 19.9
        elif code.startswith(("8", "4")):
            threshold = 29.9
        if chg < threshold:
            continue
        stocks.append({
            "code": code, "name": name,
            "change_pct": chg,
            "turnover": float(it.get("f8", 0) or 0),
            "amount": float(it.get("f6", 0) or 0),
            "close": float(it.get("f2", 0) or 0),
            "open": float(it.get("f17", 0) or 0),
            "high": float(it.get("f15", 0) or 0),
            "low": float(it.get("f16", 0) or 0),
            "prev_close": float(it.get("f18", 0) or 0),
            "volume_ratio": float(it.get("f10", 0) or 0),
            "reason_raw": "",
            "reason_tags": [],
            "limit_type": _limit_board_type(code),
            "seal_type": "未知",
            "board": _classify_board(code),
        })
    return stocks
```

---

## Step 2: 题材归类

### 2.1 基于 reason tags 的自动归类

```python
from collections import Counter, defaultdict

def classify_themes(stocks: list[dict]) -> dict[str, dict]:
    """
    Step 2: 对涨停股按上涨逻辑归类，形成题材组。
    输入: fetch_limit_up_stocks() 的返回值
    返回: {theme_name: {stocks, strength, avg_change, total_amount, summary}}
    """

    # ── 1. 提取所有 reason tags，建立题材关键词映射 ──
    # reason tags 形如: "钨钽铌资产注入+磁选装备+江西国资"
    # 同花顺的 tags 已经是很好的题材标注，可直接按首位 tag 或语义相近的 tag 分组

    # 策略: 按首位（最核心）tag + 关键词合并归类
    theme_map = defaultdict(list)

    # 题材关键词合并规则（同一概念的不同表述 → 合并）
    # ⚠️ 这是初始规则集。同花顺 tags 是人工编辑的，新概念会用新表述。
    # 建议每次复盘后根据实际遇到的 tags 持续丰富此字典。
    # Claude 做最终语义合并时会自动处理未覆盖的新词。
    TAG_MERGE_RULES = {
        # ── 科技主线 ──
        "AI": [
            "智谱AI", "人工智能", "大模型", "GPT", "AI应用", "AI算力",
            "AI金融", "AI机器视觉", "AI设计", "AI制药", "AI医疗",
            "AI智算中心电源", "AI数据中心燃气轮机", "具身智能",
        ],
        "机器人": [
            "机器人", "人形机器人", "机器狗", "电子皮肤", "XT减速器",
            "机器人电子皮肤",
        ],
        "算力": [
            "算力", "算力租赁", "算力温控", "算力光纤", "数据中心",
            "服务器电源", "IDC并表",
        ],
        "半导体": [
            "碳化硅", "光通讯", "CPO", "MPO", "光连接器", "芯片",
            "模拟芯片", "功率芯片", "MCU芯片", "芯片电感",
            "光刻胶基材", "玻璃基板封装", "玻璃晶圆", "功率半导体",
            "电子陶瓷", "光耦合器", "覆铜板", "收购半导体",
            "空芯光纤", "光纤", "光缆", "光纤油膏", "激光通信",
        ],
        # ── 资源/材料 ──
        "新材料": [
            "锆材料", "新材料", "钨", "锆", "氧氯化锆", "稀土", "磁材",
            "磷化铟", "氧化锆", "钨钽铌", "磁选装备", "含硫硅烷",
            "电子级硅基材料", "高纯四氯化硅",
        ],
        "化工": [
            "化工", "涂料", "防腐", "湿电子化学品", "电子特气",
            "PCB用化学试剂", "精细化工", "医药中间体",
        ],
        # ── 新能源 ──
        "新能源": [
            "固态电池", "锂电池", "光伏", "储能", "新能源汽车",
            "页岩油", "新能源驱动电机", "新能源材料",
            "汽车零部件", "汽车结构件", "汽车内饰", "汽车电子",
            "热管理", "核聚变",
        ],
        # ── 医药/消费 ──
        "医药": [
            "创新药", "阿尔茨海默", "原料药", "CXO", "医药",
            "止咳宝片", "集采中标", "布洛芬", "光动力疗法",
            "骨科中药", "AI制药", "AI医疗", "医疗耗材",
            "硝酸甘油", "降血脂药", "解热镇痛", "化学制药",
            "中成药", "CPHI展会", "口腔修复",
        ],
        "消费": [
            "消费", "白酒", "食品", "零售", "电商", "预制菜",
            "集成灶", "葡萄酒", "生猪养殖", "肉牛养殖",
            "宠物", "造纸", "洗护", "纺织", "职业装",
            "家装", "家居", "装饰原纸", "包装材料",
            "酒店管理", "物业管理", "社区商业",
        ],
        # ── 金融/周期 ──
        "券商金融": [
            "券商", "金融科技", "银行", "保险", "REITs",
            "AI金融", "金融", "回购",
        ],
        "国企改革": [
            "国企改革", "央企", "国资", "资产注入", "控制权变更",
            "控制权拟变更", "国资入主", "摘帽", "重整",
        ],
        # ── 特殊题材 ──
        "ST摘帽": ["摘帽", "申请撤销ST", "ST整改", "ST"],
        "军工": ["军工", "航天", "导弹", "军工电子"],
        "地产基建": [
            "地产", "基建", "水利", "生态", "建筑",
            "生态水利", "工业污水处理", "污水处理",
        ],
        "通信": [
            "光纤", "光缆", "MPO", "CPO", "光通讯",
            "空芯光纤", "激光通信", "光连接器",
        ],
    }

    def _normalize_tag(tag: str) -> str:
        """将 raw tag 映射到标准化题材名"""
        for theme, aliases in TAG_MERGE_RULES.items():
            for alias in aliases:
                if alias in tag:
                    return theme
        return tag  # 未匹配的保留原始 tag

    for s in stocks:
        tags = s.get("reason_tags") or ["未分类"]
        primary_tag = tags[0] if tags else "未分类"
        theme = _normalize_tag(primary_tag)
        theme_map[theme].append(s)

    # ── 2. 计算每个题材的强度指标 ──
    theme_analysis = {}
    for theme, group in theme_map.items():
        n = len(group)
        avg_chg = sum(s["change_pct"] for s in group) / n if n else 0
        total_amt = sum(s.get("amount", 0) for s in group)
        max_chg = max(s["change_pct"] for s in group) if n else 0
        min_turnover = min(s.get("turnover", 999) for s in group) if n else 0

        # 题材强度评级
        if n >= 8:
            strength = "🔥🔥🔥 超级主线"
        elif n >= 5:
            strength = "🔥🔥 强势题材"
        elif n >= 3:
            strength = "🔥 活跃题材"
        else:
            strength = "💤 零散题材"

        theme_analysis[theme] = {
            "strength": strength,
            "count": n,
            "avg_change": round(avg_chg, 2),
            "total_amount": total_amt,
            "max_change": round(max_chg, 2),
            "min_turnover": round(min_turnover, 2),
            "stocks": sorted(group, key=lambda x: x["change_pct"], reverse=True),
        }

    # ── 3. 按涨停数降序排列（强势题材在前）──
    theme_analysis = dict(
        sorted(theme_analysis.items(), key=lambda x: x[1]["count"], reverse=True)
    )

    return theme_analysis


# 用法
stocks = fetch_limit_up_stocks("2026-06-23")
themes = classify_themes(stocks)

print("=== 当日题材强度排名 ===")
for theme, info in themes.items():
    print(f"{info['strength']} {theme}: {info['count']}只涨停 | "
          f"均价涨幅{info['avg_change']}% | 总成交{info['total_amount']/1e8:.0f}亿")

# 查看各题材的涨停股
for theme, info in themes.items():
    if info["count"] >= 3:
        print(f"\n📊 {theme} ({info['count']}只):")
        for s in info["stocks"]:
            print(f"  {s['code']} {s['name']}: +{s['change_pct']}% | {s['seal_type']} | {s['reason_raw'][:60]}")
```

### 2.2 题材合并规则扩展

```python
def update_tag_merge_rules(custom_rules: dict[str, list[str]]):
    """
    自定义题材关键词合并规则。
    调用此函数可以添加新的合并规则或覆盖默认规则。
    
    例: update_tag_merge_rules({"低空经济": ["飞行汽车", "eVTOL", "无人机", "低空"]})
    """
    global TAG_MERGE_RULES
    TAG_MERGE_RULES.update(custom_rules)
```

> **建议：** 首次使用后，根据实际复盘结果中出现的 tags 持续丰富 `TAG_MERGE_RULES`。同花顺的 tags 是人工编辑的，新概念出现时会用新表述，需要定期更新合并规则。这也是培养盘感的过程——你会逐渐熟悉市场的题材语言。

### 2.3 AI 语义合并（Claude 自动执行）

`classify_themes()` 基于关键词规则做**硬归类**，会产生一些"零散题材"（单只或两只涨停的股票）。Claude 在生成复盘报告时应做**语义合并**——阅读每个零散题材的 `reason_raw`，判断是否可以并入现有大题材或合并成新主题。常见合并场景：

| 拆分现象 | 应合并到 | 理由 |
|----------|----------|------|
| "布洛芬" vs "创新药" | → 医药 | "布洛芬"是具体药品名，属医药大类 |
| "液冷阀门" vs "液冷" | → 液冷 | 阀门是液冷系统的组件 |
| "AI制药" vs "医药" | → 医药 | AI制药本质是医药研发，非AI主线 |
| "光刻胶基材" vs "半导体" | → 半导体 | 光刻胶是半导体上游材料 |
| "高纯四氯化硅涨价" vs "新材料" | → 新材料 | 可自行判断归材料 or 半导体 |
| "算力租赁" vs "算力" | → 算力 | 同属算力大题材 |
| "AI智算中心电源" vs "算力" | → 算力 | 智算中心属算力基础设施 |

> 这个语义合并过程本身就是复盘的核心训练——你会逐渐建立自己的题材框架，而不是被动接受标签。

---

## Step 3: 题材内角色定位（龙头/先锋/跟风）

### 3.1 角色识别算法

```python
def identify_roles(theme_stocks: list[dict]) -> list[dict]:
    """
    Step 3: 在同一题材内区分龙头、先锋、跟风。
    输入: 已排序的题材内涨停股列表（按涨幅降序）
    返回: 标注了 role 字段的股票列表，排序: 龙头 → 先锋 → 跟风
    """

    if not theme_stocks:
        return []

    # ── 1. 龙头识别 ──
    # 规则: 涨幅最高者优先；如涨幅相同 → 换手率最低者（封板更紧）→ 量比最高者（买盘更猛）
    candidates = sorted(theme_stocks, key=lambda s: (
        -s["change_pct"],
        s.get("turnover", 999),
        -(s.get("volume_ratio", 0) or 0),
    ))
    leader = candidates[0]

    # ── 2. 先锋识别 ──
    # 规则: 一字板优先 → 换手率最低 → 量比最高
    # 先锋 = 题材中最早涨停的股票（通常是那个一字板 or 最低换手的）
    remaining = [s for s in candidates if s["code"] != leader["code"]]
    pioneer_candidates = sorted(remaining, key=lambda s: (
        0 if "一字板" in s.get("seal_type", "") else (
            1 if "T字板" in s.get("seal_type", "") else 2
        ),
        s.get("turnover", 999),
        -(s.get("volume_ratio", 0) or 0),
    ))
    pioneer = pioneer_candidates[0] if pioneer_candidates else None

    # ── 3. 跟风 ──
    follower_codes = {leader["code"]}
    if pioneer:
        follower_codes.add(pioneer["code"])
    followers = [s for s in candidates if s["code"] not in follower_codes]

    # ── 4. 标注角色 ──
    leader["role"] = "🐉 龙头"
    leader["role_detail"] = _role_reason(leader, "leader")

    if pioneer:
        pioneer["role"] = "🚀 先锋"
        pioneer["role_detail"] = _role_reason(pioneer, "pioneer")

    for s in followers:
        s["role"] = "📈 跟风"
        s["role_detail"] = ""

    # 组装排序后的列表
    result = [leader]
    if pioneer:
        result.append(pioneer)
    result.extend(followers)
    return result


def _role_reason(stock: dict, role: str) -> str:
    """生成角色判定理由"""
    reasons = []
    seal = stock.get("seal_type", "")
    turnover = stock.get("turnover", 0)
    chg = stock.get("change_pct", 0)
    vol_ratio = stock.get("volume_ratio", 0) or 0

    if role == "leader":
        reasons.append(f"涨幅最高({chg}%)")
        if "一字板" in seal:
            reasons.append("一字板封死")
        elif "T字板" in seal:
            reasons.append("T字板强封")
        if turnover < 5:
            reasons.append(f"换手极低({turnover}%)封板极紧")
    elif role == "pioneer":
        if "一字板" in seal:
            reasons.append("一字板疑似最早涨停")
        if turnover < 5:
            reasons.append(f"低换手({turnover}%)预示早期封板")
        if vol_ratio > 2:
            reasons.append(f"量比{vol_ratio}倍资金抢筹")

    return " + ".join(reasons) if reasons else ""


# 用法（结合 Step 2 的输出）
themes = classify_themes(stocks)
for theme, info in themes.items():
    if info["count"] >= 3:
        print(f"\n=== {theme} ({info['count']}只) ===")
        role_assigned = identify_roles(info["stocks"])
        for s in role_assigned:
            print(f"  {s['role']} | {s['code']} {s['name']} | "
                  f"+{s['change_pct']}% | {s['seal_type']} | {s.get('role_detail', '')}")
```

### 3.2 角色定义速查

| 角色 | 图标 | 特征 | 意义 |
|------|------|------|------|
| **龙头** | 🐉 | 题材内涨幅最高 + 封板最紧（低换手/一字板） | 市场公认的题材代表，次日溢价最高 |
| **先锋** | 🚀 | 一字板或极低换手，率先封死涨停 | 真正带节奏的资金，可能比龙头更早涨停 |
| **跟风** | 📈 | 换手板居多，涨幅略低于或等于龙头 | 跟随题材情绪上涨，次日分化概率大 |
| **炸板** | 💥 | 触及涨停但未封死（收盘未涨停） | 追高风险极大，次日大概率低开 |

---

## 完整复盘报告生成

### 4.1 主函数：generate_review_report()

```python
def generate_review_report(date_str: str = None) -> str:
    """
    一键生成 A 股每日复盘 Markdown 报告。
    涵盖: 市场总览 → 涨停全表 → 题材分析 → 行业轮动 → 复盘总结
    """
    from datetime import date as _date
    import urllib.request

    if date_str is None:
        date_str = _date.today().strftime("%Y-%m-%d")

    # ── 数据采集 ──
    print(f"[1/5] 拉取涨停股数据...")
    stocks = fetch_limit_up_stocks(date_str)

    print(f"[2/5] 拉取指数行情...")
    index_data = _fetch_index_quotes()

    print(f"[3/5] 拉取行业排名...")
    industry_data = _fetch_industry_rankings()

    print(f"[4/5] 拉取龙虎榜...")
    dragon_data = _fetch_daily_dragon_tiger(date_str)

    print(f"[5/5] 题材归类 + 角色识别...")
    themes = classify_themes(stocks)

    # ── 报告组装 ──
    report = _build_report(date_str, stocks, themes, index_data, industry_data, dragon_data)
    return report


def _fetch_index_quotes() -> dict:
    """拉取主要指数行情"""
    import urllib.request

    index_map = {
        "sh000001": "上证指数", "sz399001": "深证成指",
        "sz399006": "创业板指", "sh000688": "科创50",
        "sh000300": "沪深300", "sz399303": "国证2000",
    }
    url = "https://qt.gtimg.cn/q=" + ",".join(index_map.keys())
    req = urllib.request.Request(url)
    req.add_header("User-Agent", "Mozilla/5.0")
    resp = urllib.request.urlopen(req, timeout=10)
    data = resp.read().decode("gbk")

    result = {}
    for line in data.strip().split(";"):
        if "=" not in line or '"' not in line:
            continue
        key = line.split("=")[0].split("_")[-1]
        name = index_map.get(key)
        if not name:
            continue
        vals = line.split('"')[1].split("~")
        if len(vals) < 45:
            continue
        result[name] = {
            "price": float(vals[3]) if vals[3] else 0,
            "change_pct": float(vals[32]) if vals[32] else 0,
            "change_amt": float(vals[31]) if vals[31] else 0,
            "amount_yi": float(vals[37]) / 10000 if vals[37] else 0,  # 亿
        }
    return result


def _fetch_industry_rankings(top_n: int = 15) -> dict:
    """拉取行业板块涨跌排名"""
    url = "https://push2.eastmoney.com/api/qt/clist/get"
    params = {
        "pn": "1", "pz": "100", "po": "1", "np": "1",
        "fltt": "2", "invt": "2",
        "fs": "m:90+t:2",
        "fields": "f2,f3,f4,f12,f14,f104,f105,f128,f136,f140",
    }
    try:
        r = em_get(url, params=params, timeout=15)
        d = r.json()
        items = d.get("data", {}).get("diff", [])
    except Exception:
        return {"top": [], "bottom": [], "total": 0}

    rows = []
    for i, it in enumerate(items):
        rows.append({
            "rank": i + 1,
            "name": it.get("f14", ""),
            "change_pct": it.get("f3", 0),
            "up_count": it.get("f104", 0),
            "down_count": it.get("f105", 0),
            "leader": it.get("f140", ""),
            "leader_change": it.get("f136", 0),
        })
    return {
        "top": rows[:top_n],
        "bottom": rows[-top_n:] if len(rows) >= top_n else [],
        "total": len(rows),
    }


def _fetch_daily_dragon_tiger(date_str: str) -> list[dict]:
    """拉取当日龙虎榜"""
    data = eastmoney_datacenter(
        "RPT_DAILYBILLBOARD_DETAILSNEW",
        filter_str=f"(TRADE_DATE>='{date_str}')(TRADE_DATE<='{date_str}')",
        page_size=200,
        sort_columns="BILLBOARD_NET_AMT", sort_types="-1",
    )
    stocks = []
    for row in data:
        stocks.append({
            "code": row.get("SECURITY_CODE", ""),
            "name": row.get("SECURITY_NAME_ABBR", ""),
            "reason": row.get("EXPLANATION", ""),
            "net_buy_wan": round((row.get("BILLBOARD_NET_AMT") or 0) / 10000, 1),
            "change_pct": round(float(row.get("CHANGE_RATE") or 0), 2),
            "turnover_pct": round(float(row.get("TURNOVERRATE") or 0), 2),
        })
    return stocks


def _build_report(date_str: str, stocks: list[dict], themes: dict,
                   index_data: dict, industry_data: dict,
                   dragon_data: list[dict]) -> str:
    """组装完整复盘 Markdown 报告"""

    # ── 市场情绪判断 ──
    limit_up_count = len(stocks)
    if limit_up_count >= 80:
        sentiment = "🔥🔥 极度亢奋"
    elif limit_up_count >= 50:
        sentiment = "🔥 强势"
    elif limit_up_count >= 30:
        sentiment = "✅ 温和"
    elif limit_up_count >= 10:
        sentiment = "❄️ 偏冷"
    else:
        sentiment = "🧊 冰点"

    lines = []
    lines.append(f"# 📊 A股每日复盘报告 — {date_str}")
    lines.append("")

    # ── 一、市场总览 ──
    lines.append("## 一、市场总览")
    lines.append("")
    lines.append("### 主要指数")
    lines.append("")
    lines.append("| 指数 | 收盘 | 涨跌幅 | 涨跌额 | 成交额(亿) |")
    lines.append("|------|------|--------|--------|------------|")
    for name, info in index_data.items():
        lines.append(
            f"| {name} | {info['price']:.2f} | "
            f"{info['change_pct']:+.2f}% | {info['change_amt']:+.2f} | "
            f"{info['amount_yi']:.0f} |"
        )
    lines.append("")
    lines.append(f"> 📊 今日涨停: **{limit_up_count}** 只 | 市场情绪: **{sentiment}**")
    lines.append("")

    # ── 板块统计 ──
    board_counts = Counter(s["board"] for s in stocks)
    lines.append("### 涨停板块分布")
    lines.append("")
    board_str = " | ".join(f"{board}: {cnt}只" for board, cnt in board_counts.most_common())
    lines.append(f"> {board_str}")
    lines.append("")

    # ── 二、涨停个股全表 ──
    lines.append("---")
    lines.append("")
    lines.append("## 二、涨停个股全表")
    lines.append("")
    lines.append("| # | 代码 | 名称 | 涨幅% | 换手% | 封板特征 | 上涨原因 |")
    lines.append("|---|------|------|-------|-------|----------|----------|")
    for i, s in enumerate(stocks, 1):
        reason_short = s["reason_raw"][:50] if s["reason_raw"] else "-"
        lines.append(
            f"| {i} | {s['code']} | {s['name']} | "
            f"+{s['change_pct']:.2f}% | {s.get('turnover', 0):.1f}% | "
            f"{s.get('seal_type', '-')} | {reason_short} |"
        )
    lines.append("")

    # ── 三、题材分析 ──
    lines.append("---")
    lines.append("")
    lines.append("## 三、题材分析")
    lines.append("")

    strong_themes = {k: v for k, v in themes.items() if v["count"] >= 3}
    weak_themes = {k: v for k, v in themes.items() if v["count"] < 3}

    # 强势/活跃题材（≥3 只涨停）
    for theme, info in strong_themes.items():
        role_assigned = identify_roles(info["stocks"])
        lines.append(f"### {info['strength']} {theme} ({info['count']}只涨停)")
        lines.append("")
        lines.append(f"> 均价涨幅: **{info['avg_change']}%** | "
                     f"总成交: **{info['total_amount']/1e8:.1f}亿** | "
                     f"最高涨幅: **{info['max_change']}%**")
        lines.append("")
        lines.append("| 角色 | 代码 | 名称 | 涨幅% | 换手% | 封板特征 | 判定理由 |")
        lines.append("|------|------|------|-------|-------|----------|----------|")
        for s in role_assigned:
            lines.append(
                f"| {s.get('role', '')} | {s['code']} | {s['name']} | "
                f"+{s['change_pct']:.2f}% | {s.get('turnover', 0):.1f}% | "
                f"{s.get('seal_type', '-')} | {s.get('role_detail', '-')} |"
            )
        lines.append("")

        # 上涨逻辑分析（基于 reason tags）
        all_tags = []
        for s in info["stocks"]:
            all_tags.extend(s.get("reason_tags", []))
        tag_freq = Counter(all_tags).most_common(8)
        tag_str = " → ".join(f"{t}({n}只)" for t, n in tag_freq[:5])
        lines.append(f"**💡 上涨逻辑链**: {tag_str}")
        lines.append("")

    # 零散题材（< 3 只涨停）
    if weak_themes:
        lines.append("### 💤 零散题材")
        lines.append("")
        lines.append("| 题材 | 涨停数 | 代表股 |")
        lines.append("|------|--------|--------|")
        for theme, info in weak_themes.items():
            reps = ", ".join(f"{s['code']} {s['name']}" for s in info["stocks"][:2])
            lines.append(f"| {theme} | {info['count']} | {reps} |")
        lines.append("")

    # ── 四、龙虎榜动向 ──
    if dragon_data:
        lines.append("---")
        lines.append("")
        lines.append("## 四、龙虎榜动向")
        lines.append("")
        lines.append(f"> 当日上榜: **{len(dragon_data)}** 只")
        lines.append("")
        lines.append("| 代码 | 名称 | 涨幅% | 净买额(万) | 上榜原因 |")
        lines.append("|------|------|-------|------------|----------|")
        for d in dragon_data[:20]:
            reason_short = d["reason"][:40] if d["reason"] else "-"
            lines.append(
                f"| {d['code']} | {d['name']} | "
                f"{d['change_pct']:+.2f}% | {d['net_buy_wan']:+.0f} | "
                f"{reason_short} |"
            )
        lines.append("")

    # ── 五、行业轮动 ──
    if industry_data.get("top"):
        lines.append("---")
        lines.append("")
        lines.append("## 五、行业轮动")
        lines.append("")
        lines.append("### 🔥 涨幅 TOP 10 行业")
        lines.append("")
        lines.append("| 排名 | 行业 | 涨跌幅% | 上涨家数 | 下跌家数 | 领涨股 |")
        lines.append("|------|------|---------|----------|----------|--------|")
        for r in industry_data["top"][:10]:
            lines.append(
                f"| {r['rank']} | {r['name']} | {r['change_pct']:+.2f}% | "
                f"{r['up_count']} | {r['down_count']} | {r['leader']} |"
            )

        if industry_data.get("bottom"):
            lines.append("")
            lines.append("### ❄️ 跌幅 TOP 5 行业")
            lines.append("")
            for r in industry_data["bottom"][-5:]:
                lines.append(f"- {r['name']}: {r['change_pct']:+.2f}%")

        lines.append("")

    # ── 六、复盘总结 ──
    lines.append("---")
    lines.append("")
    lines.append("## 六、复盘总结")
    lines.append("")

    # 主线
    top_themes = list(strong_themes.items())[:3] if strong_themes else []
    if top_themes:
        main_line = " → ".join(
            f"**{t}**({info['count']}只涨停)" for t, info in top_themes
        )
        lines.append(f"### 🔥 今日主线")
        lines.append(f"{main_line}")
        lines.append("")

    # 暗线
    lines.append(f"### 🧵 暗线观察")
    lines.append("")
    lines.append(f"- 涨停总数 {limit_up_count} 只，市场情绪 {sentiment}")
    lines.append(f"- 关注明日主线是否延续，以及分歧后的新方向")
    lines.append("")

    # 明日关注
    lines.append(f"### 👀 明日关注")
    lines.append("")
    if top_themes:
        for t, info in top_themes[:3]:
            leader = info["stocks"][0] if info["stocks"] else None
            if leader:
                lines.append(f"- **{t}**: 关注龙头 **{leader['code']} {leader['name']}** "
                           f"({leader.get('seal_type', '')}) 次日溢价情况")
    lines.append(f"- 涨停炸板股的反包机会")
    lines.append(f"- 新题材的首板试错机会")
    lines.append("")

    return "\n".join(lines)


# 用法
report = generate_review_report("2026-06-23")
print(report)

# ⚠️ 不要直接保存到此路径！必须使用下方的 save_review_to_project() 落地到项目目录。
```

---

## 批量复盘（多日连续）

```python
def batch_review(start_date: str, end_date: str):
    """
    批量复盘：连续多天生成复盘报告。
    用于回顾一段时间的题材演变和龙头轮动。
    """
    from datetime import date, timedelta

    start = date.fromisoformat(start_date)
    end = date.fromisoformat(end_date)

    d = start
    while d <= end:
        date_str = d.strftime("%Y-%m-%d")
        print(f"正在复盘 {date_str}...")
        try:
            report = generate_review_report(date_str)
            with open(f"review_{date_str}.md", "w") as f:
                f.write(report)
            print(f"  ✅ {date_str} 已保存")
        except Exception as e:
            print(f"  ❌ {date_str} 失败: {e}")
        d += timedelta(days=1)
        time.sleep(2)  # 每天之间暂停防封


# 用法: 复盘最近一周
# batch_review("2026-06-17", "2026-06-23")
```

---

## ⚠️ 报告落地与 Git 推送（必须执行）

**这是复盘的最后一步，必须完成，不允许跳过。** 之前出现过报告生成后只打印到终端、或保存到项目外路径（如 `/home/ubuntu/reviews/`）导致文件丢失的问题。

### 落地流程

```
1. 生成复盘报告 → Markdown 字符串
2. 保存到项目目录: reviews/{YYYY-MM-DD}/report.md
3. 保存原始数据: reviews/{YYYY-MM-DD}/data/*.json
4. 生成摘要卡片: reviews/{YYYY-MM-DD}/README.md
5. 更新索引: reviews/index.md
6. Git commit + push 到用户 fork (myfork)
```

### 目录结构（标准）

```
reviews/
├── README.md
├── index.md                    # 复盘索引（每新增一天，追加一行）
└── YYYY-MM-DD/
    ├── README.md               # 当日摘要卡片
    ├── report.md               # 完整复盘报告（Markdown）
    └── data/
        ├── stocks.json         # 涨停股原始数据
        ├── index.json          # 指数行情
        ├── themes.json         # 题材归类结果
        └── dragon_tiger.json   # 龙虎榜数据
```

### 落地函数

```python
import os, json
from datetime import date as _date
from collections import Counter

def save_review_to_project(date_str: str, report: str,
                           stocks: list[dict], themes: dict,
                           index_data: dict, dragon_data: list[dict],
                           project_root: str = None):
    """
    将复盘报告和原始数据落地到项目 reviews/ 目录。
    """
    if project_root is None:
        project_root = os.environ.get(
            "A_STOCK_PROJECT_ROOT",
            os.getcwd()
        )
    
    review_dir = os.path.join(project_root, "reviews", date_str)
    data_dir = os.path.join(review_dir, "data")
    os.makedirs(data_dir, exist_ok=True)
    
    # ── 1. 保存完整报告 ──
    report_path = os.path.join(review_dir, "report.md")
    with open(report_path, "w", encoding="utf-8") as f:
        f.write(report)
    print(f"✅ 报告已保存: {report_path}")
    
    # ── 2. 保存原始数据 ──
    with open(os.path.join(data_dir, "stocks.json"), "w", encoding="utf-8") as f:
        json.dump(stocks, f, ensure_ascii=False, indent=2)
    
    with open(os.path.join(data_dir, "index.json"), "w", encoding="utf-8") as f:
        json.dump(index_data, f, ensure_ascii=False, indent=2, default=str)
    
    themes_serializable = {}
    for theme, info in themes.items():
        d = dict(info)
        d["stocks"] = [{"code": s["code"], "name": s["name"]} for s in info["stocks"]]
        themes_serializable[theme] = d
    with open(os.path.join(data_dir, "themes.json"), "w", encoding="utf-8") as f:
        json.dump(themes_serializable, f, ensure_ascii=False, indent=2)
    
    with open(os.path.join(data_dir, "dragon_tiger.json"), "w", encoding="utf-8") as f:
        json.dump(dragon_data, f, ensure_ascii=False, indent=2)
    
    print(f"✅ 原始数据已保存: {data_dir}/")
    
    # ── 3. 生成每日摘要卡片 ──
    _save_daily_readme(date_str, stocks, themes, index_data, review_dir)
    
    # ── 4. 更新复盘索引 ──
    _update_review_index(date_str, stocks, themes, project_root)
    
    return review_dir


def _save_daily_readme(date_str: str, stocks: list[dict], themes: dict,
                        index_data: dict, review_dir: str):
    """生成当日的 README.md 摘要卡片"""
    
    limit_up_count = len(stocks)
    twenty_cm = sum(1 for s in stocks if s.get("change_pct", 0) >= 19.9)
    st_count = sum(1 for s in stocks if "ST" in s.get("name", ""))
    seal_dist = Counter(s.get("seal_type", "-") for s in stocks)
    strong_themes = {k: v for k, v in themes.items() if v["count"] >= 3}
    
    if limit_up_count >= 80:
        sentiment = "🔥🔥 极度亢奋"
    elif limit_up_count >= 50:
        sentiment = "🔥 强势"
    elif limit_up_count >= 30:
        sentiment = "✅ 温和"
    elif limit_up_count >= 10:
        sentiment = "❄️ 偏冷"
    else:
        sentiment = "🧊 冰点"
    
    lines = []
    lines.append(f"# 📊 {date_str} 复盘摘要")
    lines.append("")
    lines.append("## 市场概况")
    lines.append("")
    lines.append("| 指标 | 数值 |")
    lines.append("|------|------|")
    lines.append(f"| 涨停数 | {limit_up_count} 只 |")
    lines.append(f"| 20cm | {twenty_cm} 只 |")
    lines.append(f"| ST | {st_count} 只 |")
    lines.append(f"| 市场情绪 | {sentiment} |")
    for seal, cnt in seal_dist.most_common():
        lines.append(f"| {seal} | {cnt} 只 |")
    lines.append("")
    
    lines.append("## 指数表现")
    lines.append("")
    lines.append("| 指数 | 收盘 | 涨跌幅 |")
    lines.append("|------|------|--------|")
    for name, info in index_data.items():
        lines.append(f"| {name} | {info['price']:.2f} | {info['change_pct']:+.2f}% |")
    lines.append("")
    
    lines.append("## 核心题材")
    lines.append("")
    lines.append("| 热度 | 题材 | 涨停数 | 龙头 |")
    lines.append("|------|------|--------|------|")
    for theme, info in list(strong_themes.items())[:10]:
        leader = info["stocks"][0] if info["stocks"] else None
        leader_str = f"{leader['code']} {leader['name']}" if leader else "-"
        lines.append(f"| {info['strength']} | {theme} | {info['count']} | {leader_str} |")
    lines.append("")
    
    lines.append("## 关键看点")
    lines.append("")
    lines.append("<!-- 复盘时由 Claude 填充关键观察 -->")
    lines.append("")
    
    readme_path = os.path.join(review_dir, "README.md")
    with open(readme_path, "w", encoding="utf-8") as f:
        f.write("\n".join(lines))
    print(f"✅ 摘要卡片已生成: {readme_path}")


def _update_review_index(date_str: str, stocks: list[dict], themes: dict,
                          project_root: str):
    """更新 reviews/index.md 复盘索引表"""
    
    index_path = os.path.join(project_root, "reviews", "index.md")
    
    limit_up_count = len(stocks)
    twenty_cm = sum(1 for s in stocks if s.get("change_pct", 0) >= 19.9)
    st_count = sum(1 for s in stocks if "ST" in s.get("name", ""))
    
    strong_themes = {k: v for k, v in themes.items() if v["count"] >= 3}
    weak_themes = {k: v for k, v in themes.items() if v["count"] < 3}
    strong_count = len(strong_themes)
    weak_count = len(weak_themes)
    main_lines = "/".join(list(strong_themes.keys())[:3]) if strong_themes else "-"
    
    new_row = (
        f"| {date_str} | {limit_up_count} | {twenty_cm} | {st_count} | "
        f"{strong_count} | {weak_count} | {main_lines} | "
        f"[📄]({date_str}/report.md) |"
    )
    
    if os.path.exists(index_path):
        with open(index_path, "r", encoding="utf-8") as f:
            content = f.read()
        
        header_marker = "| 报告 |"
        if header_marker in content:
            insert_pos = content.index(header_marker) + len(header_marker)
            newline_pos = content.index("\n", insert_pos)
            new_content = (
                content[:newline_pos + 1] +
                new_row + "\n" +
                content[newline_pos + 1:]
            )
            with open(index_path, "w", encoding="utf-8") as f:
                f.write(new_content)
            print(f"✅ 索引已更新: {index_path}")
            return
    
    # 索引文件不存在，创建新文件
    lines = [
        "# 📊 复盘索引",
        "",
        "| 日期 | 涨停 | 20cm | ST | 强势题材 | 零散题材 | 主线 | 报告 |",
        "|------|------|------|----|----------|----------|------|------|",
        new_row,
    ]
    with open(index_path, "w", encoding="utf-8") as f:
        f.write("\n".join(lines) + "\n")
    print(f"✅ 新索引文件已创建: {index_path}")


# ── 完整落地调用示例 ──
# date_str = "2026-06-24"
# stocks = fetch_limit_up_stocks(date_str)
# themes = classify_themes(stocks)
# index_data = _fetch_index_quotes()
# dragon_data = _fetch_daily_dragon_tiger(date_str)
# report = _build_report(date_str, stocks, themes, index_data, {}, dragon_data)
# save_review_to_project(date_str, report, stocks, themes, index_data, dragon_data)
```

### Claude 执行清单（必须逐条完成）

复盘完成后，Claude 必须自动执行以下步骤，**不得跳过**：

1. **确认项目路径**：本项目路径为 `/home/ubuntu/a-stock-data`。

2. **调用 save_review_to_project()**：保存报告 + 数据到 `reviews/{date}/` 目录。

3. **手动完善 README.md**：函数生成的摘要卡片中"关键看点"部分是占位符，Claude 需根据复盘内容手动填充关键观察（如主线持续性、炸板情况、明日关注点等）。

4. **Git 提交**：
   ```bash
   cd /home/ubuntu/a-stock-data
   git add reviews/
   git commit -m "feat: add {date_str} daily review report"
   ```

5. **推送到用户 fork**：
   ```bash
   git push myfork main
   ```
   推送前确认 remote `myfork` 存在并指向 `xiaobei0125/a-stock-data`。

6. **验证**：确认 GitHub 上可看到提交，报告链接可访问。

> **❌ 常见错误（必须避免）**：
> - 报告只打印到终端，未保存文件
> - 保存到项目外路径（如 `/home/ubuntu/reviews/`）
> - 保存了 report.md 但漏了数据文件或 index 更新
> - 提交了但没推送
> - 推送到 origin (simonlin1212) 而非 myfork (xiaobei0125)
> 
> **以上任何一项遗漏都视为复盘未完成。**
```

---

## 综合示例：完整三步复盘

```python
# ── 完整三步复盘流程 ──

# Step 1: 获取全市场涨停股
print("=" * 60)
print("Step 1: 涨停梳理")
print("=" * 60)
stocks = fetch_limit_up_stocks("2026-06-23")
print(f"当日涨停: {len(stocks)} 只")

# 封板类型分布
from collections import Counter
seal_dist = Counter(s["seal_type"] for s in stocks)
for seal, cnt in seal_dist.most_common():
    print(f"  {seal}: {cnt} 只")

# 标签词频（跨所有涨停股）
all_tags_flat = []
for s in stocks:
    all_tags_flat.extend(s.get("reason_tags", []))
tag_counter = Counter(all_tags_flat)
print(f"\n全市场高频标签 TOP 10:")
for tag, cnt in tag_counter.most_common(10):
    print(f"  {tag}: {cnt} 次")

# Step 2: 题材归类
print()
print("=" * 60)
print("Step 2: 题材归类")
print("=" * 60)
themes = classify_themes(stocks)
print(f"识别到 {len(themes)} 个题材")
for theme, info in themes.items():
    print(f"  {info['strength']} {theme}: {info['count']}只涨停")

# Step 3: 龙头/先锋/跟风识别
print()
print("=" * 60)
print("Step 3: 题材内角色定位")
print("=" * 60)
for theme, info in themes.items():
    if info["count"] >= 3:
        role_assigned = identify_roles(info["stocks"])
        print(f"\n  📌 {theme} ({info['count']}只):")
        for s in role_assigned:
            print(f"    {s.get('role', '')} | {s['code']} {s['name']} | "
                  f"+{s['change_pct']}% | {s.get('seal_type', '-')} | "
                  f"{s.get('role_detail', '')}")

# 生成完整报告
print()
print("=" * 60)
print("生成复盘报告...")
print("=" * 60)
report = generate_review_report("2026-06-23")
print(report[:500] + "\n...(报告完整内容见上)")
```

---

## 复盘方法论核心

### 复盘的三层价值

```
第一层: 知道今天什么涨了（涨停股列表）
         ↓
第二层: 知道为什么涨（题材归类 + 上涨逻辑）
         ↓
第三层: 知道谁会继续涨（龙头/先锋识别 → 次日溢价判断）

大多数投资者停留在第一层。职业交易员的优势在第二层和第三层。
```

### 复盘自查清单（每日必问）

1. **主线持续性**: 今天的主线和昨天一样吗？还是切换了？
2. **龙头溢价**: 昨天的龙头今天溢价了吗？溢价多少？
3. **分歧还是加速**: 题材是加速（一字板增多）还是分歧（烂板增多）？
4. **新题材信号**: 有没有新出现的题材？首板股多吗？
5. **炸板观察**: 炸板股多吗？哪类股票容易炸板？
6. **量能确认**: 涨停股的换手率和量比是否健康？
7. **机构参与**: 龙虎榜上有机构买入吗？买了什么？

---

## FAQ

### Q: 同花顺热点数据覆盖全吗？
A: 实测覆盖 ~122 只强势股，其中 ~115 只涨停股。对于主板 10% 涨停的股票覆盖较全，双创板 20% 涨停的偶尔会漏（因为同花顺算法侧重市场关注度高的股票）。如需"一只不漏"，启用 `fetch_limit_up_clist_fallback()` 兜底。

### Q: 东财 clist 为什么经常 502？
A: 东财 push2 的 clist 接口对全市场扫描（`m:0+t:6,m:0+t:80,...`）有更严格的风控，批量请求容易触发 502。建议用同花顺热点作为主力数据源，clist 只在需要时启用作兜底。

### Q: 封板时间为什么拿不到？
A: 盘后复盘无法获取精确的盘中封板时间（需要实时行情流）。本 skill 用换手率 + 封板特征（一字板/T字板/换手板）作为代理变量：低换手 ≈ 早封板。这个代理在 80% 的情况下是准确的。

### Q: 题材归类不够准怎么办？
A: 扩充 `TAG_MERGE_RULES` 字典。同花顺的 tags 是人工编辑的，同一概念可能有不同表述（如"液冷泵" vs "液冷冷却液" vs "液冷服务器"——其实都是"液冷"题材）。持续丰富合并规则会让归类越来越准。

### Q: 龙头和先锋怎么区分？
A: 简单公式：龙头 = 题材内涨幅最高；先锋 = 题材内最早涨停（一字板/最低换手率）。大多数时候龙头就是先锋（因为它最早涨停、涨幅也最高）。当它们不同时，先锋是真正的"点火资金"，龙头是"市场选出来的代表"。

### Q: 非交易日能用吗？
A: 同花顺热点在非交易日返回空数据，会收到"今日非交易日或无数据"提示。可以在下一个交易日使用。

---

## 安装说明

```bash
# 1. 创建 skill 目录
mkdir -p ~/.claude/skills/stock-review

# 2. 将本文件复制为 SKILL.md
cp SKILL.md ~/.claude/skills/stock-review/SKILL.md

# 3. 确保 a-stock-data skill 已安装（依赖其 em_get / EM_SESSION 等 helper）
# ls ~/.claude/skills/a-stock-data/SKILL.md

# 4. 启动 Claude Code，说"帮我复盘今天A股"即可自动激活
```

> 📦 https://github.com/simonlin1212/a-stock-data — 依赖 a-stock-data skill
