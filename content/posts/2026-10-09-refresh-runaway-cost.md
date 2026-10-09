---
title: "免费额度 26 分钟被烧穿：一次「日志骗人」的成本排查"
date: 2026-10-09T21:00:00+08:00
draft: false
description: "日志只数到 19 次调用，控制台却是 590 多次：一个没设节流的后台刷新，怎么在 26 分钟里烧穿云 API 免费额度"
tags: ["成本治理", "日志", "后台任务", "RAG", "交叉编码器", "排查方法"]
categories: ["踩坑记录"]
---

自建的记忆服务跑得好好的：LLM 抽事实用小米 mimo，向量与重排用阿里云百炼。某天傍晚用户一句「免费额度用完了」，我直觉是提问太多——直到把账单和日志摆在一起，才发现两者差了一个数量级。文中时间均为北京时间。

## 一、对不上的两个数字

直接打重排 API 复现，返回的是：

```
{"code":"Arrearage","message":"Access denied, please make sure your account is in good standing."}
HTTP=400
```

注意错误码是 `Arrearage`（账号欠费），不是「免费额度用尽」，而且嵌入 API 报同样的错——整个账号级被拒，不是单个模型的问题。

阿里云百炼控制台显示：**嵌入模型 593 次调用、重排模型 100 多次调用**（两个模型的免费额度各 100 万 token）。而服务端日志里我只能数到 **19 次重排调用**，候选合计 1092 个。

> ⚠️ 这里踩了本次排查最大的一个坑：**看到「593 次」就默认它是重排的调用数**，于是按「593 次重排」一路归因，中途甚至把刷新次数算成了 79 次。直到有人指出「590 是嵌入的、重排只有 100 多次」，才回头把整条链重算——结论方向没错，但中间的数字错得离谱。教训见第八节第 8 条。

这 19 次倒是每一次都能对应到一条真实的用户提问：

```
18:14:48  <某组件的查询参数该用什么>...          → 171 个候选
18:14:55  <某项配置改在哪>...                   → 157 个候选
18:15:11  <某个路径约定的坑>...                 → 172 个候选
18:22:43  <某条阶梯计价规则>...                 → 102 个候选
...
18:39:59  <一次流程变更>...                    →  95 个候选
```

`Starting recall` 后面紧跟 `Reranking` 的日志在 18:40 之后就再没出现——额度是在 **18:14~18:40 这 26 分钟**里烧穿的。可 19 次、1092 个候选，按每次 1.7K token 估也就几万，**离 100 万差一个数量级**。所以这只是冰山一角。

## 二、日志为什么只有冰山一角

从代码入手找重排的所有调用点。召回管线只有一个入口：

```python
async def recall_async(
    self, bank_id, query, *,
    ...
    _quiet: bool = False,
    reranking: RecallReranking = "cross_encoder",     # ← 默认走交叉编码器
) -> RecallResultModel:
```

调用方只有 4 个：

```
api/http.py                REST 召回（用户侧）        → 日志里那 19 次
mcp_tools.py               MCP 工具                  → 手工测试
consolidation 内部召回                              → ？
reflect 内部召回（心智模型刷新）                      → ？
```

而日志为什么漏，代码里写得明明白白：

```python
if not quiet:
    logger.info("\n" + "\n".join(log_buffer))
```

**内部调用全都传了 `_quiet=True`，一条日志都不打。** 只数日志得到的成本，比真实值低一个数量级。

## 三、排除法：consolidation 我先搞错了一轮

consolidation 那处调用明确关掉了交叉编码器：

```python
reranking="interleave",   # 注释原文：Round-robin interleave fusion (no cross-encoder)
_quiet=True,
```

作者的注释解释了原因：交叉编码器实测会把「语义第 1 名的近似重复观察」压到第 37 名，导致 LLM 看不见它、于是制造重复。所以 consolidation 不重排，**排除**（我第一轮把它的 66 次算进去了，是错的）。

真凶是另一条路径：

```python
result = await memory_engine.recall_async(
    ...
    _quiet=True,          # 只是不打日志
    # 没有 reranking 参数  ← 所以用默认值 cross_encoder
)
```

**心智模型刷新的内部召回，走的是完整重排管线。**

## 四、放大链条：1 条新记忆引爆 10 次全量刷新

心智模型是常驻画像，配置了 `refresh_after_consolidation: true`：

```
每轮对话写入 → consolidation → 触发 2 个心智模型全量重刷
                              └→ 每次重刷是 reflect 工具循环
                                 └→ 循环内反复调 recall
                                    └→ 每次 recall 都走完整管线、都重排
```

