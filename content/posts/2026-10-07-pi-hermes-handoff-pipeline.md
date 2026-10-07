---
title: "Pi 干活，Hermes 写博客：把两个 agent 打通成一条交接流水线"
date: 2026-10-07T13:30:00+08:00
draft: false
description: "现场解决问题的 Pi 和常驻容器里的 Hermes，靠什么把「过程」交出去？一次不新开任何端口的 agent 交接实践：通道选型、会话语义、端口映射打不通的反直觉坑，以及给 agent 写交接 prompt 时比格式更重要的三条纪律。"
tags: ["Hermes", "Pi", "Agent", "工作流", "Docker", "API"]
categories: ["技术实践"]
---

> 我手里有两个 agent，它们的短板正好互补。Pi 贴着现场干活，工具只有 read / write / edit / bash，能直接摸到宿主机的真实环境——但它干完活就走了，过程散在终端里。Hermes 常驻在 Docker 里，接着 QQ，技能库里已经躺着整套博客发布流程——但它不在现场，看不到报错，也复现不了问题。这篇记录的是把它们接成一条交接流水线的过程：通道怎么选、会话怎么续、以及几个「看起来能用、其实连不上」的坑。

---

## 一、目标：把「现场记录」交棒，而不是把「结论」复述一遍

两个 agent 的形态差异，决定了它们的短板是互补而不是重叠：

| | Pi（现场） | Hermes（常驻） |
|---|---|---|
| 运行位置 | 宿主机上，贴着待排查的环境 | Docker 容器内，由 Compose 常驻 |
| 工具面 | read / write / edit / bash | 完整工具集 + 技能库 |
| 优势 | 能直接操作真实环境、拿到第一手报错 | 随时在线、可被消息唤起 |
| 短板 | 会话一结束，过程就散了 | 不在现场，无法复现问题 |
| 已有能力 | —— | 博客技能已覆盖构建、脱敏扫描、Pages 发布、线上复核 |

想要的效果很简单：Pi 解决完问题，把过程「投」给 Hermes，让它按站内既有流程写成文章。

人工搬运（复制报错、重述排查过程、再拼一遍上下文）显然不达标——要的是把现场记录**直接送过去**，而不是让人在中间当搬运工。

---

## 二、先摸清 Hermes 那一侧有什么通道

### 2.1 容器里开着两组监听

容器发布了两组端口，进容器各打一遍就知道它们分别是什么：

```bash
# 服务健康检查
curl -s -o /dev/null -w '%{http_code}' http://localhost:<API端口>/health            # 200
# 模型列表，需要鉴权
curl -s -o /dev/null -w '%{http_code}' http://localhost:<API端口>/v1/models        # 401
# Dashboard
curl -s -o /dev/null -w '%{http_code}' http://localhost:<Dashboard端口>/api/health # 200
```

结论很清楚：一组是 OpenAI 兼容的 API server（`/v1/chat/completions`），另一组是 Dashboard。

### 2.2 三条候选通道

| 通道 | 做法 | 取舍 |
|---|---|---|
| 把 API server 绑到全网卡，宿主机直连 | 改监听地址 | ❌ 否决。宿主机上的 agent 不需要为此把接口暴露出去，保持回环是刻意的安全边界 |
| CLI 一次性调用 | `docker exec` 进容器跑带 `--oneshot` 的聊天命令，用 `--query-file -` 从 stdin 读素材 | ✅ 实测可用，二进制安全、不做 shell 解释；但每次都是独立的 CLI 会话，和常驻 gateway 的会话库、记忆域不在一条线上 |
| 入容器打回环上的 OpenAI 兼容 API | `docker exec` + 容器内发请求 | ✅ **最终选择**，因为它有会话语义 |

#### CLI 通道也实测过

```bash
printf 'Reply with exactly: PONG\n' | docker exec -i hermes hermes chat --query-file - --oneshot -Q
```

原样返回：

```
PONG

session_id: 20261007_140336_2ed355
```

`--query-file -` 从 stdin 读正文、不做 shell 解释；`--oneshot` 表示答完即走；`-Q` 抑制横幅和工具预览。注意最后那行 `session_id`：它属于 CLI 自己的会话体系，和 gateway 的会话库不是同一个——这正是上表里「不在一条线上」的具体含义，也是它最终没被选中的原因。

选第三条的关键在于那两个请求头：

- `X-Hermes-Session-Id`：续接同一个会话。历史从服务端状态库载入，而不是靠请求体把上下文回灌一遍
- `X-Hermes-Session-Key`：给长期记忆划一个独立域。它和会话 id 是正交的——key 跨会话稳定，id 会轮换
- 响应头里会回传 `X-Hermes-Session-Id`，客户端存下来，下次接着用

换句话说：走这条通道，投喂过去的不是一段「无状态文本」，而是「同一个会话里的下一句话」。

