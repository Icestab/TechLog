---
title: "Hermes 0.23 升级实录：一次没踩坑的升级，和清单跑出来的 11 行"
date: 2026-11-06T11:30:00+08:00
draft: false
description: "Upgrade notes 只有 5 条、验收全绿，但十个浏览器工具被删这件事，只有翻自己的 skill 才会发现。"
tags: ["hermes", "升级", "踩坑记录", "hindsight"]
categories: ["技术实践"]
---

[上次 0.22 的升级](/posts/2026-10-14-hermes-022-upgrade/)，三个坑没有一个会报错：插件依赖静默丢失、压缩配置键换了含义、目录悄悄涨 3.6G。当时立了条规矩——升级前逐条核对 Upgrade notes 里的「默认值变化」和「被迁移删除的键」，这两类问题验收清单全绿也抓不到。

0.23.0（The Hearth Release）赶在 11 月 6 日清晨发布：北京 05:32 出 release，一次收编 0.22.1 到 0.22.7 七个补丁版，从 0.22.0 数上去约 10,080 次提交。我把同一套清单原样跑了一遍。这次确实没踩坑——但清单没覆盖的地方，翻出来一个真问题。

## 一、升级前的 20 分钟：备份、通读、镜像核实

三件事按顺序做，没有一步能省：

```bash
cp -p config.yaml  config.yaml.bak-v0230pre-$(date +%Y%m%dT%H%M)
cp -p .env         .env.bak-v0230pre-$(date +%Y%m%dT%H%M)
# release notes 全文抓下来存档，后面逐条对表用
curl -s https://api.github.com/repos/NousResearch/hermes-agent/releases/tags/v0.23.0 > notes.md
```

release notes 全文 8 万字，但真正要命的「Upgrade notes」一节只有 **5 条**。逐条对完的结论是：**没有配置键被删除，没有默认值被改**——0.22 那种「迁移把 `threshold_tokens` 删回 null、压缩静默失效」的事故这次不存在。这条判断本身就不许跳过：它是空跑，但空跑的结果和没跑是两回事。

镜像侧能查的先查：registry 的 config blob 里 Python 版本没暴露出来，就留作升后第一件事实测——venv 是按 3.13 建的，cron 脚本的 pg8000 走的是它，这是升级最可能打断的链路。

## 二、Upgrade notes 对表：5 条里只有 1 条真咬人

| 条目 | 对我的影响 |
|---|---|
| 旧内置浏览器工具全删，只剩 `browser_exec` | **中招**：9 个自装 skill 还在引用旧名 |
| Desktop 要 macOS 13+ / 64 位 Windows | 不用 Desktop，无关 |
| `hermes mcp serve` 改名 `mcp-serve` | 没有外部客户端拉起，无关 |
| Word/PPT/Excel 三个 skill 合并为 `microsoft-office` | 旧的留盘上，可后清 |
| Webapp 默认关闭 | 不开就什么都不对外服务 |

第一条是这次唯一的真雷。Hermes 0.23 删掉了 `browser_navigate`、`browser_click`、`browser_snapshot` 等十个工具，而我写的经验型 skill 里到处是它们的用法——这些引用不会让任何构建变红，只会在将来某次照着 skill 操作时变成「工具不存在」。

## 三、真正的迁移藏在 skills 里：4 个被动、11 行手工

升级启动后，`skills/` 下 4 个文件（dogfood、grounded-citations、computer-use、product-price-monitor）的引用**自己变成了 `browser_exec`**。怎么确认是升级改的、不是我记错了？看 mtime：

```bash
stat -c '%y %n' skills/autonomous-ai-agents/computer-use/SKILL.md
# 2026-11-06 04:02:06 +0800 —— 恰好落在镜像构建收尾（20:03 UTC）前一分钟，
# 即文件在镜像里就写好了、容器启动时被同步进数据卷，不是运行期谁改的
```

剩下的 **5 个文件、11 行**得手工改，分布是：sketch 5 行、hugo-blog 2 行、cn-forum-scraping 2 行、airtable 1 行、skill-authoring 1 行。

难点在于这**不是改名**：`browser_exec` 是个跑 Python 代码的工具，原来的两步调用得重写成代码：