按分钟看实测的放大倍数：

```
18:11   16 次写入 → 16 次 consolidation →  0 次刷新
18:12   14 次写入 →  9 次 consolidation → 11 次刷新
18:14    0 次写入 →  0 次 consolidation → 11 次刷新   ← 纯刷新
18:16    1 次写入 →  1 次 consolidation → 10 次刷新
18:23    1 次写入 →  1 次 consolidation → 10 次刷新
18:27    0 次写入 →  1 次 consolidation → 11 次刷新
```

**1 条新记忆就能引爆 10 次全量刷新。**

### 决定性证据：一次刷新的内部结构

一条没被 quiet 掉的日志完整暴露了一次刷新做了什么：

```
[REFLECT ...] done | iterations=7
| llm=[agent_1=3257ms, ..., agent_7=62421ms] (147288ms)
| tools=[ search_mental_models(...)=302ms/8047c,
          search_mental_models(...)=261ms/8042c,
          search_observations(...)=571ms/24335c,
          search_observations(...)=631ms/30161c,
          recall((query='...', max_tokens=4000))=951ms/30938c,
          recall(...)=1191ms/29542c, recall(...)=941ms/22433c,
          recall(...)=1207ms/23216c, recall(...)=749ms/24983c,
          recall(...)=717ms/19341c, recall(...)=926ms/21684c,
          recall(...)=855ms/19336c ] (9302ms)
| total=172155ms
```

**一次刷新 = 7 轮 LLM + 12 次工具调用。** 其中会打重排 API 的是 `recall` 和 `search_observations`（两者内部都调 `recall_async` 且没传 `reranking`），`search_mental_models` 直接查库不碰召回。后来用对照实验精确测出来（方法见第六节）：**8 次**。所以单次刷新的重排成本在 8~10 次之间浮动，取决于 LLM 当轮选了哪些工具。

**但这里的次数我算错了。** 当时把日志里 `Triggering refresh for 2 mental models` 这类**触发事件**（含被守卫跳过的）当成实际执行次数，数出「79 次刷新」。事后查数据库的 `async_operations` 表才发现，失控窗口内真正落库执行的只有 **15 次**：

```sql
select status, count(*) from async_operations
 where operation_type='refresh_mental_model' group by 1;
-- completed 16 | failed 2 | pending 2（历史累计；失控窗口内 15 次）
```

改正后数量级依然成立，而且和控制台严丝合缝：

```
15 次刷新 × 8 次重排调用 = 120 次
+ 用户侧 19 次召回        ≈ 139 次
控制台：重排模型 100 多次   ✓
```

## 五、它是怎么停下来的（比 token 数字更重要）

刷新在 18:38 突然停了，写入却还在继续。日志给出了原因：

```
18:40:28 [CONSOLIDATION] Skipped refresh for mental model <模型A>: its last refresh failed
18:40:28 [CONSOLIDATION] Skipped refresh for mental model <模型B>: its last refresh failed
```

**不是收敛，是撞墙。** 失败原因：

```
Task exceeded the 300s wall-clock limit for 'refresh_mental_model'
(stage=llm.openai.reflect.attempt=1/4.backoff) and was cancelled.
```

单轮 LLM 最长 62 秒，7 轮加工具调用与退避轻松超过 300 秒。超时被判为失败，而系统有个守卫：

```python
def _automatic_refresh_paused(...):
    """True when the model's last refresh failed, so automatic triggers must skip it.
    ... It stays paused until a refresh succeeds, which only an explicit one can do now."""
    return row["last_refreshed_at"] is None \
        or row["last_refresh_failed_at"] > row["last_refreshed_at"]
```

**两个心智模型从此冻结。** 讽刺的是冻结救了额度——但这是靠故障实现的止损。

## 六、根因是一行默认值

```
HINDSIGHT_API_MENTAL_MODEL_MIN_REFRESH_INTERVAL_SECONDS = 0    # 默认值
```

`0` = 没有任何节流，每次触发都真跑。叠上三个因素就成了成本放大器：

| 因素 | 说明 |
|---|---|
| 触发频率 | 每次 consolidation 都触发，而 consolidation 每次写入都跑 |
| 每模型独立 | 2 个心智模型 → 每次触发跑 2 次 |
| 单次极贵 | 一次 = 7 轮 LLM + 8 次内部 recall（每次重排 55~172 个候选） |

**关键点：这些内部召回对排序精度几乎没有要求**——它要的是「捞全证据」而不是「精准 top-5」。同一套代码里 consolidation 主动关掉了交叉编码器并写下了实测理由，reflect 却没关，于是为一堆「只需要捞全」的召回付了最贵的排序费用。

