---
title: "自建 newapi 网关收编全部 LLM 上游：部署、计费对账与 pi/Hindsight 迁移实录"
date: 2026-11-04T16:00:00+08:00
draft: false
description: "上游 key 散在三个客户端手里，一次泄漏事故后决定收敛：自建 newapi 网关的一天实录——Caddy 静默崩溃、缺定价直接拒绝服务、bge 测试误报、中断流虚账 5.5 倍，最后靠 99.8% 缓存命中证明会话头透传成功。"
tags: ["newapi", "LLM网关", "反向代理", "计费", "pi", "hindsight", "postgres", "API迁移"]
categories: ["踩坑记录"]
---

此前 LLM 上游 key 散落在各客户端：pi、Hindsight、hermes 各自持有一把上游 key，谁也管不着谁。11 月初的一次密钥泄漏事故（整份 `.env` 被写进了大模型上下文，同一账户的硅基流动余额被烧干、rerank 一度 402）之后，我决定部署 newapi 网关统一收敛上游：硅基流动（rerank）、阿里百炼（向量）、小米 MiMo（LLM）、opencode（coding LLM），客户端只持网关自己签发的 token。

原计划「SQLite 起步」，因为本机已有 postgres:16 容器、库和账号也是现成的，改成直接 PG 起步，省掉以后 SQLite→PG 的迁移。环境：new-api v1.0.0-rc.47（`calciumion/new-api:latest`，构建于 2026-10-25）、Caddy v2.11.7、postgres:16、pi、Hindsight，全部跑在 Docker Compose 上。

## 一、一天撞上的八个坑

部署与迁移在一天之内撞了一串问题，每条后文都有独立的排查：

| # | 现象 | 表象 |
|---|---|---|
| 1 | Caddy 反代崩溃循环 | `docker ps` 显示 `Restarting (1)`，**日志里没有任何 error 行**，反代的所有站点全部中断 |
| 2 | 渠道下拉没有 mimo 和 opencode | 控制台「添加渠道」翻遍了也没有这两个上游 |
| 3 | 没配价的模型拒绝服务 | 不是统计不准，是直接报错不让调 |
| 4 | 改汇率后已填单价自动变了 | 按「元」填的 0.5，汇率改完显示 0.47，像被重置 |
| 5 | bge-reranker 一点测试就报错 | `invalid embedding request type`，另一个 reranker 却正常 |
| 6 | 当天记账 $0.28、上游实收约 $0.05 | 网关统计「多出来很多」 |
| 7 | 分析页看不到分模型 token | 只有费用和调用次数，token 只有顶部一个总量 |
| 8 | pi 多行粘贴一行一提交 | 连按 8 次 Esc 才把一条消息发出去 |

## 二、排查实录

### 1. Caddy 崩溃循环，日志里没有 error

```
$ docker ps --filter name=caddy
RESTARTING (1)1 seconds ago

$ docker logs caddy --since 10m | grep -iE 'error|fatal|wrong'
（无输出——只有 maxprocs / GOMEMLIMIT / "using config from file" 三条 info 后进程就死了）
```

日志静默退出推翻了「看日志定位」的常规思路，改用官方镜像对 Caddyfile 做离线 validate：

```
$ docker run --rm -v <宿主机compose目录>/caddy/Caddyfile:/etc/caddy/Caddyfile:ro caddy \
    caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
{"level":"error","msg":"adapting config using caddyfile: parsing caddyfile tokens for 'log':
wrong argument count or unexpected line ending after 'file', at /etc/caddy/Caddyfile:4
import chain ['/etc/caddy/Caddyfile:27 (import common_config)']"}
```

两个独立错误：一是把注释挪成单独一行时，`log` 指令的 `output file` 路径和 `file` 被拆到两行（或被注释吃掉）；二是我给的片段里 `flush_interval -1` **写在了站点层**，而它是 `reverse_proxy` 的子指令，删掉即启动成功。

附带推翻一个常识预设：给 LLM 流式反代**不需要** `flush_interval -1`。Caddy `reverseproxy.go` 的源码注释写着——上游响应被识别为流式、或 Content-Length 为 -1 时该选项被忽略，写入立即 flush。响应是 `text/event-stream` 时本来就逐块推。

### 2. 渠道类型缺失，去源码里定性

```
$ curl -s https://raw.githubusercontent.com/QuantumNous/new-api/main/constant/channel.go \
    | grep -in 'opencode|mimo|xiaomi'
（无输出）
```

