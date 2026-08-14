# Strategy N1 — 冻结规格

状态：正式模拟盘（纸交易）。按云电脑运行中的代码冻结，不是调参入口。  
日期：2026-08-14（Asia/Shanghai）  
版本：`Strategy_N1_70cc50a_risk085`  
研究基线：E2 @ `70cc50ad965234b99a5a136d4e4f283eedd4281f`

N1 相对 E2 **只改一处**：Strong Bull 单笔风险 `0.65% → 0.85%`。  
宇宙、因子、排序、开平仓、止损、第六仓条件全部与 E2 相同。禁止把 X/Y/Z 或其他候选叠进 N1。

给人看的解析配置：[config/n1.toml](./config/n1.toml)  
一天怎么走：[README.md](./README.md)

---

## 1. 这是什么

美股多头、日频信号、月度换池。不做空、不杠杆。

1. 每月用 PIT 流动性池（`stage_250` → Top50）圈可交易股票。
2. 每个交易日 13:35 America/New_York，用**已收盘**日线算因子，在 Top 55% 里选人。
3. 先按 C 层综合分筛，再按行业相对强度（E1B）重排，最多 5 仓；D3 扩展可到第 6。
4. 仓位 = 体制风险预算 × 分数仓位乘数，再砍单票 / 行业 / 发行人 / 体制上限。
5. ATR 跟踪止损 + 12 日时间止损离场。

研究谱系（只为读旧名）：B/C 是两层因子；E1B 是开仓排序键；D3 是第六仓扩展；E2 是母版；E3/E4 开关在 N1 上是关的。

---

## 2. 宇宙（每月换一次）

数据：月度 PIT `stage_250`。当月可交易 = `rejection_reason is None` 的 Top50，且 `common_stock`。

| 项 | 值 |
| --- | --- |
| Top N | 50 |
| 最低价 | 10 美元 |
| 最少历史 | 200 个交易日 |
| 20 日平均成交额 | ≥ 1 亿美元 |
| 行业映射 | 11 个 SPDR 行业 ETF（见 `config/n1.toml`） |

参考 ETF（不算多头候选人）：SPY、QQQ、XLK、XLF、XLV、XLI、XLE、XLP、XLY、XLU、XLB、XLRE、XLC。

隔离 2024–2025 的股票→行业用 `config/sector_by_symbol.csv`（105 只，字段 `symbol,sector,reference_etf`）。来自隔离树 `reference_evidence.json`，**不是 PIT**。票不在表里 → 不能算 `relative_sector_score`，不能进 E1B，不能占行业 30% 额度，当天不能新开。`FI.US` 在 2025-06 Top50 但不在表里，当天不能新开。

纸交易不读这张 CSV。当月 E2 bootstrap 每一行自带 `sector` 和 `reference_etf`，缺任一字段当天宇宙作废。

行业名必须落在这 11 个：信息技术 XLK、金融 XLF、医疗保健 XLV、工业 XLI、能源 XLE、必选消费 XLP、可选消费 XLY、公用事业 XLU、材料 XLB、房地产 XLRE、通信服务 XLC。

---

## 3. 因子（每日，已完成日线）

窗口收益：`return_n = close[t] / close[t-n] - 1`，n ∈ {20, 44, 88}。只用已收盘 bar，不用当天未完成小时线。

### 3.1 B 层

| 因子 | 规则 |
| --- | --- |
| `return_20` / `return_44` / `return_88` | 20/44/88 日价格收益 |
| `relative_spy_score` | 0.5×(r44−SPY44) + 0.5×(r88−SPY88) |
| `relative_sector_score` | 0.5×(r44−行业ETF44) + 0.5×(r88−行业ETF88) |
| `trend_quality_score` | 四项各 +1：收盘>DMA20、收盘>DMA50、DMA20 近 5 日斜率>0、DMA50 近 10 日斜率≥0；近 20 日回撤 < −8% 再 −1 |
| `volatility_penalty` | ATR% 分位>0.70 且（非近高或 20 日收益≤0）→ −1；分位>0.85 且 20 日收益≤0 → 再 −1 |
| `volume_score` | ADV≥1 亿 → +1；5 日量/20 日量≥1.10 且 20 日收益>0 → +1；量比<0.70 且 20 日收益≤0 → −1 |
| `eligible` | r44>0 **且** r88>0 **且** ADV≥1 亿 |

