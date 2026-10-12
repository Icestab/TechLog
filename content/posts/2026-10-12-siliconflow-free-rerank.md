---
title: "白嫖硅基流动的免费重排模型：把 Hindsight 的召回质量捡回来，代价是 3 秒"
date: 2026-10-12T14:00:00+08:00
draft: false
description: "关重排省下的是额度不是钱。分清「额度免费」和「限速免费」之后，换一个限速免费的 provider 把召回质量拿回来，代价只剩延迟——顺带查出这个 bank 级开关还在收后台刷新的税"
tags: ["hindsight", "rerank", "siliconflow", "rag", "召回质量", "免费额度", "agent-memory"]
categories: ["技术日志"]
---

上一篇《[Hindsight 成本调优](/posts/2026-10-11-hindsight-cost-tuning/)》里，第一个降本杠杆就是「关掉重排」：两个 bank 都关，召回从 3437ms 掉到 435ms，重排 API 调用归零，顺手白拿一个 8 倍提速。

这一篇要把这个杠杆**打开一半**。因为事后看，当初关它的理由有个没分清的地方：**它省下的是额度，不是钱。**

## 一、先分清两种「免费」

重排模型本来就在免费额度里，账单上根本没出现这一项——关掉它并没有让账单变小，只是让额度不被烧穿。这两种「免费」是不同的东西：

| | 额度免费（百炼） | 限速免费（硅基） |
|---|---|---|
| 计费 | 累计量用完就没 | 费用恒为 0 |
| 撞上限 | 功能只能关掉 | 429，等一分钟重试 |
| 成本上界 | 一个会耗尽的池子 | 一个可以排队的速率 |

当初关重排是为了保额度。**换一个限速免费的 provider，质量就能拿回来，代价只剩延迟。**

顺带先排除一个误判：我直接打了百炼的 rerank 端点，HTTP 200、分数区分度正常（0.865 vs 0.0007）——**不是额度耗尽导致的静默降级，是被人为关掉的**。这一步不做，后面所有分析都会跑偏。

## 二、A/B：关掉到底损失了多少

在 `pi` bank 上开回开关、跑同一批 query 对比：

```
每次召回的候选数：205 / 230 / 246
重排改动了顺序的条数：205/205, 230/230, 246/248
平均 |排名变化|：59.6 / 70.0 / 82.7      最大：189 / 199 / 235
```

**几乎全部重排**——这不是微调，是两套完全不同的排序。

关键在探针怎么设计：**目标词不能出现在记忆文本里**，否则关键词臂会让两边都命中同一个结果，你测出来的「没差别」其实是探针失效。用英文 query 打中文记忆库就是好探针：

| 查询（词面不重合） | 目标记忆 | 关重排 | 开重排 |
|---|---|---|---|
| `which timezone format should timestamps use` | 北京时间（UTC+8） | #3 | **#1** |
| `how should tools be invoked to avoid approval popups` | 北京时间（UTC+8） | 没进结果 | **#20**（RRF 排名 207） |
| `did the user accept an always-allow allowlist` | 北京时间（UTC+8） | #32 | **#16** |
| `how should tools be invoked...` | hugo-blog | #25 | **#11** |
| `did the user accept an always-allow allowlist` | hugo-blog | #5 | #33 |
| 中文：`Hindsight 的 consolidation 成本优化怎么做的` | 04:00 那次任务记录 | **#1** | 掉出前列 |

汇总：**开重排后 top-10 里，有 5/10/7 个来自 RRF 排名 >15**。

再看一个中文 query 的具体重排轨迹（`Hindsight 的 consolidation 成本优化是怎么做的`）：

```
新 top-10 里来自 RRF 排名 >15 的：7 个
RRF#1 → 第 3      RRF#2 → 第 21     RRF#3 → 第 40     RRF#5 → 第 73
新 #1 来自 RRF#15  新 #4 来自 RRF#160
```

**这里有个重要的缓解机制**：Hindsight 不是无条件信重排。交叉编码器打完分之后还有一道 `combined_scoring`，把 `cross_encoder` 分和 recency / proof_norm / temporal 等信号加权融合——最终顺序不是纯 bge 顺序，重排的升降会被软化。这也是为什么离线直接打 bge 端点拿到的排序，和走 Hindsight 出来的最终 top-N 对不上。

