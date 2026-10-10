---
title: "报错让你调大超时，真因是模型的思考 token"
date: 2026-10-10T10:40:00+08:00
draft: false
description: "接连 5 次刷新超时，报错还建议把墙钟调大；真凶是思考型模型思考期间不吐字节，被读超时判死"
tags: ["超时排查", "LLM", "reasoning", "Hindsight", "成本治理"]
categories: ["踩坑记录"]
---

> 文中时间均为北京时间。

上一篇刚给记忆服务的后台刷新加了 6 小时节流，转头我又干了件自以为省钱的事：把两个心智模型的刷新从 `full` 切成官方知识页推荐的 `delta`。

```json
{"mode": "delta", "fact_types": ["observation"],
 "exclude_mental_models": true, "refresh_after_consolidation": true}
```

`delta` 只读「上次刷新之后变动过的记忆」，对现有内容做外科手术式编辑；`full` 每次重读全量、从头重写。听起来稳赚。

切完之后，接连 5 次刷新全部失败（前 3 次是默认配置，后 2 次分别在调大墙钟、调大读超时之后），报错原文：

```
Task exceeded the 300s wall-clock limit for 'refresh_mental_model'
(stage=llm.openai.mental_model_delta_ops.attempt=1/4.backoff) and was cancelled.
Raise HINDSIGHT_API_REFLECT_WALL_TIMEOUT if this is a legitimately long operation,
or set it to 0 to disable the limit.
```

最后一行是引擎给的修复建议：把墙钟调大。我照做了，掉进两个坑——而这两次「按报错提示调参」，正好把整轮排查的过程摊开了。

## 一、越调越糟的两次尝试

| 时间 | 动作 | 结果 |
|---|---|---|
| 08:58:21 | delta 首次刷新 | 300s 墙钟超时 |
| 09:40:18 | 重试（同时有别的任务在跑） | 300s 超时 |
| 10:01:31 | 干净环境重试，排除抢资源 | 300s 超时，假设一推翻 |
| 10:09:46 | 墙钟 300s → 600s | 600s 照样超时：10 分 08 秒只完成 5 次工具调用 |
| 10:21:38 | 读超时 120s → 300s | **更糟**：只完成 1 次工具调用就耗尽墙钟 |

失败还有个副作用：`last_refresh_failed_at` 晚于 `last_refreshed_at` 时模型会被**冻结**，代码注释写着这个字段 "is what stops the automatic triggers" —— 它是熔断器，防止配置一直失败还一直烧钱。代价是心智模型停更了 1.5 小时（08:58–10:36）。

调大读超时那一步，是翻代码时看到全局默认只有 120 秒，而且有条关键注释说明它**是读超时、不是总时长**：数据只要还在持续流出就不算超时，只有窗口内一个字节都没有才判失败。库里确实躺着一条 **482 秒却成功**的调用，印证了这点。

但方向对了一半，操作是反的：读超时调大，等于让一次真挂住的调用吃满 300 秒才判失败，重试次数变少，600 秒墙钟反而更快被耗尽。

## 二、先花 5 分钟把服务商排掉

两次都死在「调用失败 → 退避」，开始怀疑服务端。不再猜，直接绕过所有中间层探测同一个端点：

```bash
BASE=$(docker exec <容器名> printenv HINDSIGHT_API_LLM_BASE_URL)
KEY=$(grep -E '^<LLM_API_KEY>=' <compose目录>/.env | head -1 | cut -d= -f2-)

curl -s -o /tmp/r.json -w "%{http_code} %{time_total}\n" -m 120 \
  -X POST "$BASE/chat/completions" -H "Authorization: Bearer $KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"<模型A>","messages":[{"role":"user","content":"say hi"}],"max_tokens":8}'
```

小请求 1.43s / 40K 字符 prompt 3.56s / 带 tools 3.15s / 流式首字节 0.139s，五种形态全部秒回。**服务端没问题**，最大嫌疑方向被排除，注意力拉回自己的配置。

## 三、真线索是数据库里的一列

回看 `llm_requests` 表结构，有个一直没注意的字段 `thoughts_tokens`。查它：

```sql
select scope, count(*) as calls,
       sum(coalesce(thoughts_tokens,0)) as thinking,
       sum(output_tokens) as output
from llm_requests where started_at > now() - interval '6 hours' group by 1;
```

| scope | 调用 | 思考 token | 输出 token |
|---|---|---|---|
| consolidation | 27 | 58,375 | 35,360 |
| reflect_tool_call | 29 | 14,693 | 9,534 |

**思考比可见输出还多。** 单次最长的两条：思考 6,448 / 输出 2,243 / 耗时 218 秒；思考 5,506 / 输出 2,320 / 耗时 482 秒。

返回体里也确认了字段：`message keys: ['content', 'role', 'tool_calls', 'reasoning_content']`。

**根因一句话：LLM 是思考型模型，思考阶段长时间不产出可见输出，被客户端读超时判为失败；重试与退避叠加，最终撞上外层墙钟。** 报错里那句 `Connection error (HTTP None): TimeoutError` 完全是伪装——不是网络有问题，是模型在思考。