`candidate_b_score` =

- 0.35 × rank(r44)
- + 0.35 × rank(r88)
- + 0.15 × rank(r20)
- + 0.10 × rank(相对 SPY)
- + 0.05 × rank(相对行业)
- + 0.20 × (trend_quality / 4)
- + 0.15 × (volume_score / 2)
- + volatility_penalty

这里的 rank 是 pandas `Series.rank(pct=True).fillna(0)`，不是 C 层那套确定性排序。

B 层 `volatility_penalty` 的 ATR%：**不是横截面**。对该票自己的历史算 `(high-low)/close`，再 `rank(pct=True)` 取**最后一根**。

### 3.2 C 层（实际用来筛 Top 55%）

在 B 层之上再加三个质量因子（都要求 44/20 日收益完整、路径长度>0、20 日波动>0）：

| 因子 | 公式 |
| --- | --- |
| `trend_efficiency` | r44 / sum(\|近 44 日日收益\|) |
| `volatility_adjusted_momentum` | ((r44+r88)/2) / 近 20 日收益标准差 |
| `relative_strength_acceleration` | (r20−SPY20) − (r88−SPY88) |

`candidate_c_score` =

- 0.60 × rank(candidate_b_score)
- + 0.20 × rank(trend_efficiency)
- + 0.10 × rank(vol-adj momentum)
- + 0.10 × rank(RS acceleration)

`score_percentile` = rank(candidate_c_score)。不合格质量的票当天 `eligible=false`。

C 层 / 止损用的分位是 `_deterministic_rank`：只对有限值，按 `(-value, symbol)` 排序，第 k 名（0 起）得 `(N-k)/N`。并列按代码字母序打破。不是 pandas rank。

### 3.3 E1B（开仓排序，不是再筛一层）

先按 `candidate_c_score` 取 eligible 的 **Top 55%**（`momentum_top_fraction=0.55`）。进入之后，开仓顺序改为：

1. `relative_sector_score` 降序
2. `score_percentile` 降序
3. 代码字母序

### 3.4 DMA / 斜率（钉死）

DMA 是收盘简单移动平均，不是 EMA。`DMA20 = close.rolling(20).mean()`，`DMA50 = close.rolling(50).mean()`，取最后一根。

斜率不是回归。`_slope(series, days) = series[-1] / series[-days-1] - 1`。

- DMA20 斜率：对 `DMA20.dropna()` 取 days=5 → `DMA20[-1]/DMA20[-6] - 1`，要 > 0
- DMA50 斜率：对 `DMA50.dropna()` 取 days=10 → `DMA50[-1]/DMA50[-11] - 1`，要 ≥ 0

---

## 4. 体制（两套广度，不要混）

### 4.1 确认体制 `active`：看写死的 19 只 + SPY/QQQ

**不看**当月 Top50。看 `config.py` 里写死的股票池。

`UNIVERSE` 是早期双动量 30 个名字（11 ETF + 19 股票）。体制用的是去掉 ETF 后写死的 19 只，**不会**随每月 Top50 换人。要改成按月度池判体制，必须另冻新策略。

**BREADTH_SYMBOLS（缺一只就返回 None，当天不改体制）：**  
AAPL.US, MSFT.US, NVDA.US, AVGO.US, META.US, GOOGL.US, AMZN.US, TSLA.US, JPM.US, BAC.US, LLY.US, UNH.US, COST.US, WMT.US, HD.US, CAT.US, GE.US, XOM.US, CVX.US

| 代码 | 均线 | 条件 |
| --- | --- | --- |
| SPY.US | SMA200 = close.rolling(200).mean() 最后一根 | close > SMA200 |
| QQQ.US | SMA200 | close > SMA200 |
| 上面 19 只 | SMA100 = close.rolling(100).mean() 最后一根 | close > SMA100 |

`regime_breadth` = 19 只里收盘在 SMA100 之上的比例。

观察投票（三票）：SPY 上 +1，QQQ 上 +1，`regime_breadth ≥ 0.50` +1。