所以结论是双向的，不是单边变好：**语义和跨语言召回明显变好，但 RRF 本来就排对的会被压下去。**

## 三、接上硅基流动

翻 Hindsight 源码，它原生就支持一大批重排 provider，`cross_encoder.py` 里排了 14 个：

```python
class LocalSTCrossEncoder / RemoteTEICrossEncoder / CohereCrossEncoder /
      ZeroEntropyCrossEncoder / SiliconFlowCrossEncoder / TypeSafeCrossEncoder /
      RRFPassthroughCrossEncoder / FlashRankCrossEncoder / LiteLLMCrossEncoder /
      LiteLLMSDKCrossEncoder / JinaMLXCrossEncoder / GoogleCrossEncoder /
      AlibabaCloudCrossEncoder / MultiCrossEncoder
```

**硅基流动是其中一个**，走和 alibaba 同一个 Cohere 兼容客户端：

```python
# engine/cross_encoder.py:930
class SiliconFlowCrossEncoder(CrossEncoderModel):
    """
    SiliconFlow exposes a Cohere-compatible /rerank endpoint.
    Shares the HTTP client with ZeroEntropy/Cohere-custom-endpoint.
    """
    RERANK_PATH = "/rerank"
```

默认值也正好指向免费档（`config.py`：`BAAI/bge-reranker-v2-m3` + `https://api.siliconflow.cn/v1`）。改动全在 compose：

```diff
-      HINDSIGHT_API_RERANKER_PROVIDER: alibaba
-      HINDSIGHT_API_RERANKER_ALIBABA_API_KEY: ${DASHSCOPE_API_KEY}
-      HINDSIGHT_API_RERANKER_ALIBABA_MODEL: qwen3.7-text-rerank
+      # 免费 id 是 BAAI/bge-reranker-v2-m3；带 Pro/ 前缀的同名模型是收费的
+      # model/base_url 显式写死，不吃镜像默认值
+      HINDSIGHT_API_RERANKER_PROVIDER: siliconflow
+      HINDSIGHT_API_RERANKER_SILICONFLOW_API_KEY: ${SILICONFLOW_API_KEY}
+      HINDSIGHT_API_RERANKER_SILICONFLOW_MODEL: BAAI/bge-reranker-v2-m3
+      HINDSIGHT_API_RERANKER_SILICONFLOW_BASE_URL: https://api.siliconflow.cn/v1
```

`DASHSCOPE_API_KEY` 不能删——嵌入还在用它，只摘掉重排对它的引用。改完重启（约 60 秒），日志里有三行确认：

```
INFO - Reranker: provider=siliconflow
INFO - Reranker: initializing SiliconFlow provider at https://api.siliconflow.cn/v1 with model BAAI/bge-reranker-v2-m3
INFO - Reranker: SiliconFlow provider initialized
```

最后把 bank 开关打开（**bank 级，每个 bank 单独设**，默认值是 `true`，新建的 bank 会立刻开始打重排）：

```bash
PATCH /v1/default/banks/<bank>/config
{"updates": {"enable_reranking": true}}
```

## 四、实测：token 和延迟

把一次真实召回的 264 个候选原样送去打分，直接读响应里的 `meta.tokens.input_tokens`——这是精确计量，不是估算：

| 候选数 | rerank 耗时 | input_tokens | 速率 |
|---:|---:|---:|---:|
| 264 | 2.90s | 26,890 | 18.6 次/分钟 |
| 300 | 3.72s | ≈30,570 | 16.4 次/分钟 |
| 100 | 1.32s | 11,399 | 43.9 次/分钟 |
| 50 | 0.79s | 6,503 | 76.9 次/分钟 |

延迟随候选数基本是线性的（约 0.35s 固定开销 + 9.7ms/候选），每档都重复测过，确认不是冷启动。

**更正上一篇的一个数**：那里写「单次约 5 万 token」是按 300 候选估的，实测是 **26,890 token / 264 候选（101.9 token/候选）**，300 候选约 30,570。量级没变，但数字要对，以本文为准。

端到端召回：

| | 关重排 | 开重排 |
|---|---:|---:|
| recall 总耗时 | 0.65s | **3.58–4.56s** |
| 其中 reranking 阶段 | 0.000s（passthrough） | 3.00–3.72s |