### 2.3 反直觉的一课：端口映射了 ≠ 能连上

compose 里明明写着端口映射，但从宿主机打过去：

```bash
curl -s -o /dev/null -w '%{http_code}' http://localhost:<API端口>/v1/models
000      # 连不上
```

原因不在网络策略，而在**监听地址**：adapter 只 bind 容器内的回环地址，而 Docker 的端口发布是从容器外部（网卡地址）转发进来的——于是这条映射实际上打不通。启动日志里也能对上，它明确打印了监听在回环地址上。

而这个结果恰恰是**想要的暴露面**：对外不可达，只允许容器内的进程访问。代价是宿主机侧必须 `docker exec` 进容器再发请求，而不是「映射了就能直连」。

> 映射是「给外部留的口子」，不是「内部可用性的保证」。这两件事在 Docker 里经常被混为一谈。

### 2.4 踩坑：第一次请求 401

修正前的写法，是进容器用 `sh -lc` 执行一行 `curl`，把 key 以 `$API_SERVER_KEY` 展开：

```bash
$ docker exec hermes sh -lc 'curl ... -H "Authorization: Bearer $API_SERVER_KEY" ...'
{"error": {"message": "Invalid gateway API key (API_SERVER_KEY)", "type": "gateway_auth_error",
           "code": "gateway_auth_failed"}}
```

原因不是 key 错，而是**测试写法错**：那个 shell 里 `$API_SERVER_KEY` 是**空的**，等于发了一个空的 Bearer。真正的 key 存在容器的 `.env` 文件里，由应用启动时自己加载，**不在容器进程的环境变量里**（`docker inspect` 的 `.Config.Env` 里看不到它）。

教训：不要把「宿主机上有」「容器 `.env` 里有」「`exec` 进去的 shell 里有」当成同一件事。修正的做法就是在容器内读 `.env` 取值：

```bash
$ KEY=$(docker exec hermes sh -lc 'grep -m1 "^API_SERVER_KEY=" <数据目录>/.env | cut -d= -f2-')
$ echo "key len: ${#KEY}"
key len: 64
```

同一个请求随即返回 `"content": "OK"`。顺带一个有信息量的数字：这次调用的 `usage` 里提示 token 数是 **20824**——系统提示 + 记忆 + 工具 schema 的固定开销，这就是「把 agent 接进来」的真实成本量级。

### 2.5 会话续接实测

同一个会话里问两轮：

| 轮次 | 输入 | 输出 |
|---|---|---|
| turn 1 | 这是通过管道发来的测试。请只回复两个字：收到 | 收到 |
| turn 2 | 我刚才让你回复什么？只回复那两个字。 | 收到 |

两次响应头里的 `X-Hermes-Session-Id` 一致，说明上下文确实续上了，而不是每次无状态重放。

---

## 三、方案：宿主机上的两个脚本

两个脚本都写在宿主机，流程统一为：`docker exec` 进容器 → 容器内读 `.env` 取 key → 向回环上的 API 端点 POST。**key 全程不落宿主机磁盘。**

**`hermes-feed.sh`——通用文本管道。** stdin / 文件 / 参数进，回复出。会话 id 持久化在本地状态文件里，多次投喂自动续接同一会话；支持开新会话、指定会话、指定记忆域、指定模型、设置超时。

管道内部有两个设计点，值得写下来：

**① key 只在容器内出现。** 容器里的 python 自己打开 `.env` 取值，宿主机侧从头到尾没有存过 key 变量——它既不会落进宿主机磁盘，也不会进 shell 历史。

```python
# ① key 从容器内的 .env 读，不走环境变量
key = ""
with open("<Hermes 数据目录>/.env", encoding="utf-8", errors="replace") as fh:
    for line in fh:
        if line.startswith("API_SERVER_KEY="):
            key = line.split("=", 1)[1].strip().strip('"')
            break

# ② 会话与记忆域靠请求头，不是请求体
headers = {"Content-Type": "application/json",
           "Authorization": "Bearer " + key}
if session_key:
    headers["X-Hermes-Session-Key"] = session_key   # 长期记忆域，跨会话稳定
if session_id:
    headers["X-Hermes-Session-Id"] = session_id     # 续接上一次会话

# ③ 走 OpenAI 兼容端点；显式禁掉代理环境变量的干扰
body = json.dumps({"model": "hermes-agent",
                   "messages": [{"role": "user", "content": text}]}).encode("utf-8")
req = urllib.request.Request("http://<回环地址>:<API端口>/v1/chat/completions",
                             data=body, headers=headers, method="POST")
opener = urllib.request.build_opener(urllib.request.ProxyHandler({}))
with opener.open(req, timeout=timeout) as resp:
    data = json.loads(resp.read().decode("utf-8", "replace"))
    sid = resp.headers.get("X-Hermes-Session-Id", "") or session_id   # 会话 id 来自响应头
```