| 票数 | 观察体制 |
| --- | --- |
| 3 | bull |
| 0 | bear |
| 1 或 2 | neutral |
| SPY/QQQ/19 只任一缺数据 | None，当天不改体制 |

**确认（滞后 2 日）：** 观察体制连续 2 个交易日相同，才把 `active` 改成新体制。用的是确认后的 `active`，不是当天观察值。

state 里没有体制记录时：`active = neutral`，`pending = None`，`pending_days = 0`，`last_observation_date = None`。第一天只观察，**不会**当天切到 bull。从 neutral 要连续 2 个交易日观察为 bull，第三天起 `active` 才是 bull。回测必须从这段空状态起步，不能一上来当 Strong Bull。

副作用：巨头先崩会晚降仓；小盘活、巨头死会按巨头砍仓。这是代码事实，不是笔误。

### 4.2 Strong Bull：另看当月宇宙的 C 广度

确认体制已经是 bull 之后，再算 `candidate_c_market_breadth`（对象是**当月宇宙股票**，不是上面 19 只）：

- 宇宙里每只至少 max(50, 45) 根收盘
- 上当当且仅当：`close > 近 50 日收盘均值` **且** `close/close[-45] - 1 > 0`（44 日收益）
- 缺数据 → fail-closed
- `c_breadth` = 上当只数 / 宇宙只数

Strong Bull = `active==bull` **且** `c_breadth >= 0.65` **且** SPY 44 日收益>0 **且** QQQ 44 日收益>0。

### 4.3 风险格子（N1 唯一改动）

| 状态 | 单笔风险 `trade_risk` | 多头上限 |
| --- | ---: | ---: |
| Strong Bull | **0.85%（N1；E2 是 0.65%）** | 100% |
| Bull | 0.50% | 90% |
| Neutral | 0.35% | 60% |
| Bear | 0.25% | 20% |

取值见 [config/n1.toml](./config/n1.toml)。N1 只用 Strong Bull `0.0085`。

---

## 5. 开仓

- 时刻：美东 13:35，当天尚未做过入场。
- 只做多普通股，不做 ETF。
- 必须在当月 Top50、eligible、Top 55%、有 ATR20、不在黑名单。
- 基础最多 **5** 仓。D3 扩展条件满足时可到 **第 6**（E2 的 `e3_sixth_slot=false`，第六仓走 D3/D2 扩展，不是 E3 那套分位）。
- D3 扩展门槛：`score_percentile≥0.80`，趋势效率分位≥0.60，加速分位≥0.65。E3/E4 的扩展分位和 20 日时间止损，N1 不用。

不能新开：

1. 所有 ETF：SPY、QQQ、IWM、11 个行业 ETF，以及 C 层参考 ETF
2. 不在当月 PIT Top50 允许集合里的股票
3. 当天财报 `earnings_today`
4. 当天已被强制平仓的代码 `forced`
5. 已持仓（只占用名额，不重复开）
6. 当日已离场的代码，当天禁止再进（回测 `blocked_reentry`）

已有 ETF 多头只许减仓/平仓，不许加仓。SPY/QQQ 只走对冲路径（N1 纸交易不做新开对冲当主仓）。

---

## 6. 仓位（算股数的 ATR ≠ 仓上止损的 ATR）

```
qty = floor(NAV × trade_risk × size_mult / (stop_atr_mult × ATR20))
```

| 项 | 规则 |
| --- | --- |
| `size_mult` | C 分位≥0.80 → 1.15；≥0.50 → 1.00；否则 0.85 |
| `stop_atr_mult`（只进分母） | 默认 2.0；若横截面 ATR% 分位≥0.70 **且** 趋势效率分位≥0.50 **且** C 分位≥0.80 → 2.5 |
| 横截面 `atr_percentile` | `_deterministic_rank(atr20/close)`，atr20 来自当日 morning 表 |
| 再砍 | 单票 15% NAV、行业 30%、发行人 15%、体制多头上限 |
| 发行人 | GOOG/GOOGL 合并为 ALPHABET |

单票/行业/发行人上限见 `config/n1.toml`：15% / 30% / 15%。

