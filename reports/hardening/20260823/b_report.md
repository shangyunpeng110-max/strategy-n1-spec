# N1 加固清单 B — 纸面执行一致性（只读）

- 核查日：2026-08-23（Asia/Shanghai）
- 根：`/home/box/quant-trader-n1-paper`
- 身份：`Strategy_N1_70cc50a_risk085`
- 约束：未改纸盘、未下单、未碰 N1_CRON_ARMED / 0.85% / 202609.json，未读钥匙，未 clone。
- 本地草稿：`/workspace/n1-hardening-b-20260823/`

## 总表

| 项 | 结论 | 一句话 |
|---|---|---|
| B1 信号时间 | **部分过** | N1 三路齐全仅 7 个交易日（要求 20）。拼 decisions morning-1335 可覆盖 33 天。不装成 20 天三路已齐。 |
| B2 成交 | **部分过** | 纸面成交价与决策 ref 可核；无 15m 缓存，回测假设价全部「未核到」。 |
| B3 持仓/权重/regime | **过** | 6 仓皆 8 月 Top50 普通股，无 ETF，无 V 残留，单票<15%，regime=bull。 |
| B4 止损 | **不过** | 可分类 hourly_stop/time_stop 只有 1 次（8/19 V time_stop）。forced_exit 18 次但未写类型。不足 10 次。 |
| B5 失败闭环 | **过** | 8/13–18 缺 ATR → 0 targets；鉴权/下单失败有日志；acceptance 例行在；systemctl 假失败已剥离。 |

禁止装完：B1 不把 7 天三路说成 20 天；B2 不编 15m；B4 不把未分类 forced_exit 算成 10 次 hourly/time stop。

详细证据见同目录 `b1_signal_calendar.csv` / `b2_fills.csv` / `b3_positions.md` / `b4_stops.csv` / `b5_failclosed.md`。

## B1
- N1 活体 13:35 三路：8/13, 8/14, 8/17, 8/18, 8/19, 8/20, 8/21（共 7）。shadow 延迟均值 1.934s，signal 日志均 13:35:01 rc=0。
- 8/12 交易日漏：当日 N1_CRON_ARMED missing no-op，武装在 16:51 ET。
- 拼 inherited decisions 可到 33 天（约 +37s），无周一常漏、无每天晚 10 分钟。不够 20 天 N1 三路 → 部分过。

## B2
- executions 48 笔；有决策 ref 48；15m 核到 0。
- 8/19 V 卖 fill 368.825 vs ref 368.67 = +4.20 bps；CRM 买 205.54 vs 205.8 = -12.63 bps。
- 全样本 |bps| 均值 28.89；|bps|>30 异常 10 笔，几乎全是 7 月–8/11 继承成交。

## B3
- 现仓 AMZN/CRM/JPM/MSFT/PANW/PLTR，皆 8 月 Top50，无 ETF，无 V。JPM 约 14.10% 最高。regime=bull。信息技术约 29.15%。

## B4
- 唯一可分类：8/19 V time_stop，days_held=12，initial=entry-2.5×ATR 相符。
- 其余 forced_exit 未写 hourly_stop/time_stop 类型，不分类。

## B5
- 8/13–18 atr20 50/50 None → targets=0；8/19 ATR 恢复后才开 CRM。
- 8/13 09:35 OAuth rc=1 有日志；13:35 正式窗 rc=0。
- systemctl FileNotFound 是环境假失败，不当 fail-closed 破了。
