---
title: "Hermes 0.22 升级实录：三次验收全绿，三个坑全是静默的"
date: 2026-10-14T18:30:00+08:00
draft: false
description: "插件依赖只写失败、压缩配置键换含义、目录悄悄涨 3.6G——三个问题没有一个会报错，只在日志里留一行。"
tags: ["hermes", "hindsight", "升级", "踩坑记录", "压缩", "限流"]
categories: ["技术实践"]
---

今天一次性把三件套升完：Hermes 0.21.5 → 0.22.0、Hindsight 0.10.3 → 1.0.0、Hindsight 插件 1.2.1 → 1.2.2。镜像 tag 命名也换了代，从日期式变成语义化版本，升级前得去 registry 的 tag 列表确认，别按历史习惯拼。

麻烦的是，这三次验收全是绿的，问题全在后面两个小时里陆续冒出来。

## 一、验收清单跑了什么

```bash
hermes --version        # Hermes Agent v0.22.0 (2026.9.24) · upstream e06df9c2
grep -i hindsight /opt/data/logs/agent.log | tail
                        # … initialized … client=0.10.3  ← 写路径的关键一行，见第二节
hermes cron list        # 3 个 job 仍 active
hermes config check     # 改完配置后应为 0 警告
```

Python 从 3.13 跳到 3.14.7，我先排了影响面：定时任务脚本全是 stdlib，不受影响；只有几个手动脚本依赖 pg8000，它是纯 Python 包，3.14 下 import 正常。带二进制 wheel 的包才是升级风险点——只看「能不能跑」不够，得先看它装在哪一层。

两个默认值变化也逐条对了本地：`sessions.auto_prune` 默认开启，但实测库里 133 个会话、符合「已结束且 90 天无活动」的是 0 条，对我是空操作；`session_reset` 的 idle 分钟数从来没生效过（路由函数只认挂起原因，上游 issue 还开着），我的会话轮换一直靠手动开新会话。

## 二、坑一：插件依赖静默丢失，记忆只读不写

升级后半小时，日志里出现这么一行：

```
WARNING Hindsight retain failed: No module named 'hindsight_client'
```

麻烦在于**只有写路径坏**：记忆注入、召回、统计全都正常，只有落库失败。只验读路径就会得出「一切正常」的错觉。

根因是一条完整的静默链：

- 插件代码在挂载卷上，升级不丢；但它的 Python 依赖只在 `hermes plugins install` / `hermes update` 时装
- 换镜像升级这三条路一条都没跑
- 启动钩子只做 UID、卷、配置、技能同步，不装依赖
- discovery 的警告机制读 `manifest.python_dependencies`，而插件的 plugin.yaml 故意不写依赖（包管理器读的是 pyproject）——警告一次都没触发

修复，0.22.0 起不需要 root（新包管理器把依赖装进挂载卷，容器内普通用户可写）：

```bash
hermes plugins install hindsight --force --yes-deps --enable
docker restart <容器名>       # 必须重启：活进程不会自己发现新环境
```

验证两步：启动行要出现 `client=0.10.3`（重启前是 `client=unknown`），再 `grep -c "retain failed"` 应为 0。这一步我已经写进升级 runbook 的恢复段——备份如果排除了 `installs/`，还原后必须补跑这两条，否则会得到「备份成功、恢复成功、但记忆在静默丢失」。

## 三、坑二：一个配置键的含义，换来一堆 429

修完依赖当晚，API 开始返 429。我一开始以为是调用太密，把 60 秒滚动窗口算出来才发现不是：**峰值一分钟 140 万 token、5 次调用**，全天 159 次调用、输入合计 3495 万 token。瓶颈是单请求体积——当时每次调用要重发 30 万 token 的上下文。

那压缩为什么没触发？这里是个挺典型的坑：

| 键 | 含义 |
|---|---|
| `compression.threshold` | **比例**（0.50 = 窗口的 50%），两个版本都一样 |
| `compression.threshold_tokens` | 绝对 token 上限；旧版本默认 256000 |

我的配置里写的是 `threshold: 256000`——把 token 数写进了比例键，渲染出来是这样：

