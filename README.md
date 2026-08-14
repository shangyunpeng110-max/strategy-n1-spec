# Strategy N1 规格

当前模拟盘策略。完整因子、排序、开平仓、体制判定、DMA/斜率/分位、黑名单见 `Strategy_N1_规格_20260814.md`。

N1 = E2 全套规则 + Strong Bull 单笔风险 0.85%（E2 是 0.65%）。

不是实盘授权。本仓库不含 `state.db`、长桥 token、env。

## 文件

- `Strategy_N1_规格_20260814.md` — 策略、因子、怎么用
- `config/strategy_e.toml` — N1 风险（0.0085）
- `config/strategy.toml` — 44/88、Top55%、5 仓、13:35
- `config/candidate_c.toml` — 宇宙门槛与行业 ETF
- `reports/validation/strategy_n1/20260814/` — 隔离窗对照

不要改云电脑上正在跑的 N1 逻辑和门闩。