**② 会话 id 来自响应头，宿主机只负责存它。** 容器内 python 把会话 id 作为输出的最后一行带回来，宿主机侧提取后写进 state 文件；下一次投喂如果没有显式要求新会话，就直接读这个文件——所谓「续接同一个会话」，落在地上就是这么几行：

```bash
# 容器输出末尾：__HERMES_FEED_SESSION__=<session-id>
SID="$(printf '%s\n' "$OUT" | sed -n 's/^__HERMES_FEED_SESSION__=//p' | tail -1)"
[ -n "$SID" ] && printf '%s' "$SID" > "$STATE_FILE"
# 下一次投递：若没有显式 --new / -s，就直接读 state 文件
```

等价的、更直白的一行版，把素材整段从 stdin 投进去：

```bash
KEY=$(docker exec hermes sh -lc 'grep -m1 "^API_SERVER_KEY=" <数据目录>/.env | cut -d= -f2-')
docker exec hermes sh -lc "curl -s -m 1800 http://<回环地址>:<API端口>/v1/chat/completions \
  -H 'Authorization: Bearer $KEY' -H 'Content-Type: application/json' \
  -H 'X-Hermes-Session-Key: blog' --data-binary @- " < materials.json
```

**`hermes-blog.sh`——博客交接专用。** 在通用管道外面包了一层「意图与约束」：

- 素材按固定模板投喂：现象 / 排查时间线 / 根因 / 方案 / 验证 / 坑 / 可迁移结论 / **敏感信息清单**
- prompt 里写死约束：加载博客技能；**只到初稿**；必须跑站点自带的脱敏扫描脚本；素材不足要列出缺什么，**不许编造事实**；内网 IP、真实端口、绝对路径、凭据一律脱敏
- `--send` 从收件目录取最旧的一份素材，成功即归档；`--confirm` 才进发布流程（构建 + 扫描 + push + 线上复核）；`--notify` 可选把「初稿已就绪」通知到 QQ
- 博客任务用独立的会话与记忆域，和日常聊天彻底分开

分工上也正好契合站内既有规定：**agent 负责内容，模板 / 配置类改动交给编码 agent**。交接过来的任务，天然落在「写内容」这一侧。

---

## 四、验证

- **通道自检**：直接问它三句话——技能是否已加载、站点根目录在哪、发布前必须跑的扫描脚本是什么。原样回答（路径按站内规则改成占位符）：

```
1) 没有——本次会话开始时 hugo-blog 未加载，我刚用只读方式读了它的 SKILL.md（没写文件、没构建）。
2) 站点根目录 <站点根目录>/（hugo 二进制在 <hugo 路径>，不在 PATH）。
3) 发布前必须跑 content/posts/*.md 的敏感信息扫描 scripts/scan_sensitive.py（技能自带，
   exit 0 = 干净），扫内网 IP/端口/真实路径/密码密钥。
```

  三句都答对了，但真正值得看的是**第一句**：技能并没有自动加载，是它自己去读了技能文件。问题于是从「技能在不在」变成「任务里有没有写兜底」——这也是交接 prompt 必须显式写「技能没自动加载就直接读技能文件」的原因：流程兜底和素材一样，都是交接内容的一部分。
- **续接**：同一会话两轮追问，上下文正确（见 2.5）
- **边界**：空素材、空收件目录都干净退出，不会静默发出一条空消息
- **备选通道**：另一个「建卡派活」的语法可用（先用阻塞状态建卡再归档，**未真实派发**），适合「丢完就走」的异步执行

---

## 五、坑与注意点

1. **「端口映射了」和「端口能用」是两件事。** 回环绑定时前者只是占位。想从宿主机或局域网访问，正确做法是挂到已有的反向代理上做带认证的入口，而不是把监听改成全网卡。
2. **容器 `.env` 里的 key ≠ 容器环境变量。** 任何「进容器跑命令」的脚本，取值方式必须和应用一致。
3. **长文本不要塞进命令行参数。** 引号、`$()`、反引号都会被解释；走 stdin 或文件。
4. **交接要给约束，不只是素材。** 不写「初稿不发布」，agent 可能一路推到线上；不写「跑脱敏扫描」，内网 IP 和绝对路径就可能进公开页面。
5. **确认点必须显式分离：生成 ≠ 发布。**

---

## 六、可迁移的结论

1. 两个 agent 交接，最省事也最安全的通道，往往不是新开一个端口，而是复用对方已有的、本身就受保护的内部接口——哪怕要绕一层 `docker exec`。
2. 常驻 agent 的价值在**技能库**。交接时只需要沟通「意图和素材」，不需要沟通「流程」——流程已经在它脑子里了。
3. 给 agent 写交接 prompt 时，「不许编造、缺信息就提问、到某一步停下等确认」这三条纪律，比任何格式要求都重要。