```
$ hermes config show
◆ Context Compression
  Threshold:    25600000%
  Token cap:    (无)
```

25600000% 当然永远不触发。**旧版本靠那个默认绝对值兜住了**，所以这个错一直没暴露；而 0.22.0 的迁移把默认值删了，上游的理由写在迁移注释里——这个默认值会把 1M 窗口的模型硬压在 256K。兜底一撤，我的配置错误第一次裸奔，上下文一路涨，重发体积越来越大，429 越吃越频繁（今天 18 个时间点），自动降级也跟着触发了 5 次，目标是上一代模型——而它这个月就要下线。

两行修好，这次是写到正确的键上：

```yaml
compression:
  threshold: 0.5              # 比例 = 窗口 50%（1M 窗口 → 524k）
  threshold_tokens: 256000    # 绝对上限，取两者较低 → 256k 触发
```

热加载生效，当天就压了：

```
17:46:03 Preflight compression: ~324,175 tokens >= 256,000 threshold
17:47:26 context compression done: messages=403->90 rough_tokens=~61,708
```

压缩前主模型输入中位数 247,250，压缩后头几轮掉到 6.5 万–7.4 万。

同时我把 fallback 删了——限流只重试不降级。取舍是：自动降级保「随时能出话」，代价是模型被静默换掉；重试保模型一致，代价是高峰期要等退避。我选后者，因为降级目标一旦下线，这条路会直接断掉。

## 四、坑三：目录悄悄涨了 3.6G，备份排除清单得跟着改

升级当天 `/opt/data` 涨到 6.4G，`du` 逐个子目录对账，多出来的主要是新包管理器的环境目录 `installs/`：

```
3.6G  installs/                       ← 今天新建
  1.5G  environments/<A>/workspace/   ← hermes 仓库快照，其中 tools/chromium = 258M
  485M  environments/<A>/venv/        ← googleapiclient 95M、botocore 29M …
  1.5G  environments/<B>/workspace/   ← 同上，整套工具链复制了第二份
  485M  environments/<B>/venv/
```

结构上是**每个 PM 环境自带一份可重定位快照**——仓库、node、ffmpeg、python、chromium 全复制进去。好处是环境能整体搬走，代价就是 Chromium 这类大件按环境数翻倍，我这里恰好有两个环境（一次插件安装、一次重启各建一个），于是 chromium 258M×2、整套工具链 ×2。这不是配置错误，是设计如此，但升级前没人提醒你它会占 GB 级。

排除清单该按「**能不能重建**」分，不按大小分：

| 排除 | 量级 | 重建方式 |
|---|---|---|
| `installs/` | GB 级 | `hermes plugins install <name> --force --yes-deps` + 重启 |
| `cache/`、`home/.cache/` | 百 MB 级 | uv/scratch 全自生成 |
| `.venv/`、`bin/`、`lsp/`、`.npm/` | 百 MB 级 | 重新下载/装 |

顺手清掉两块死重：一个早期懒装机制遗留的目录（477M，代码里 grep 不到任何引用），以及我已经弃用的 opencode 全套，一共回收 1.4G，`/opt/data` 从 6.4G 回到 5.0G。

真正必须备份的，只有「换了机器就没了」的：会话库、配置与凭据、技能、定时任务注册表、deploy key、各 CLI 的 OAuth 会话。

## 五、三次验收都是绿的

版本对、插件注册对、定时任务对、网关对——问题全出在「配置语义」和「派生状态」上：一个键换了含义、一批依赖属于可重建状态、一份旧清单没跟上。

所以升级前值得单独做一件事：把 Upgrade notes 里两类条目逐条对到本地——**默认值发生变化的键**、**被迁移删除的键**。前者决定你是不是在用旧理解配新版本，后者决定你有没有一段功能悄悄归零。光跑验收清单，抓不到这两种。

取舍上：如果你的部署是「换镜像」而不是「跑安装器」，就要接受「依赖缺失」这类静默失败是常态，验收清单里必须显式包含**写路径**测试；如果你一直用官方默认配置，这两个坑大概率都遇不上——它们专咬长期没动过、又恰好和新默认值撞名的老配置。