服务端日志是硬证据，两种模式在直方图上完全分离：

```
[4] Reranking [rrf-passthrough]: 300 candidates scored in 0.001s (pre-filtered 228)   ← 关
[4] Reranking [cross-encoder]:   300 candidates scored in 3.724s (pre-filtered 486)   ← 开
```

`pre-filtered 486 → 300` 说明候选是先从 486 里筛到预算上限 300，再整批送进重排的。

**一个意外的旁证**：给 `hermes` 开完开关后立刻做了一次召回，top-1 就是几分钟前刚写进库的那条硅基重排记录（`ce=0.921`）——新鲜记忆立刻被召回并排到第一，说明通路和时效性都对。

**第二个没预料到的结果：硅基比百炼慢 2–3 倍**（0.69–1.27s vs 3.00–3.10s）。`bge-reranker-v2-m3` 是 568M 参数的 XLM-RoBERTa-large，比 8B 级的 qwen3.7-text-rerank 小得多，本来预期更快——实际更慢，大概率和免费档的调度优先级有关。**换 provider 解决了额度，没解决延迟。**

## 五、这个开关还收了谁的税

这是全过程最值得记的一步。`enable_reranking` 是 bank 级的，但它的作用范围比想象中大——引擎里唯一的降级判断是这样的：

```python
def _resolve_reranking(config_dict, reranking):
    """Only "cross_encoder" is downgraded, and only to "rrf".
    "interleave" is an explicit caller choice ... and "rrf" is
    already rerank-free, so neither is overridden."""
    if reranking == "cross_encoder" and not config_dict.get("enable_reranking", True):
        return "rrf"
    return reranking
```

把所有内部 `recall_async` 调用点 grep 一遍，会发现：

- **consolidation 豁免**——它显式传了 `reranking="interleave"`，不受开关影响（这点上一篇验证过）
- **但 reflect（心智模型刷新）的内部召回没传参数**，走默认 `cross_encoder`，被这个开关一起管

也就是说，给 bank 打开重排，等于**给后台刷新路径按每次内部召回加 3 秒**。这不是「聊天时慢 3 秒」，是「后台刷新变慢」：

```
两个模型各做了 9 / 8 次工具调用（≈ 内部召回次数）
刷新基线耗时：95s / 133s      墙钟上限 300s
加税后预估：  130s / 164s      （+35s / +31s）
```

TPM 那边更要留意——重排的主要消费者不是聊天，是后台刷新，这和直觉相反：

```
300 候选 ≈ 30,570 token/次 → 16.4 次/分钟就把 TPM 500,000 用满
两个刷新并发 × 9 次 ≈ 18 次 / 2–3 分钟 ≈ 275k token/分钟 → 占 TPM 的 55%
```

**余量不是 20 倍，是不到 2 倍。**

上面那组「加税后」的耗时是预估，**判据要等下一轮刷新**：实际 <200s、日志无 429、`mental_models.last_refresh_failed_at` 为空——三条都满足，这套配置就可以长期保留；墙钟上限只有 300s，超了就是刷新失败并冻结心智模型，那是另一个坑。

顺带一个观测上的缺口：`/metrics` 里 `phase="reranking"` 只打了阶段标签，**没有 `provider`/`model` 标签**（LLM 那些指标是带的）。所以「重排到底打的哪家」从指标上看不出来，只能靠启动日志或外部账单确认——这和上一篇「日志和指标都会骗人」是同一类教训。

## 六、坑

**① 空 key 会让容器起不来（最该记的一条）。** `provider=siliconflow` + 空 key 会在构造 `MemoryEngine` 时抛 `ValueError`，而这是**启动路径**——「先改配置、key 回头再填」这个顺序是有风险的，中间任何一次重启（含断电）都会让记忆服务起不来。做法：`.env` 里先放一个非空占位符（构造函数只检查非空），后续真触发重排时 401 会走降级路径（未打分的候选保持 RRF 顺序排在后面），**recall 不会报错**，等于给自己留个安全垫。

**② 免费 id 和 `Pro/` id 长得几乎一样。** 免费是 `BAAI/bge-reranker-v2-m3`，收费是 `Pro/BAAI/bge-reranker-v2-m3`。用错 id 就会以为免费档不能用——所以 model 显式写死。