main 分支的渠道类型枚举到 `ChannelTypeSGLang = 63` 为止，没有 opencode；小米 MiMo 类型（PR #7493）尚未合并。二进制里能搜到的 `OpenCode`/`XiaomiMiMo` 字样是前端的图标/品牌注册表，不是渠道类型。**正解是「OpenAI 类型 + 自定义 Base URL」**，这本来就是网关的正统用法。

### 3. 缺定价 = 拒绝服务

```
模型 xxx 的价格未配置。请前往「系统设置 → 运营设置」开启自用模式，
或在「系统设置 → 分组与模型定价设置」中为该模型配置价格；
```

上游从 commit `4b439ad`（2026-03-02，"Return error when model price/ratio unset"）起**停止静默回退默认价**：没配价的模型直接报错。二进制里能确认 rc.47 含该逻辑，以及给自用场景留的 `AcceptUnsetRatioModel` 开关。配价是硬前提，不是可选项。

### 4. 改个汇率，已填的单价全变了

- 美元是记账本位（quota 内部按美元换算，`1 USD = 500000 quota`），汇率**只影响显示**；源码里的 `USDExchangeRate` 是手填项，二进制内 grep 不到任何自动刷新任务——设计如此。
- 实测当日汇率 6.68（行情 6.684 / 6.678 / 银行中间价 6.6803），此前按印象填的 7.1 是错的。
- 「改汇率后单价变」的机制：**价格以美元入库，显示按当前汇率换算**。元模式填 0.5（当时汇率 7）→ 入库 `$0.0704`；汇率改成 6.68 → 显示 `$0.0704 × 6.68 ≈ 0.47`。美元底值没变，计费不受影响——**只有填价那一刻的汇率会污染入库值**（把美元价当元填会低估 6.68 倍）。

### 5. bge-reranker 测试报错，是测试代码的启发式误判

上游 `controller/channel-test.go`（main 分支，我逐行核过）：

```go
if strings.Contains(strings.ToLower(testModel), "embedding") ||
   strings.HasPrefix(testModel, "m3e") || // m3e 系列模型
   strings.Contains(testModel, "bge-") || // bge 系列模型 → 判定为 embedding！
   strings.Contains(testModel, "embed") ||
   channel.Type == constant.ChannelTypeMokaAI {
    requestPath = "/v1/embeddings" // 修改请求路径
}
```

模型名含 `bge-` 就被猜成向量模型，测试按 embedding 格式发请求 → 类型断言失败 `invalid embedding request type`。`Qwen/Qwen3-Reranker-0.6B` 名字不含触发词所以正常。**测试按钮与真实路由是两套代码**（同族历史 bug：issue #2734 报 rerank 测试失败但实际可用，PR #2736 修）。

### 6. 当天记账 $0.28、上游实收约 $0.05

逐条拆账，不猜：

```
$ docker exec postgres psql -U postgres -d newapi -Atc \
  "SELECT id, prompt_tokens, completion_tokens, quota, other::jsonb->'stream_status'->>'end_reason'
   FROM logs ORDER BY id;"
```

| 类别 | 条数 | 消耗 | 占比 |
|---|---|---|---|
| 正常完成（`end_reason=done`，缓存命中 98–99.9%） | 12 | $0.045 | 16% |
| **`client_gone`（客户端中断）** | **8** | **$0.235** | **84%** |

8 条异常请求集中在 5 秒内：`prompt ≈ 98K、completion=0、cache_tokens=0、elapsed 2–3s`，每条按 98K 输入**全价**计费约 $0.0294。上游对账显示**上游是按缓存价收的费**——网关记账约为实收的 5.5 倍。

**触发原因是我自己**：pi 的输入框我误以为不支持多行，用终端级粘贴（右键 / Shift+Insert）贴了多行文本，换行被逐个当成 `Enter` 提交，一行一跑，我连按 8 次 Esc 中断，每按一次就产生一条全价中断请求。

根因是上游已知 bug（issue #4168，查证时仍 open）：`service/text_quota.go` 在 `usage == nil` 时用本地估算构造合成 usage，绕过零扣费保护——

```go
if usage == nil {
    usage = &dto.Usage{
        PromptTokens:    relayInfo.GetEstimatePromptTokens(),
        CompletionTokens: 0,
        TotalTokens:     relayInfo.GetEstimatePromptTokens(),
    }
}
```

