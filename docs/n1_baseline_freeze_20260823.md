# N1 基准冻结（硬化 A，2026-08-23）

**性质**：不可改基准清单。不是提高收益任务。  
**纸交易仍是 N1。** 本文件不改选股、风险、止损、regime。  
**日期**：2026-08-23（Asia/Shanghai）  
**作者**：量化研发  
**清单**：[n1_hardening_checklist.md](./n1_hardening_checklist.md) A1–A3

---

## A1 四数基准（再次确认）

隔离窗 2024-01-03 .. 2025-12-31。NAV 100000。成本 1bp + 10bp。2026 未打开。

| 指标 | 公布四舍五入 | 精确值（gate） |
| --- | ---: | ---: |
| CAGR | 21.82% | `0.21819999998526063` |
| MaxDD | 7.71% | `0.07707808566551255` |
| 2024 | +27.48% | `0.2747886912126327` |
| 2025 | +16.14% | `0.1613911938486754` |
| 交易数 | 245 | `245` |
| 累计 | 48.05% | `0.48052835999222854` |
| 暴露 | 54.44% | `0.5444384696094042` |

三次独立复现（同一隔离树、同一 pin）四数 **exact 对齐**。最近一次：N5 隔离开跑前的 N1（2026-08-23 01:05 上海），`/workspace/quant/strategy-n5/independent/artifacts/20260822/n1_repro.json`。N4、N3 开跑前的 N1 同样 exact。摘要：`reports/hardening/20260823/a1_n1_repro.json`。

本轮没有为 A1 再开第四趟、没有搜参、没有改 0.85%。

---

## A2 不可改校验值

哈希是文件内容 **SHA-256**。改其中任一文件 = 基准漂移，必须新开字母策略，不得回写 N1。未哈希 `state.db` / token / env。

### 规格仓（本机 `/home/box/GitHub/strategy-n1-spec`）

| 文件 | SHA-256 | 字节 |
| --- | --- | ---: |
| `config/n1.toml` | `e67ef65f3f3c8d7132b47f390811510c494b32d136298dd10c01149834074546` | 2985 |
| `SPEC.md` | `36337484b11b1b5e994080d7e6c34ca5ceb617b10e9cf43f714370c3a86359e3` | 13252 |
| `README.md`（本提交前） | `ad832ff7a9b10479d9d15be9933cc5b33075229c9d90be39653f37b26c7e9115` | 3215 |
| `config/sector_by_symbol.csv` | `fd62cd5765c3ff6fe6f812f2a32d35adc84976ce2a7bc8f731026448b15ca568` | 2827 |
| `reports/isolated-2024-2025.md` | `ca9b936484e6ad5c1dc6e18bbc22fa6f9b4816e166f2554d07dfb17e4808ba4f` | 487 |
| `reports/isolated-2024-2025.json` | `32c3d5be430510a40178b23151e9ac8d812019b8927f605546a098694c67d72e` | 1119 |

GitHub `main` 在本提交前：`411fb6763a54e142224334b41f71a3b197455b7d`。

### 隔离数据 pin（禁止 2026）

| 项 | SHA-256 |
| --- | --- |
| monthly manifest | `cb11cbcf067857e8d2fedacb6ce9ed7cb8df4a6eee00f84a3d8d5b69702e5afd` |
| reference_evidence.json | `f102cebf7f3f3c902ea970a0cbc7648db8ca76334c4848c6311f73131d71fd1a` |
| monthly_plan_override.json | `0ef946e11bfee87332de091975bc33abf2436b794b7c6c61b2b971e5fb7bfcee` |
| ISOLATION.json | `f65a80b6a477b1442b076b954505b2e7a025bef65e12710cfee20da04b9e8f20` |
| stage_250 文件数 | 24 |
| stage_250 文件名列表 SHA-256 | `653d64f0e0316691c71b308b50ac47cd46cfab0d30cde29ada31351d3b897ae2` |

### 隔离引擎 n-lift（只读哈希，本轮未改 src）

| 文件 | SHA-256 |
| --- | --- |
| `config/strategy.toml` | `7ddc56f7227d2d4a5213d327c3391e292e645298e658b167f19f1f1d4fd9edd1` |
| `config/candidate_c.toml` | `e72b4bf3a7d70405c72f19b0674d6797dcfb330d8db16b16430be08a74dd19c7` |
| `config/strategy_d3.toml` | `49e497fcd6dfcdcff4a6626d84b894cb9c27b0376620a833a234de6cf1542fa5` |
| `config/strategy_e.toml` | `21d4004d843d8019bbd239ef4296a009faf5c10b626669c541fa84969059d680` |
| `src/quant_trader/candidate_e.py` | `57a7a55129a2ba93f1a62cd1b38f1d7f619c812640d89401c5e56644ffa42ba2` |
| `src/quant_trader/candidate_c.py` | `2a807f11ac171bae891d1e7e86be3c5878a7af06d577316327aef04922d53179` |
| `src/quant_trader/candidate_c_risk.py` | `7f490c58df62d9e42f53dbe48e7196907b336559fad5b11063095cb8660aa03d` |
| `src/quant_trader/backtest.py` | `ee0c61159b043b25bbe25955bf4b0aef1a3345199eb96a5347ef3e20aeac32f7` |

父提交钉：E2 `70cc50ad965234b99a5a136d4e4f283eedd4281f`。N1 身份：`Strategy_N1_70cc50a_risk085`，Strong Bull `0.0085`。

### 纸盘宇宙快照（只读哈希，未改纸盘）

| 项 | 值 |
| --- | --- |
| 生效宇宙 | `config/strategy_e2/20260803.json` |
| SHA-256 | `38cfc1a351a0f8488d207ac28a48e74762e736aae4db1f4e40a99c8a22ce2a33` |
| 202609.json | 不存在（禁止提前写） |
| state.db / token | 未哈希 |

---

## A3 见红线文档

[n1_redlines.md](./n1_redlines.md)
