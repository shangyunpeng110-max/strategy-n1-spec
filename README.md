# Strategy N1

美股多头、日频信号、月度换池。长桥 Demo **纸交易**。不是实盘授权。

相对 E2 只改一格：Strong Bull 单笔风险 `0.65% → 0.85%`。  
冻结：2026-08-14　`Strategy_N1_70cc50a_risk085`  
E2：`70cc50ad965234b99a5a136d4e4f283eedd4281f`

不要改云电脑上正在跑的 N1。提高收益另冻新策略。

| 文件 | 内容 |
| --- | --- |
| [config/n1.toml](./config/n1.toml) | 生效参数 |
| [SPEC.md](./SPEC.md) | 公式和执行钉死 |
| [config/sector_by_symbol.csv](./config/sector_by_symbol.csv) | 隔离复现用的 105 只行业表 |
| [reports/isolated-2024-2025.md](./reports/isolated-2024-2025.md) | 2024–2025 对照 |
| [docs/n1_hardening_checklist.md](./docs/n1_hardening_checklist.md) | **N1 加固清单**（不改逻辑，只做验证与运营加固） |
| [docs/n1_baseline_freeze_20260823.md](./docs/n1_baseline_freeze_20260823.md) | 硬化 A：四数基准 + 不可改哈希 |
| [docs/n1_redlines.md](./docs/n1_redlines.md) | 硬化 A3：选股 / 风险 / 止损 / regime 红线 |

## 参数

| 项 | 值 |
| --- | --- |
| 宇宙 | 月度 PIT Top50，价≥10，历史≥200 日，ADV≥1 亿，普通股 |
| 信号 | 美东 13:35；半日市不开新仓 |
| 筛选 | C 分 Top 55%，再按行业相对强度排序 |
| 仓数 | 最多 5；扩展可到第 6（C≥0.80、趋势效率≥0.60、加速≥0.65） |
| 单笔风险 | Strong Bull **0.85%** / Bull 0.50% / Neutral 0.35% / Bear 0.25% |
| 多头上限 | 100% / 90% / 60% / 20% |
| 集中度 | 单票 15%，行业 30%，发行人 15%（GOOG/GOOGL 一家） |
| 算股数 ATR | 2.0×，满足条件 2.5×（只进分母） |
| 仓上止损 | 初始 2.5×，跟踪 3.0×（`max_r≥1` 才跟）；打 **1h close** |
| 时间止损 | 12 个交易日 |
| 回测成交 | 15m 的 13:45 open × (1±10bp)，佣金 1bp |

两套广度不要混：体制看写死的 19 只；Strong Bull 看当月宇宙。

## 一天（美东）

半日市：不算 morning、不开新仓。19 只是油门，不是买名单。

1. 取当月 Top50。
2. 更新体制：19 只 `close>SMA100` 的比例，加上 SPY/QQQ 是否在 SMA200 上。三票全中为观察 bull，全不中为 bear，否则 neutral。连续 2 日相同才改 `active`。初始 `neutral`。
3. 已收盘日线算 B/C 因子。B 层用 pandas `rank(pct=True)`，C 层用确定性排序。
4. 四根 1h（09:30–12:30）合成 morning，算 Wilder ATR20。缺 ATR 不能开。
5. 13:35 在 eligible 且 Top 55% 里按行业相对强度 → C 分位 → 代码排序。去掉财报日、当天已平、已持仓、强制平仓。
6. 按 `active` 取风险格子。Strong Bull 还要宇宙 C 广度≥65% 且 SPY44>0 且 QQQ44>0。
7. `qty = floor(NAV × trade_risk × size_mult / (stop_atr_mult × ATR20))`，再砍上限。
8. 回测用 15m 13:45 open；纸交易看长桥限价回执。
9. 每根已完成 1h 的 close 对 `active_stop`。打中当天不能再开。没打中才用 high 更新峰值。或满 12 日。

复现数据：PIT `stage_250`、日线、1h、15m、快照里的 `earnings_events`、本仓库行业表。窗 2024-01-03..2025-12-31，**不要读 2026**。`FI.US` 不在行业表里，当天不能新开。

## 隔离窗

2024-01-03 .. 2025-12-31。2026 未打开。成本 1bp+10bp。

| | N1 | E2 |
| --- | ---: | ---: |
| CAGR | 21.82% | 21.10% |
| MaxDD | 7.71% | 7.67% |
| 2024 | +27.48% | +23.91% |
| 2025 | +16.14% | +18.09% |

新策略过门：同一窗 CAGR > 21.82% 且 MaxDD ≤ 7.71%。不过不上盘。

本仓库没有运行时、`state.db`、token、持仓、行情。