## 四、解法：按路径关思考，不是全局关

`reasoning_effort` 各档实测（同一道推理题）：

```
{}                                 2.68s  思考=101  输出=140
{"reasoning_effort":"low"}         2.80s  思考=27   输出=72
{"reasoning_effort":"medium"}      2.67s  思考=62   输出=135
{"reasoning_effort":"high"}        4.53s  思考=128  输出=215
{"reasoning_effort":"none"}        2.30s  思考=0    输出=81
{"enable_thinking": false}         思考=108          ← 没关掉
{"thinking":{"type":"disabled"}}   思考=0            ← 这个也有效
```

Hindsight 有现成的 env（全局 + 按操作各一个），默认全部未设，等于用 provider 默认值——思考开启：

```yaml
environment:
  # 思考期间不产出可见输出 → 读超时 → 重试 → 墙钟耗尽
  HINDSIGHT_API_MENTAL_MODEL_REFRESH_LLM_REASONING_EFFORT: "none"
  HINDSIGHT_API_REFLECT_LLM_REASONING_EFFORT: "none"
  # 排查期临时调大的两个值，关思考后改回默认
  HINDSIGHT_API_REFLECT_WALL_TIMEOUT: "300"
  HINDSIGHT_API_LLM_TIMEOUT: "120"
```

改完重建容器约 25 秒，`docker exec <容器名> printenv | grep REASONING_EFFORT` 确认生效。

**刻意不关的两条**：`CONSOLIDATION`（观察合并）和 `RETAIN`（事实抽取）保持思考。理由是它们没有工具循环兜底，而且是记忆质量的源头——而刷新/反思有工具循环，模型可以边查边改，少想几步不会塌。关完我专门回读了一次调用记录确认：consolidation 的思考 token 仍在 1,800 上下，确实没被牵连。

## 五、验证：同一个阶段，唯一变量是思考开关

```
10:34:53  reflect_tool_call            1.5s   2,829 in   思考 0    84 out
10:35:13  reflect_tool_call           26.8s  19,327 in   思考 0 2,681 out
10:35:40  reflect                     10.6s   2,469 in   思考 0 2,371 out
10:35:55  mental_model_delta_ops      10.9s   9,212 in   思考 0 1,387 out  ← 之前超时的阶段
```

**76 秒跑完**，此前 5 次全部超时。

| | 修复前 | 修复后 |
|---|---|---|
| 工具调用耗时 | 5 ~ 93s | **1.5 ~ 3.3s** |
| `mental_model_delta_ops` 阶段 | >300s 超时 | **10.9s** |
| 整轮刷新 | 连续 5 次失败 | ✅ 76 秒 |
| 单次成本 | ¥0.1138（full 基线） | **¥0.0614**（省 46%） |

其它验证点：记忆内容确实被编辑了（6,017 → 6,914 字符，不是空跑）、delta 水位线正常推进、`last_refresh_failed_at` 为空没冻结。

## 六、踩到的坑

- **报错给的建议本身可能是陷阱。** `Raise ..._WALL_TIMEOUT` 是引擎的通用文案，它只知道「任务超时了」，不知道为什么超时。顺着调只会浪费轮次——我浪费了两轮。
- **「调大超时」在「失败重试」场景下会起反作用。** 读超时越大，一次挂住的调用吃掉的时间越多，重试次数越少，墙钟越快耗尽。
- **分清「总时长超时」和「读超时」，两者解法方向相反。** 前者靠加大预算，后者要靠让产出更连续。判断方法很简单：找一条耗时远超超时值却成功的调用，存在即为读超时。
- **`Connection error` / `TimeoutError` 会伪装。** 真实原因可能只是「模型在想，没输出字节」。
- **失败的代价不只是那一次的钱**，熔断器会停掉整条自动更新链。
- **思考 token 按 output 计费，且可能比可见输出还多。** 只统计 `output_tokens` 的成本核算会漏掉一大块。
- **别一刀切全关。** 只关有工具循环兜底的路径，保留质量源头。

## 七、可迁移的结论

1. **换用「带思考」的模型时，所有基于时间的假设都要重算。** 响应变慢不只是慢一点，它会让原本安全的「读超时 + 重试 + 墙钟」组合全面失效。
2. **不要相信报错里的修复建议，要相信对照实验。** 这里的决定性证据是同一阶段、唯一变量是思考开/关，耗时从 >300s 变成 10.9s。
3. **怀疑外部服务前，先花 5 分钟直接探测它**，用 curl 绕过所有中间层测同一端点的多种形态。这一步排除了最大的嫌疑方向。
4. **谁该关思考、谁不该关**：有工具循环、允许模型边查边改的路径（刷新、反思）可以关，换稳定性和速度；无工具兜底、决定内容质量的路径（合并、抽取）留着，别为省 46% 的单次成本拿记忆质量去换。回退阶梯也留好了——`reasoning_effort` 支持 `none/low/medium/high`，改一个 env 重启 25 秒就能退回。