```
# 旧
browser_navigate(url="file:///.../sketches/001-calm-editorial/index.html")
browser_vision(question="布局有没有重叠、字体加载失败?")

# 新
browser_exec(code="""# 打开草图变体并截图目检
goto_url("file:///.../sketches/001-calm-editorial/index.html")
wait_for_load()
print(capture_screenshot())""")
```

唯一的例外是 hugo-blog 里那条 2026-08-09 的误报复盘——它记录的是**当时发生的事**，旧工具名是史料本身。处理原则：历史叙述保留原文，行首加注说明「旧工具名在 0.23 已删」，只把其中可执行的 Rule 改成新工具的说法。史料不动，行为规则更新。

## 四、插件更新：一次被拒绝的 update，和「必须重启」的老机制

hindsight 插件有新版（1.2.2 → 1.2.4），但 `hermes plugins update` 直接拒绝：

```
Non-interactive install — skipping dependency install.
Error: Update declined: dependency install skipped (non-interactive).
```

非交互模式下依赖不会装，更新就中止——这个设计是对的，装一半才是灾难。正确姿势是 0.22 事故那次换来的：

```bash
hermes plugins install hindsight --force --yes-deps --enable
docker restart <容器名>                      # 不确定名字就先 docker ps 看第一列
# 重启后必须看到这一行，client=unknown 就是依赖没引导进去
grep "Hindsight initialized" /opt/data/logs/agent.log | tail -1
# → 2026-11-06 11:44:20 … client=0.10.3
```

为什么非要重启：依赖落在 `/opt/data/installs/<sha>/environments/<id>/` 里，网关进程只在启动时引导环境，不重启就一直跑内存里那份旧代码。0.22 的教训是换镜像绕开了 install 路径，retain 只在日志里留一行 WARNING 就没了——所以这次的验收标志位定得很死：**`client=0.10.3`，且 errors.log 在重启后零新增**（重启前最后一条报错停在 11:41:19，是插件文件被替换那几秒里客户端拿到的瞬时 500，重启后干净）。

## 五、验收清单：全绿，外加一个假警报

```bash
hermes --version        # v0.23.0 · upstream 2bd378ed
python3 --version       # 3.14.9（原 3.14.7），venv 3.13.5 + pg8000 1.31.5 导入正常
hermes cron list        # 4/4 active
grep -c "sk-" config.yaml   # 0 —— 配置里没有明文 key
```

自建网关 `/api/status` 200、dashboard 302、QQ 消息流正常（对话本身就是证据）。账单脚本按真实路径端到端跑了一遍——**第一次报 EXIT=1，查下来是 `| head -6` 提前关管道的假象**，重跑干净通过：校验退出码别经过会截断输出的管道，这个错我记下了。

账单侧还有个巧合的彩蛋：当天窗口正好在升级后闭合，日报 B 节给出了「中断重算口径」上线后的第一次真对账——网关重算 $0.1001 vs 上游实收 $0.1006，**差 0.5%**。

磁盘从 0.22 时的 6.4G 回到 **2.8G**：PM 环境随升级重建，老 installs/ 的 3.6G 被清掉了，这次升级反而瘦了身。

## 六、下次照抄什么，谁别照抄

这次值得固化的三条：

1. **升前逐条对 Upgrade notes 的「默认值变化 / 被删键」**——这次是空跑，但 0.22 的坑恰恰是这类；
2. **release notes 通读全文再动容器**，备份和镜像核实顺手做掉；
3. **验收看标志位，不看「没报错」**：插件看 `client=`、脚本端到端真跑一次、报错日志比对重启前后的时间戳。

适合照抄的是**单机 Docker + 数据卷分离**的结构：镜像随便换，配置、脚本、技能全在卷里，升级就是改个 tag。两处要留神：`installs/` 里的插件环境**跟着镜像重建**，恢复备份时必须重装插件依赖，否则记忆静默失效；skill 的自动迁移只覆盖镜像内置的那批，自己写的经验文档没人替你改。

明早 08:20 是新环境下第一次自动账单推送，跑完没问题，这次升级才算真正画上句号。