客户端中断 → 读不到上游最终 usage（拿不到 `cache_tokens`）→ 按估算全额计费。受影响版本 v0.12.0+，rc.47 在坑内。方向相反的 issue #7062（中断退成 $0 但上游实际收费）说明这是结构性难题：上游完整 usage 拿不到时两边就对不齐。

### 7. 分析页没有分模型 token

从容器里抠前端产物：

```
$ docker exec newapi sh -c 'grep -aoE "spec_rank_bar.{0,120}" /new-api'
spec_rank_bar:{type:"bar",...,yField:"Count",title:{text:"Call Count Ranking"}}
```

三个图表全部硬编码 `yField:"Count"`（调用次数），只有顶部汇总卡有总量；带 token 指标的切换器在分流页的节点筛选里。上游 PR #4329、issue #5432 同需求都还 open，升级也等不来。

数据其实在 `quota_data` 表里，有按模型聚合的 `token_used` 与 `cache_read_tokens` 列：

```sql
SELECT model_name, count, token_used, cache_read_tokens, quota
FROM quota_data WHERE created_at > extract(epoch from now())::bigint - 8400;
```

### 8. pi 多行粘贴

pi 输入框其实支持多行，正确路径全在 `docs/keybindings.md`：

| 按键 | keybinding id | 作用 |
|---|---|---|
| `Ctrl+V` | `app.clipboard.pasteImage` | 走 pi 自己的剪贴板粘贴，不逐行提交 |
| `Ctrl+G` | `app.editor.external` | 外部编辑器写完整段 |
| `Ctrl+J` / `Shift+Enter` | `tui.input.newLine` | 插入换行 |
| `/hotkeys` | — | 列出全部生效快捷键 |

附带一个元教训：为多行粘贴专门抱怨过工作流别扭，当时没人去查内置 keybindings——**抱怨工具不好用时应先查它内置的能力**。

## 三、根因一句话

1. Caddy 静默崩溃 = Caddyfile 语法错误 + `flush_interval` 放错层级；日志无 error 是因为错误发生在 config 适配阶段，**validate 才是可靠探针**。
2. 渠道类型缺失 = new-api 根本没实现这两个上游；OpenAI 兼容协议 + 自定义 Base URL 是通用兜底。
3. 定价报错 = 上游 2026-03 起从「静默默认价」改为「缺价拒绝」。
4. 汇率改价变 = 单一美元本位存储 + 显示层换算，**填价瞬间的汇率决定入库值**。
5. bge 测试失败 = 测试代码按模型名启发式猜端点，与真实路由两套逻辑。
6. 计费虚高 = 上游 #4168：中断流拿不到 usage 时估算全额计费、缓存折扣丢失；叠加我的一次终端粘贴。
7. 分析页无 token = 图表 spec 硬编码 Count，功能未实现。
8. 多行粘贴 = 终端粘贴的换行被当 Enter，pi 内置的 `Ctrl+V`/`Ctrl+G` 没用上。

## 四、可复制的落地

### 1. compose（关键：不发布任何端口）

```yaml
# <宿主机compose目录>/newapi/docker-compose.yml
services:
  newapi:
    image: calciumion/new-api:latest
    container_name: newapi
    restart: unless-stopped
    environment:
      SQL_DSN: ${SQL_DSN}
      SESSION_SECRET: ${SESSION_SECRET}
      TZ: Asia/Shanghai
    networks:
      - caddy_network

networks:
  caddy_network:
    external: true
    name: caddy_network
```

要点：不发布任何端口，外部只能从 Caddy 进；数据库走宿主机地址 `SQL_DSN=postgresql://<账号>:<密码>@<宿主机IP>:5432/newapi`；`SESSION_SECRET` 用 `openssl rand -hex 32` 生成的随机值显式固定（否则每次重启全员掉登录）；HTTPS 确定后追加 `SESSION_COOKIE_SECURE=true` + `SESSION_COOKIE_TRUSTED_URL=https://<域名>`，**缺一不可**——缺了容器崩溃循环且报 `SESSION_COOKIE_SECURE=true requires SESSION_COOKIE_TRUSTED_URL`（首启踩过）。

### 2. Caddy 反代

```caddyfile
newapi.<域名> {
    reverse_proxy newapi:3000
}
```

`flush_interval` 要么放进 `reverse_proxy` 块内，要么干脆不写——SSE 默认即 flush。

