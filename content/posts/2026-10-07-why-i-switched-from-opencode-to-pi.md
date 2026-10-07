---
title: "我为什么把 OpenCode 换成了 Pi"
date: 2026-10-07T10:00:00+08:00
draft: false
description: "OpenCode 该有的都有，我还是换成了只有 read/write/edit/bash 的 Pi。这是一次关于「掌控感 vs 开箱即用」的取舍记录。"
tags: ["AI 编码", "OpenCode", "Pi", "CLI", "工作流"]
categories: ["技术日志"]
---

## 一、那天晚上我只想让一个东西改两行配置

上周在家里的 NAS 上折腾 Caddy 的证书配置，需求具体到有点寒酸：读一个 Caddyfile，改两行，跑一次配置校验。

我打开 OpenCode，然后依次做了这些事：确认自己在哪个模式（plan 还是 build）、扫一眼侧边那一列 provider 和 MCP 的连接状态、等它在目录里把索引建完。等真正把话发给模型时，我离那个"改两行"的需求已经隔了三四次交互。

不是慢，是**仪式感太重**。

那一刻我的念头很明确：给我四个工具，然后闭嘴。

## 二、说我为什么当初选 OpenCode，现在也一样承认它好

OpenCode 是 AnomalyCo（原 SST）做的，定位是"全家桶式"的终端编码 agent：75+ provider、LSP 集成、MCP、子 agent、plan/build 双模式、多会话并行，外加桌面端、IDE 插件和终端 TUI。社区规模大概是 Pi 的十倍以上。

我用了很久，不后悔。如果今天有人问我"第一个 CLI agent 装什么"，我还是会推它——开箱即用，遇到问题搜得到答案，坑都有人替你踩过了。OpenCode ≈ VS Code：你不需要先想清楚自己的工作流，先跑起来再说。

## 三、触动我切换的瞬间

并不是某一个 bug 把我推走的。真正让我动摇的，是一个很朴素的观察：我 90% 的活儿是「读一个文件 → 改两行 → 跑一条命令 → 看一眼日志」。

这类活不需要 plan 模式，不需要子 agent 编排，不需要 MCP，也不需要 LSP——但每次都得先穿过这些功能的"门厅"。

我在为一个两行的改动付复杂度的税。而这个复杂度不是别人强加给我的，是我自己当初挑工具时主动选的。有点恼火，也有点可惜：它是好工具，只是我用它做了它压根没打算优化的小事。

## 四、实际对比

| 维度 | OpenCode | Pi |
|---|---|---|
| 定位 | 全家桶式编码产品，开箱即用 | 极简 agent harness，自己搭 |
| 内置工具 | 12+ 工具、plan/build 双模式、子 agent、LSP、MCP | 核心只有 read / write / edit / bash，其余全是可选 |
| 扩展方式 | 插件 + 配置驱动，扩展点有边界 | TypeScript 扩展、skills、prompt templates、themes，可打包成 Pi package 发 npm |
| 模型支持 | 75+ provider | 多 provider，可带本地模型；另有 print/JSON、RPC、TypeScript SDK |
| 形态 | 终端 TUI + 桌面端 + IDE 插件 | 终端为主，兼作可嵌入的库（SDK 可嵌进自己的应用） |
| 社区规模 | 大，约 Pi 的十倍以上 | 小，但方向上很稳 |
| 类比 | VS Code | Neovim |

## 五、换到 Pi 之后的真实体验

**上手成本比想象中高一点。** 第一小时我一直在找"plan 模式在哪"，答案是：没有，而且是有意不做。子 agent、权限弹窗、内置 todo 同理——官方的态度很直接：计划写进文件，权限交给容器，子 agent 你自己搭。

**爽点在"少"**。四个工具，模型拿到的东西干净，我做小事的时候不再需要穿过一堆功能。另外它不止是 CLI：print 模式可以进管道，RPC 和 SDK 能把 agent 嵌进自己的服务里，扩展是 in-process 的 TypeScript，想改行为就直接改代码。

```bash
# 无交互跑一句，结果可进管道
git diff | pi -p "给这个 diff 写一条提交信息"

# 以只读工具先探一遍仓库，不改文件
pi --tools read,grep,find,ls -p "讲清这个仓库的结构，别动文件"
```

**别扭的地方也得说清楚**：

- 生态小。搜报错经常搜不到东西，只能翻源码、翻 issue，或者直接问 agent 自己。OpenCode 那个十倍 star 的差距，实际体现在"你踩的这个坑，别人是不是已经踩过并且写下来了"。
- 没有内建权限弹窗，意味着安全边界要自己承担——它默认以启动它的用户权限运行，没有内建沙箱。对我这种把它放在隔离环境里干活的人没问题，但这是自由换来的，不是白送的。
- 扩展要写 TypeScript。不写 TS 的人，会撞到一堵墙。
- 有那么一两个下午，我确实怀念点一下就能用的 plan 模式。

## 六、卸载 OpenCode 的过程

我先把体积量了一遍，再决定删什么——删之前得知道自己在删什么：

```bash
du -sh ~/.npm-global/lib/node_modules/opencode-ai   # 二进制   ~186MB
du -sh ~/.config/opencode                           # 配置      52MB
du -sh ~/.cache/opencode                            # 缓存      10MB
du -sh ~/.local/share/opencode                      # 数据     6.1MB
```

数据目录里是 `auth.json`（登录凭证）和 `opencode.db`（历史会话库）。我没有一上来就全删，先把数据留着，观察一段时间确认没有回头的欲望。

```bash
opencode uninstall                     # 全部清理
opencode uninstall --keep-config       # 保留配置
opencode uninstall --keep-data         # 保留登录凭证和历史会话库
```

最后清掉约 **320MB**（186 + 52 + 10 + 6.1）。顺带一句：OpenCode 和 Pi 都是 TypeScript 写的、MIT 协议、开源，互不冲突，可以共存——卸载是选择，不是对抗。

## 七、结论：谁适合 Pi，谁该留在 OpenCode

**适合换到 Pi 的人**：任务以小而具体为主（读、改、跑一条命令）；喜欢 Neovim 那种"我的环境我说了算"的掌控感；愿意为一个新工具写点 TypeScript 把工作流搭出来；能接受生态小、文档短、坑要自己趟。

**该留在 OpenCode 的人**：要开箱即用；依赖 MCP、子 agent、plan 模式、LSP 这些现成能力；团队里需要统一的工具链，不可能人人为自己搭一套 harness；或者单纯不想把时间花在维护 agent 上。

我不打算写"Pi 完胜"。这两个东西解决的是不同的问题：一个替你把工作流想好，一个给你积木让你自己拼。选哪个，取决于你愿意为掌控感付多少维护成本——和十年前在 VS Code 和 Neovim 之间选，是同一道题。

我现在把 Pi 放在主力位置，但 OpenCode 的数据没删干净。真到了要做多 agent 编排的那天，我会把它装回来。