**③ 候选数上限是服务级的，不能分 bank。** `HINDSIGHT_API_RERANKER_MAX_CANDIDATES`（默认 300）和它的 `_LOW/_MID/_HIGH` 分层覆盖都读服务级配置，想「只降一个 bank」做不到。

**④ 交叉编码器会把近似重复挤在 top。** 实测出现过 top-1 和 top-2 是两条几乎同文的记忆——交叉编码器给相似文本打相似分，不做多样性抑制，而召回有 token 预算，重复项白占预算。缓解靠调用端的 `prefer_observations=true`（丢掉已被 observation 覆盖的原始 fact）。

**⑤ 免费档不是承诺。** 硅基的免费模型目录会轮换，本文的价格快照日期是 2026-08-21，`Pro/` 前缀那条线随时可能动。上线后偶尔看一眼计费页，别等账单出来才发现免费 id 已经下架。

**⑥ `bge-reranker-v2-m3` 的能力边界，以及两个被排除的选项。** 它 568M 参数、XLM-RoBERTa-large 骨干、8k 上下文，官方验证中/英/日/韩/俄/法/阿排序正确、支持跨语言，比 8B 级的 qwen3.7-text-rerank 小一档——**效果会略低但够用**。顺带算清另外两条路：

- **付费重排不划算**：硅基上 `Qwen/Qwen3-Reranker-8B` 是 $0.04/M token，按实测 30,570 token/次 = ¥0.014/次，一天 100 次就是 ¥1.4/天 ≈ **¥42/月，比整个 LLM 账单还贵**
- **本地方案也不行**：镜像里 `flashrank` 和 `onnxruntime` 都装好了，理论上能纯本地零额度，但默认模型 `ms-marco-MiniLM-L-12-v2` 是**纯英文**、MS MARCO 段落训练的，对中文记忆库基本没用；容器还限 800m / 1 CPU，CPU 跑重排只会更慢

## 七、谁该开，谁该关

**开**：在意召回质量、能接受单次 3 秒、后台刷新墙钟余量够（预估 164s vs 上限 300s，还剩一半）。质量收益是实打实的——语义和跨语言查询的提升，恰好是记忆系统最常走的路。

**关**：墙钟余量本来就紧（这一刀切掉约 45%），或者对聊天实时性敏感。RRF 在词面重合的查询上并不差。

**折中**：候选数压到 100——rerank 降到 1.32s、11,399 token，刷新只加 12 秒，TPM 余量回到约 4 倍。质量损失远小于收益（截断发生在打分之前，top-10 基本不动）。

三条回滚路径，按代价从小到大：

```bash
# B：只关开关（bank 级，一条命令，不用重启容器）
PATCH /v1/default/banks/<bank>/config
{"updates": {"enable_reranking": false}}

# C：降候选数（服务级，两个 bank 一起生效，要重启）
#   HINDSIGHT_API_RERANKER_MAX_CANDIDATES: "100"
#   实测：rerank 1.32s / 11,399 token → 刷新只 +12s，TPM 余量回到 ~4 倍

# A：换回原 provider（要重启）
#   HINDSIGHT_API_RERANKER_PROVIDER: alibaba
#   HINDSIGHT_API_RERANKER_ALIBABA_API_KEY: ${DASHSCOPE_API_KEY}
#   HINDSIGHT_API_RERANKER_ALIBABA_MODEL: qwen3.7-text-rerank
```

三条可迁移的结论：

1. **遇到「免费额度被烧穿」，先分清是额度免费还是限速免费。** 前者累计用完就没，后者只要不超速率就永远 ¥0。找到后者，之前为了保额度关掉的功能就能重新打开——成本上限从「会耗尽的池子」变成「可排队的速率」。
2. **开关类配置要追到「谁继承了这个默认值」。** 一个 bank 级开关会静默影响所有没显式指定排序策略的内部调用方，这种连带不报错、不进日志，只会在某天变成「刷新怎么变慢了」——改之前把调用点 grep 一遍。
3. **额度类服务要读响应里的 usage 字段来计量，别估算。** `meta.tokens.input_tokens` 一次真实调用就给出了精确的 per-candidate 成本，和「300 × 每条大概多少」的差价能到 2 倍。