### 3. 四个渠道与请求头覆盖

| 渠道 | 类型 | Base URL | 模型 | 特殊配置 |
|---|---|---|---|---|
| 小米 | OpenAI | `https://api.xiaomimimo.com/v1` | `mimo-v2.6-flash` | — |
| opencode | OpenAI | `https://opencode.ai/inference/openai/v1` | `deepseek-v4.1-flash` | **请求头覆盖（JSON）** 见下 |
| 硅基流动 | SiliconFlow(40) | `https://api.siliconflow.cn/v1` | `Qwen/Qwen3-Reranker-0.6B` | — |
| 阿里百炼 | OpenAI | `https://dashscope.aliyuncs.com/compatible-mode/v1` | `qwen3.7-text-embedding` | — |

opencode 渠道的请求头覆盖（2026-11-09 补全为三个头）：

```json
{"x-opencode-session": "{client_header:x-opencode-session}",
 "User-Agent": "{client_header:user-agent}",
 "x-opencode-client": "{client_header:x-opencode-client}"}
```

控制台里的「请求头覆盖」与「透传请求头」开关**是同一份 `header_override` 配置的两个入口**（开关的勾选状态由 JSON 解析得出，JSON 坏了开关变灰并报 `Fix the Request Header Override JSON first`）。开关一键写入的是内置 Codex / Claude Code 头模板，**不含 opencode 的头**，所以这个渠道必须手写 JSON。opencode 官方自 2026-09-06 起强制要求 `x-opencode-session`（缺失报错），该头用于 prompt cache 的会话路由。

这里有个容易被名字骗了的机制：**网关的「透传」是白名单，不是全量转发**。出站请求头由网关重新组装，除 `Content-Type`、`Accept` 这类基础头外一律不带——想让上游看到哪个头，就得在覆盖 JSON 里逐条写 `{client_header:名字}`，把入站请求头的值取出来。opencode 官方的接入要求恰好是「自报身份 + 会话粘连」两件事：`User-Agent` 别用通用 HTTP 库的名字，`x-opencode-session` 每个会话一个稳定值；`x-opencode-client` 是客户端名字位（官方 CLI 发 `cli`/`desktop`，第三方按惯例填自己的短名，如 `pi`），给上游的用量统计归因用。

三个行为细节：

- 客户端没带某个头时，对应那一行**整条不下发**（源码 `relay/channel/api_request.go` 里空值直接省略），占位符也没有「取不到就填默认值」的回退语法——上游问题单 7818 提的正是这个缺口：缺会话头会被上游 400，想要 `{client_header:名字|回退值}` 兜底，目前没有。
- 转发时的跳过清单只排除凭据类（`authorization`、`x-api-key`、`cookie`）和逐跳头（`connection`、`transfer-encoding` 等），`user-agent` 不在其中，可以正常透。
- 客户端侧也得真的带上这些头：pi 内置就会发会话头，身份头要在客户端配置里加；hermes 这类客户端把请求头烘进客户端初始化并进程内缓存，改完配置要重启客户端才生效，不会热补。

### 4. 定价：美元入库

| 模型 | 输入 | 输出 |
|---|---|---|
| `mimo-v2.6-flash` | $0.14/1M（缓存命中 $0.0028） | $0.28/1M |
| `qwen3.7-text-embedding` | ¥0.5/1M ≈ $0.075 | 0 |
| `Qwen/Qwen3-Reranker-0.6B` | $0.01/1M | 0 |
| `deepseek-v4.1-flash` | $0.30/1M | $1.20/1M |

元模式填人民币价最贴实际支出（三家人民币标价不随汇率动），opencode 填 `美元价 × 汇率`。填完做数量级验证：deepseek 10K 入 / 2K 出 ≈ $0.0108。落库形态是 `billing_mode: tiered_expr`、解码为 `tier("base", p * 0.3 + c * 1.2 + cr * 0.006)`——价格直接以美元入账，汇率彻底退出计费链路。

### 5. pi 接入（agent 目录 `models.json`，内置 provider 覆写）

```json
{
  "$schema": "https://pi.dev/schemas/models.schema.json",
  "providers": {
    "opencode": { "baseUrl": "https://newapi.<域名>/v1",
                  "apiKey": "!cat <pi agent目录>/newapi-token" },
    "xiaomi":   { "baseUrl": "https://newapi.<域名>/v1",
                  "apiKey": "!cat <pi agent目录>/newapi-token" }
  }
}
```