---

## 7. 离场

真正比价的止损位和上一节的 `stop_atr_mult` 不是同一套。

| 规则 | 值 |
| --- | --- |
| 初始止损 | 入场价 − **2.5 × 入场 ATR** |
| 跟踪止损 | 持仓最高价 − **3.0 × 入场 ATR**；`max_r ≥ 1` 才开始跟 |
| `active_stop` | `max(initial_stop, trailing_stop)`；还没有 trailing 就只用 initial |
| 时间止损 | 12 个交易日（E4 未开，不会延长到 20 日） |
| 触发价 | 已完成 **1h 的 close**，不看 low |
| 旧开关 | 双动量的 3.2/4.0 高 ATR 开关，N1 不用 |

每根已完成的 1h（bar 起点 + 1 小时才算完成）：

1. `hourly_stop_price = 该小时 close`
2. `close <= active_stop` → 记 `hourly_stop`，当天禁止再开这只
3. 没打止损才用该小时 **high** 更新 `peak_high`，再决定要不要抬 trailing

纸交易若当时拿不到小时线，才退回最新价。隔离复现必须有 1h，不要用这个退路。

---

## 8. 执行钉死

### 8.1 三套 K 线不要混

| 用途 | 周期 | 用哪根 |
| --- | --- | --- |
| morning / ATR20 | **1h** | 美东 09:30、10:30、11:30、12:30 合成 13:30 |
| 盘中止损、抬高 peak | **1h** | 09:30 … 15:30 的 **close** |
| 隔离回测成交（21.82% 那条） | **15m** | 13:35 之后、16:00 之前第一根 = **13:45 open** |

### 8.2 morning 合成

半日市不用。正常交易日取美东整点小时线 09:30、10:30、11:30、12:30 共 4 根（秒/微秒必须为 0）：

| 字段 | 值 |
| --- | --- |
| 时间戳 | 当天 13:30 America/New_York（再转 UTC） |
| open | 09:30 那根的 open |
| high | 四根 high 的最大 |
| low | 四根 low 的最小 |
| close | 12:30 那根的 close |
| volume | 四根 volume 之和 |

算指标时只用不晚于决策时刻的 morning 历史，且最后一根的 NY 日期必须等于决策日。13:35 决策时，当天这根 13:30 合成 bar 已经在手里。

### 8.3 ATR20（Wilder，不是简单均值）

用 **morning 合成 K 线** 算，不是日线。`atr_period = 20`。缺 ATR 的票当天不能开仓。

```
TR_t = max(H-L, |H-C_{t-1}|, |L-C_{t-1}|)
ATR_20 的第 20 根 = 前 20 根 TR 的算术平均
之后 ATR_t = (ATR_{t-1} × 19 + TR_t) / 20
```

### 8.4 成交价

**隔离回测：** 信号 13:35 ET。在当天 15m 执行帧里取 `bar_time > 13:35` 且 `bar_time < 16:00` 的第一根，正常交易日就是 **13:45** 的 **open**。

`fill = open × (1 + sign(delta) × 10bp)`，佣金 1bp。买贵卖便宜。没有这根 15m 则当天不成交。

不要用 1h 的 14:30 open 当成交价，对不上 21.82%。

**纸交易：** 长桥 Demo 限价，`limit_deviation_bps = 10`，实际成交看券商回执。

### 8.5 财报黑名单

当天 `event_date` 在黑名单里的股票不能新开。

- 月度 E2 bootstrap JSON 里的 `earnings_events`：`known_at ≤ 决策时刻` 才算看见
- 纸交易另加 `var/paper/strategy_e2/earnings_updates/{YYYY-MM}/`（schema `strategy_e2_prospective_earnings_update.v1`）
- 隔离复现 21.82%：用隔离树快照自带的 earnings，**不要**读实盘 updates，也别现场拉长桥日历

---

## 9. 运行约束

- 纸交易 only，长桥 Demo。
- 独立 `state.db`、独立门闩 `N1_CRON_ARMED`。
- 本仓库不含持仓、token、`state.db`、env。
- 隔离窗数字见 [reports/isolated-2024-2025.md](./reports/isolated-2024-2025.md)。
