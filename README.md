# Strategy N1 规格

当前模拟盘策略。E2 基线冻结；N1 只改 Strong Bull 单笔风险。

不是实盘授权。本仓库不含 `state.db`、长桥 token、crontab 实盘密钥。

## 给回测 / 研发

1. 读 `Strategy_N1_规格_20260814.md`
2. 风险参数以 `config/strategy_e.toml` 为准：`strong_bull_trade_risk = 0.0085`
3. 隔离窗对照见 `reports/validation/strategy_n1/20260814/`
4. 不要改云电脑上正在跑的 N1 逻辑和门闩

正式版：**N1** = E2 @ `70cc50a` + Strong Bull 风险 0.85%。