要点：pi 内置 mimo 的 provider id 是 **`xiaomi`** 不是 mimo；只覆写 `baseUrl` + `apiKey`，模型目录、协议、`x-opencode-session` 的发送行为全部保留内置；`!command` 形式让配置文件里没有明文密钥；`/model` 打开即重载。**覆写不改变 `/model` 清单**——菜单还是旧的，判断生效与否要看网关日志。

### 6. Hindsight 分两阶段迁移

```yaml
# 阶段1：仅 rerank
HINDSIGHT_API_RERANKER_SILICONFLOW_BASE_URL: https://newapi.<域名>/v1
HINDSIGHT_API_RERANKER_SILICONFLOW_API_KEY: ${NEWAPI_KEY}
# 阶段2：LLM + embeddings（模型名全部不动，字符串已与渠道对齐）
HINDSIGHT_API_LLM_BASE_URL: https://newapi.<域名>/v1
HINDSIGHT_API_LLM_API_KEY: ${NEWAPI_KEY}
HINDSIGHT_API_EMBEDDINGS_OPENAI_BASE_URL: https://newapi.<域名>/v1
HINDSIGHT_API_EMBEDDINGS_OPENAI_API_KEY: ${NEWAPI_KEY}
```

token 文件由我在自己终端写入、用 python 程序化注入 `.env`（命令只含路径，值全程不回显不进会话），用完即删，运行时唯一密钥载体是 `.env`（600）。改动前先 `cp` 日期后缀备份。`DIMENSIONS=1024` 首写锁死，未动。

## 五、验证

### 1. 缓存命中率是最硬的证据

```
$ docker exec postgres psql -U postgres -d newapi -Atc \
  "SELECT id, prompt_tokens, completion_tokens, quota, other::jsonb->>'cache_tokens' FROM logs ..."
11 | 93070 | 238  | 14103 | 0      ← 冷启动
12 | 93312 | 124  | 373   | 93184  ← 99.9% 缓存命中
13 | 93462 | 233  | 442   | 93312
15 | 94143 | 970  | 893   | 93952
16 | 96701 | 1293 | 1301  | 95104
```

- **99.8% 的 `cache_tokens / prompt_tokens`** 说明 opencode 按 `x-opencode-session` 精准路由到了同一 cache 分片——**会话头透传生效的铁证**（头部实测 `x-opencode-session: <pi会话UUID>`；当时 `user-agent: Go-http-client/1.1` 还是网关自己的出站身份，说明身份头当时没透——2026-11-09 补上 `User-Agent` 与 `x-opencode-client` 的透传后才完整，见四.3）。
- 这天 deepseek 正常完成的部分合计 17,597 quota，其中首条冷启动一条就占 14,103（80%），之后每轮只有几百——**长会话越聊越便宜**，冷启动才是成本大头。

### 2. 四渠道当日统计

| 模型 | 次数 | 输入 token | 输出 token | 费用 |
|---|---|---|---|---|
| `deepseek-v4.1-flash` | 8（另 8 条中断） | 564,494 | 3,139 | $0.2764（含虚账） |
| `mimo-v2.6-flash` | 4 | 177,965 | 1,546 | $0.0011 |
| `qwen3.7-text-embedding` | 3 | 54 | 0 | ≈0 |
| `Qwen/Qwen3-Reranker-0.6B` | 1 | 170 | 0 | ≈0 |

### 3. Hindsight 三链路（按 token 分账）

阶段 1（仅 rerank）：网关日志一条 `Qwen/Qwen3-Reranker-0.6B`，`token_name=hindsight`，55,469 token → quota 277（$0.01/1M 定价分毫不差）；Hindsight 侧 `300 candidates scored in 3.834s`，零报错。

阶段 2：触发 retain + reflect 后，网关出现 `token_name=hindsight` 的三类记录——`mimo-v2.6-flash`（7 条）、`qwen3.7-text-embedding`（5 条）、rerank（2 条）；`docker logs hindsight` 近 3 分钟 error 计数 0。容器连通性也顺手验了：hindsight 容器内 `fetch("https://newapi.<域名>/api/status")` → 200，DNS + TLS 直接可用，无需加网络。

## 六、坑与注意点

