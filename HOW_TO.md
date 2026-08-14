# 别人怎么用 N1

读完这份就能独立算出某天该不该开仓、开谁、开多少、怎么停。公式细节在 `Strategy_N1_规格_20260814.md`。参数以 `config/` 为准。

不是实盘授权。不要改云电脑上正在跑的 N1。

## 你需要的数据

1. **当月宇宙**：PIT `stage_250` JSON。可交易 = `rejection_reason is None` 且 `common_stock`。隔离复现用 `isolated-2024-2025/evidence/selection_2024_2025/monthly_universe/stage_250/`。
2. **日线**：宇宙股票 + SPY.US + QQQ.US + 11 个行业 ETF。NY 日历，已收盘。复现窗 2024-01-03..2025-12-31，2026 不能读。
3. **小时线**：美东 09:30/10:30/11:30/12:30 四根整点（合成 morning）+ 13:35 之后到 16:00 的执行 K 线。
4. **财报**：快照里的 `earnings_events`（`known_at` 不晚于决策时刻）。复现不要拉实盘日历。
5. **体制 19 只日线**：AAPL MSFT NVDA AVGO META GOOGL AMZN TSLA JPM BAC LLY UNH COST WMT HD CAT GE XOM CVX，各至少 100 根；SPY/QQQ 各至少 200 根。

## 一个交易日（美东，按这个顺序做）

半日市：不算 morning、不开新仓。

1. **取当月 Top50。** 有效日 ≤ 今天。不在名单里的不能新开。
2. **更新体制（油门，不是买名单）。**
   - 19 只：close > SMA100 的比例 = `regime_breadth`
   - SPY、QQQ：close > SMA200
   - 三票：SPY 上 + QQQ 上 + (regime_breadth≥0.50)。3=观察 bull，0=bear，否则 neutral。缺任何一只当天不改体制。
   - 连续 2 个交易日观察相同，才改 `active`。第一天从 `active=neutral` 起，当天不会变成 bull。
3. **用已收盘日线算因子（只对 Top50 + 行业 ETF + SPY）。**
   - 先 B 层：`candidate_b_score`、`relative_sector_score`、`eligible`（r44>0 且 r88>0 且 ADV≥1 亿）
   - 再 C 层：`candidate_c_score`、`score_percentile`
   - B 层横截面用 pandas `rank(pct=True)`；C 层用确定性排序分位。两套不要混。
4. **合成 morning，算 Wilder ATR20。** 四根小时线 → 一根 13:30 bar。缺 ATR 不能开。
5. **13:35 筛开仓名单。**
   - eligible 且非 ETF 且在 Top50
   - 按 `candidate_c_score` 取 Top 55%
   - 去掉：财报日、当天已平、已持仓、强制平仓
   - 剩下的按 E1B 排：行业相对强度 → C 分位 → 代码
   - 最多 5 个新开；D3 扩展条件满足可到第 6（score≥0.80 且趋势效率分位≥0.60 且加速分位≥0.65）
6. **定风险格子。**
   - `active` 先定 bull/neutral/bear 的风险和上限
   - Strong Bull 还要：active=bull **且** 当月宇宙 C 广度≥65% **且** SPY44>0 **且** QQQ44>0
   - C 广度看的是月度池，不是那 19 只
   - N1 只改 Strong Bull 单笔风险为 **0.85%**（E2 是 0.65%）。`strategy_e.toml` 为准；`candidate_c.toml` 里的 0.006 是旧值，N1 不用它。
7. **算股数（向下取整）：**
   `qty = floor( NAV × trade_risk × size_mult / (stop_atr_mult × ATR20) )`
   - size_mult：C 分位≥0.80 → 1.15；≥0.50 → 1.00；否则 0.85
   - stop_atr_mult：默认 2；若横截面 ATR% 分位≥0.70 且趋势效率分位≥0.50 且 C 分位≥0.80 → 2.5
   - 再砍到：单票 15% NAV、行业 30%、发行人 15%（GOOG/GOOGL 算一家）、体制多头上限
8. **成交（回测）：** 当天 13:35 之后、16:00 之前第一根执行 K 线的 open × (1±10bp)，佣金 1bp。没有这根 K 线就不成交。
9. **离场（每天都查持仓）：** 价破 入场价−倍数×ATR 或 最高价−倍数×ATR；或持有满 12 个交易日。当天平掉的，当天不能再开。

## 19 只股票是什么

**不是买名单，是仓位油门。**

早期 `UNIVERSE` 去掉 11 只 ETF 剩的 19 只大盘股。换月 Top50、因子、排序、止损都跟它们无关。它们只和 SPY/QQQ 一起决定 `active` 体制，从而决定单笔风险和多头上限。

巨头先崩会晚降仓；小盘活、巨头死会按巨头砍仓。要改成按月度池判体制，另冻新策略，不要改 N1。

## 怎么验收你做对了

隔离窗 2024-01-03..2025-12-31，2026 不打开，成本 1bp+10bp：

| | N1 | E2 |
| --- | ---: | ---: |
| CAGR | 21.82% | 21.10% |
| MaxDD | 7.71% | 7.67% |
| 2024 | +27.48% | +23.91% |
| 2025 | +16.14% | +18.09% |

对不上先查：体制是否从 neutral 起、19 只是否写死、两套广度有没有混、B/C 分位函数有没有用反、morning 是否用四根小时线、成交是否 13:35 后第一根 open。

新策略过门：同一窗 CAGR>21.82% 且 MaxDD≤7.71%。不过不上盘。

## 纸交易和回测的差别

- 回测成交 = 执行 K 线 open+滑点
- 纸交易 = 长桥 Demo 限价 10bp，以券商成交为准
- 纸交易财报还读 `var/paper/strategy_e2/earnings_updates/`；复现不要读
- 本仓库没有 token、state.db、持仓
