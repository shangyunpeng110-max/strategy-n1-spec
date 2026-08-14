# Strategy N1

当前模拟盘策略。别人要能照着算信号、照着复现，从 `HOW_TO.md` 开始。

N1 = E2 全套 + Strong Bull 单笔风险 **0.85%**（E2 是 0.65%）。只改这一格。

不是实盘授权。不要改云电脑上正在跑的 N1 逻辑、crontab、门闩、state.db。

## 读这个顺序

1. [HOW_TO.md](./HOW_TO.md) — 一天怎么走、19 只是油门、怎么验收
2. [Strategy_N1_规格_20260814.md](./Strategy_N1_规格_20260814.md) — 公式、分位、黑名单、执行细节
3. `config/` — 冻结参数（N1 风险看 `strategy_e.toml` 的 0.0085）
4. `reports/validation/strategy_n1/20260814/` — 隔离窗对照数字

## 配置

| 文件 | 用途 |
| --- | --- |
| `config/strategy_e.toml` | N1：`strong_bull_trade_risk = 0.0085`，发行人上限 15% |
| `config/strategy.toml` | 44/88 日、Top55%、最多 5 仓、13:35、ATR20 |
| `config/candidate_c.toml` | 宇宙门槛、行业 ETF。里面的 0.006 **不是** N1 的 Strong Bull 风险 |

## 行业 ETF

信息技术 XLK、金融 XLF、医疗 XLV、工业 XLI、能源 XLE、必选消费 XLP、可选消费 XLY、公用事业 XLU、材料 XLB、房地产 XLRE、通信 XLC。