1. Caddy 报错不进容器日志时，用 `caddy validate` 离线定位（挂同一个 Caddyfile 跑官方镜像）；`flush_interval` 是 `reverse_proxy` 子指令，放站点层直接配置错误，而 SSE 场景根本不需要它。
2. `SESSION_COOKIE_SECURE` 与 `SESSION_COOKIE_TRUSTED_URL` 必须成对，缺一容器秒崩且是崩溃循环不是报错退出。
3. 网关缺定价 = 拒绝服务（不是记账偏差）；模型名三处必须逐字一致：渠道模型列表 = 定价模型名 = 客户端请求的 model 字段（`Qwen/Qwen3-Reranker-0.6B` 带斜杠、大小写敏感）。
4. **计费虚账的判别 SQL**：`end_reason='client_gone'` 且 `cache_tokens=0` 的行是估算全额计费，对账时剔除（上游 #4168 未修）。同类陷阱：测试按钮报错不代表链路坏（`bge-` 启发式误报），**以 curl 打真实端点为准**。
5. 元/美元填价只在输入那一刻过汇率：换显示币种后要把价格按当前币种重输一遍，否则入库值被旧汇率污染；更省事的是一直用美元填。
6. 内置 provider 覆写只改地址不改菜单：`/model` 清单不变是正常的，别据此判断失败；pi 的 mimo provider id 是 `xiaomi`。
7. pi 多行文本用 `Ctrl+V`（剪贴板粘贴）或 `Ctrl+G`（外部编辑器）；终端级粘贴会把换行当 Enter 逐行提交，每 Esc 一次就产生一条全价中断请求。
8. 网关按客户端分 token：pi 与 Hindsight 各持一把网关令牌，日志里 `token_name` 直接分账；**Hindsight 是无人值守服务，令牌额度必须无限、无过期**，否则后台记忆任务会静默失败。
9. 备份习惯：每次改 compose / `.env` 前 `cp` 日期后缀备份，密钥文件 600、目录 700。
10. 上游 key 收敛后：渠道 key 与客户端旧 key 本就不同，吊销旧 key 不影响网关；吊销后散落在各备份里的副本自动变成死值——这正是收敛 + 吊销的收益。
11. **「透传」这个词别按字面理解**：网关出站默认只发基础头，客户端带来的头要逐条写进 `header_override` 才会上游；会话头、身份头这类「软头」漏配不报错，代价是上游把你当匿名客户端——会话粘连失效、用量统计归因不到，而且现象只在缓存命中率和上游后台里看得出来。空值占位符整条不下发、没有回退语法，客户端可能不带某头的场景要留意。

## 七、可迁移的结论

**网关类软件的「渠道类型」缺失不等于不支持。** OpenAI 兼容协议 + 自定义 Base URL 是通用兜底；判断支持与否去查源码枚举（`constant/channel.go`），别信图标注册表里的字样。同理，**测试按钮和真实路由是两套代码**，误报以真实端点 curl 为准。

**对账类问题先拆结构再下结论。** 「统计多收了」这一天的账按 `end_reason` 拆成正常/异常两类后，84% 的偏差集中在一类异常行上，单条根因随后在上游 issue 里找到原文——先 SQL 拆账、再查上游 issue、最后才怀疑自己的配置。这次的顺序救了我：如果先去翻自己的定价配置，会白查一天。

**会话粘连型客户端最值钱的验证指标是缓存命中率。** 99.8% 的命中率同时证明了「头透传生效」和「计费路径正确」，比任何日志文案都硬；反过来，客户端中断是缓存统计与计费的最大破坏者——一次手滑的 8 次 Esc 就制造了全天 84% 的虚账。

**网关层的「透传」要当成白名单来配。** 出站请求头是重新组装的，客户端身份越具体（用户代理、客户端名、会话 ID），上游能给的优化就越多——缓存路由和用量归因都吃这些头；漏配不会报错，只会静默降级成匿名流量。配套地，客户端侧也得真带上这些头，而且客户端通常把请求头烘进初始化阶段缓存，改完要重启才生效。

取舍上：**上游 key 数量少、只想统一记账的个人用户**，直接单容器 + PG/SQLite + Caddy 就够，不必照搬生产模板里的 Redis（多实例共享状态才用得上）；**多 key 多客户端、需要按客户端分账**的场景，网关的 `token_name` 分账和 `header_override` 才真正值钱。反过来，如果你只是偶尔调一两个模型、也不在乎分账，继续在各客户端各持一把 key 更省事——这次的全部复杂度（定价、汇率、渠道测试、虚账对账）都是为「收敛」付的代价。