### 修复与验证

```yaml
environment:
  HINDSIGHT_API_MENTAL_MODEL_MIN_REFRESH_INTERVAL_SECONDS: "21600"   # 6 小时
```

语义：同一模型每 6 小时最多自动刷新一次，期间所有触发折叠成一个被停靠的刷新；显式刷新不受限且会释放被折叠的那次。改完重启，写一条记忆走完整链路验证：

```
<模型A> | pending | retry_count=0 | created=20:57:38 | next_retry_at=次日02:46:01
<模型B> | pending | retry_count=0 | created=20:57:38 | next_retry_at=次日02:45:33
```

创建时间 = consolidation 结束那一刻（确实触发了），`next_retry_at` = 上次成功刷新（20:45:33）+ 整整 6 小时，`retry_count=0` 说明是**停靠**而非失败重试。

**单次刷新的重排成本用对照实验实测**：记下 `phase="reranking"` 的直方图 → 触发一次显式刷新 → 再取一次看增量落在哪个桶：

```
总数   293 → 301   (Δ+8)
亚毫秒 263 → 263   (Δ+0)   ← 一次都没增加
慢      30 →  38   (Δ+8)   ← 新增全落在 0.5~1.0 秒（网络往返）
```

和工具轨迹逐一对上：`recall` ×5 + `search_observations` ×3 = 8 次走完整管线，`search_mental_models` ×4 不碰召回。**预测 +8，实测 +8。**

## 七、第二刀：把重排成本彻底归零

第一刀只砍掉了「失控」，日常成本还在——每一轮对话都在烧重排 token。

### 每轮的真实成本

一次召回会把整个候选池送给重排模型打分。用真实记忆样本实测（随机 100 条打 API 读 `usage`）：

```
169.5 token/候选
典型召回 170 个候选  →  约 29,000 token
打满上限 300 个候选  →  约 50,800 token
```

而重排模型单价约 0.5~0.6 元/百万 token、免费额度 100 万 token——**免费额度只够约 37 轮对话**（打满上限只够 20 轮）。「免费额度刚用就没了」不是意外。

### 候选池上限是第一个旋钮

```yaml
HINDSIGHT_API_RERANKER_MAX_CANDIDATES: 300   # 默认值；另有 _LOW/_MID/_HIGH 分级覆盖
```

这一刀切在交叉编码器**之前**（`trim_merged_candidates`），被截掉的候选直接丢弃：

| 上限 | 每轮 token | 相对成本 |
|---|---|---|
| 300（默认） | ~51K | 100% |
| 100 | ~17K | 33% |
| 50 | ~8.5K | 17% |

### 但更彻底的是关掉交叉编码器

```bash
curl -X PATCH .../v1/default/banks/<bank> -H 'Content-Type: application/json' \
  -d '{"enable_reranking": false}'
```

效果是把 `cross_encoder` 降级成 `rrf`——直接用融合后的顺序，**跳过整个交叉编码器**。关键细节：降级发生在 `recall_async` 内部，而 reflect 的内部召回也走这个函数、配置解析没有 internal 分支——**关一次，对话召回和后台刷新一起归零**。

代价是排序精度。参考维护者自己的判断：consolidation 早就主动关掉了，理由是它会把语义第 1 名的近似重复观察压到第 37 名。

**四层验证**：

| 层 | 证据 |
|---|---|
| 行为 | 同一查询、同一 300 候选池：`cross-encoder 0.836s` → `rrf-passthrough 0.002s` |
| 仪表 | 前后取重排耗时直方图：总数 +1、passthrough +1、**真调用 +0** |
| 存储 | `banks.config` 里 `enable_reranking = false` |
| 代码 | 降级点在 `recall_async` 内部，所有调用方共享 |

### 嵌入还是要留着——但它几乎免费

召回有四路（语义 / BM25 / 时序 / 图谱），前三路有开关，**语义那一路没有**，它是记忆系统的核心。但成本和重排完全不是一个量级——单价其实差不多，差别在**单次消耗量**：

```
嵌入：29 token/次      （只嵌一句查询）
重排：50,000 token/次  （要送 300 个候选）
                    ↑ 差 1700 倍
```

按「每天 100 次召回 + 50 条新记忆」算：`100×29 + 50×84 = 7,100 token/天 ≈ 21 万 token/月 ≈ ¥0.11/月`。免费额度 100 万 token 能撑约 140 天——比它的 90 天有效期还长。

**最终账单**：

