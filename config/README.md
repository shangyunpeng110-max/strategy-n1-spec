# 三份 toml 哪份算数

外人只看仓库时，按这张表取值。不要把三份文件并起来用。

N1 纸交易加载顺序（代码事实）：先读 `strategy.toml` 当底座，再被 `candidate_c.toml` 覆盖风险格子，Strong Bull 时再被 `strategy_e.toml` 改一格。

## 用这些

| 项 | 文件 | 值 |
| --- | --- | --- |
| 信号时刻 | strategy.toml | 13:35 |
| 最多新开 | strategy.toml | 5 |
| 动量窗口 | strategy.toml | 44 / 88 |
| Top 分数比例 | strategy.toml | 0.55 |
| ATR 周期 | strategy.toml | 20 |
| 时间止损 | strategy.toml | 12 根（交易日） |
| 限价偏离 | strategy.toml [execution] | 10 bp |
| Top N / 最低价 / 最少历史 / ADV | candidate_c.toml | 50 / 10 / 200 / 1e8 |
| 单票上限 | candidate_c.toml | 15% |
| 行业上限 | candidate_c.toml | 30% |
| Bull / Neutral / Bear 单笔风险 | candidate_c.toml | 0.50% / 0.35% / 0.25% |
| Bull / Neutral / Bear 多头上限 | candidate_c.toml | 90% / 60% / 20% |
| Strong Bull 广度门槛 | candidate_c.toml | 65% |
| 行业名 → ETF | candidate_c.toml [sector_reference_etfs] | 11 个中文名 |
| 股票 → 行业 | sector_by_symbol.csv | 隔离 2024–2025 用的 105 只 |
| Strong Bull 单笔风险 | strategy_e.toml | **0.85%** |
| 发行人上限 | strategy_e.toml | 15%（GOOG/GOOGL 算一家） |

## 不要用这些

`strategy.toml` 里这些是早期双动量残留，N1 **不读**：

- `universe = [...30 只...]` — 体制 19 只从这里去掉 ETF 得到，**不是**交易宇宙
- `donchian_entry_bars` / `donchian_exit_bars` / `supertrend_*` — N1 开平仓不用
- `[risk] trade_risk / single_name_cap=0.20 / sector_cap=0.40` — 被 candidate_c 的 15%/30% 覆盖

`candidate_c.toml` 里的 `strong_bull_trade_risk = 0.006` 是 E2 旧值。N1 用 `strategy_e.toml` 的 **0.0085**。

## 行业怎么来

没有一张写死的全市场股票→行业表。

- **隔离复现 21.82%/7.71%：** 用本目录 `sector_by_symbol.csv`（从隔离树 `reference_evidence.json` 抽出）。这张表不是 PIT，和当时跑出 21.82% 的口径相同。票不在表里 → 算不了 `relative_sector_score`，当天不能当新开。
- **纸交易：** 当月 E2 bootstrap JSON 每一行自带 `sector` 和 `reference_etf`，缺一行当天宇宙作废。
- `candidate_c.toml` 只做 **中文行业名 → 行业 ETF**，不回答某只股票是哪个行业。