| 项 | 状态 |
|---|---|
| 重排模型 | **¥0**（纯 RRF） |
| 嵌入模型 | **~¥0.1/月**（免费额度内） |
| LLM（抽取 / consolidation / 刷新） | 不变，成为唯一主要成本项 |

### 顺带修正了一个测量方法

判断「这次召回有没有真打 API」靠耗时直方图：passthrough 是纯内存排序（1~2ms），真调用是网络往返（250~2500ms）。最初把分界线划在 1 毫秒，结果一次 2 毫秒的 passthrough 被误判。**正确分界是 0.25 秒**——两档实际相差 100 倍以上，分界线划哪都不影响判断。

## 八、坑

1. **日志会骗人。** 内部调用传了 `_quiet=True`，重排日志一条不打。**只数日志得到的成本可能比真实值低一个数量级。** 锚点用外部账单，日志只做归因。
2. **改配置前先查谁覆盖谁。** 这套系统分层配置（服务端 env → bank 配置 → 模型 trigger），而刷新路径的解析函数**根本不读服务端 env**：

   ```python
   per_model = (trigger or {}).get("min_refresh_interval_seconds")   # ① 模型 trigger
   if per_model is not None: return max(0, int(per_model))
   from_bank = bank_config.get("mental_model_min_refresh_interval_seconds", DEFAULT)  # ② bank
   return max(0, int(from_bank))
   ```

   当时 bank 配置里显式存着 `0`，看着会盖掉服务端设的 `21600`。实测那只是旧默认值快照，重启后会重新 materialize。**但这一步不验证，就会得到「改了但没生效」的静默失败。**
3. **冻结类保护要问清谁能解除。** 守卫只拦自动触发，只有显式刷新成功才解冻，而显式刷新又要向量服务可用——额度没恢复前，模型一直是死的。
4. **临时故障和永久故障的判定不同。** 限流 / 5xx / 额度重置这类临时失败不设失败标记；**超时**会被当真失败并冻结。这个系统里超时是永久性判定。
5. **额度耗尽期间写入会静默丢失。** 6 条 retain 重试用尽失败，那段对话没进库。好在原文还在任务表里：

   ```sql
   select task_payload::text from async_operations
   where status='failed' and operation_type='retain' order by created_at;
   ```

   重新提交即可捞回（事实数 269 → 301）。
6. **别把两个成本源搞混。** 同一个循环烧了两份额度：内部召回的**重排调用**（云端按次）和 reflect 循环的 **LLM token**（另一个厂商）。只盯一个必然得出错误结论。
7. **连指标也会骗你，而且是同一个骗法。** `hindsight_recall_phase_duration_seconds_count{phase="reranking"}` 看着正是「重排调用次数」——但打点在 `if reranking == "cross_encoder" / else` **分支之后无条件执行**，`interleave` 的 passthrough 也被计进去。当时这个数是 293，真实 API 调用只有 38 次，**又一次高估近 8 倍**。可靠做法是拿外部账单当锚点，或用耗时直方图把两种模式分开。
8. **调用次数不是成本，token 量才是。** 这是本次最大的认知错误。两个模型单价几乎一样、调用次数也差得不多（嵌入 590 次、重排 120 次），但真实消耗差 **1700 倍**（嵌入 29 token/次、重排 50,000 token/次）。**只看调用次数归因，会把成本算反。** 凡是单次请求大小差异大的场景，都必须按 token 量归因。

## 九、可迁移的结论

1. **成本放大器公式：「每次 X 都触发 Y」×「Y 内部有 N 次昂贵调用」。** 单独看每环都合理，乘起来就是灾难。审配置先找嵌套触发，再找单次成本。
2. **用外部账单当锚点，用日志做归因。** 日志只告诉你哪些调用被记录了，账单才告诉你总共多少次。差额就是被静默掉的后台路径——通常就是真凶。
3. **「后台任务」是成本黑洞的常见位置。** 用户可见 19 次召回、后台真实 120 次重排，差 6 倍多；只数用户可见那部分，会漏掉 6/7。凡带「自动」「后台」「定时」的机制，都要单独核算它内部发起多少次外部调用。
4. **归因按 token 量，不按调用次数。** 「每次 29 token」和「每次 5 万 token」在控制台的次数列上看着同一量级，成本差 1700 倍。**调用次数是线索，不是结论。**
5. **给内部调用关掉精度开销是有依据的优化。** 同一代码库不同路径策略不一致（consolidation 关了、reflect 没关），往往是配置之外还能继续优化的点。
6. **看指标先看打点位置。** 叫 `reranking` 的计数器不代表「重排 API 被调用多少次」，只代表「代码执行到这一段多少次」。名字是别人起的，语义得自己读代码确认。
