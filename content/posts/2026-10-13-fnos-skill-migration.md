---
title: "fnOS 接入换代：旧 trim-cli 全部下线，新 skill 真机一次通过"
date: 2026-10-13T22:00:00+08:00
draft: false
description: "把散装的 trim-cli 换成自带六平台二进制、走 OAuth PKCE 的 fnOS skill，顺手清掉 .env 里更早版本留下的变量。"
tags: ["fnOS", "NAS", "Hermes", "自动化"]
categories: ["技术实践"]
---

今晚把家里 NAS（飞牛 fnOS）的接入方式整个换代了：旧的 trim-cli 一套全部下线，换成新 skill 包，真机一次通过。这版是飞牛直接挂在官网上的，以前用的那套是论坛里流传的分享版，更像测试用的东西——再加上飞牛后续只会更新官网这一版，旧版不再跟进，两套并存只会互相打架，不如一次切干净。

## 一、新旧两版差在哪

| 维度 | 旧版 | 新版 skill |
| --- | --- | --- |
| 分发 | 手动放一个裸二进制到数据目录 | skill 自带 6 个平台二进制，包装脚本按 `uname` 现选 |
| 认证 | 账号密码 / 手工 token 那一路 | OAuth 2.0 Authorization Code + PKCE |
| 结构 | 单文件说明 | 任务路由表 + 13 个 entry + 21 篇 reference + manifest |
| 传输 | 混着更早一版的 WebSocket 变量 | 显式 `http/https`，明文要额外加开关 |
| 文件读写 | 得开 SSH 跑命令 | `file rename/mv/cp/mkdir/rm/upload` 一组专用命令 |
| 影视（Media）接口 | —— | 本包刻意不含，包装器 `exit 2` 挡掉 |
| 来源 | 论坛里流传的分享版，更像测试用 | 飞牛官网直接挂出 |
| 后续 | 不再跟进 | 跟着飞牛更新 |

## 二、这个 skill 包是怎么组织的

一共 45 个文件。`SKILL.md` 不写细节，只给一张任务路由表：登录、文件、相册、存储、Docker……各自指向 `entries/` 里的具体入口，入口再引到 `reference/` 的接口合约。查东西先选最具体的入口，表里没有的能力去翻 `manifest.json` 的 entries 列表。

二进制按平台放在 `bin/` 下（linux/darwin/windows × x64/arm64 共 6 个），`scripts/trim-cli` 是个 63 行的 POSIX shell 包装器：解析 `uname` 后 `exec` 对应二进制。它同时干了一件我认为挺聪明的事——把 `media` 参数拦在参数解析之前：

```sh
media)
  echo "Media commands are not available in this skill package." >&2
  exit 2
  ;;
```

也就是说，影视接口不是"文档里说不支持"，而是命令根本跑不起来。这种"用包装器做能力裁剪"的做法，比我以前在说明文档里写一句"别用这个命令"可靠得多。

## 三、认证：不再碰 NAS 账号密码

登录只有一条路：交互式 `login`。CLI 打印授权链接、等你在浏览器里自己登录、自己确认授权、自己把一次性 code 粘回提示符。账号密码和 code 都不经过我，也不进命令行参数。

```bash
trim-cli --host <NAS IP> --port <API端口> --scheme http --allow-insecure-http login --no-open
```

几个我特意记下来的合约：session 里存 access/refresh token 和过期时间；access token 临期自动刷新；接口返回 401 时刷新并**只重试一次**，连续 401 就要求重新登录，不循环；refresh **不能扩权**，所以新增模块时如果 session 缺对应 scope，只能重新走一遍授权。远程明文 HTTP 必须显式 `--allow-insecure-http`，不给默认放行——这个"宁可多敲一个 flag"的设计我认可。

## 四、清理：删三样，清五行 env

删掉的东西：旧 trim-cli skill、旧二进制、以及两个旧的配套 skill（一个 SSH 查状态、一个 paramiko 批量改名）。判断依据是它们引用的路径已经不存在，且新包能力覆盖前者的绝大部分。

更值得说的是 `.env` 里那 5 行残留：

```
FNOS_HOST / FNOS_PORT / FNOS_SCHEME=ws / FNOS_ALLOW_INSECURE_WS=true / FNOS_USER
```

`FNOS_SCHEME=ws` 是更早一版走 WebSocket 接口的接入方式，新版 CLI 只认 `auto/http/https`。删之前做了两头验证——新二进制里搜不到任何 `FNOS_*` 变量名，全代码树也搜不到消费者：

```bash
strings bin/trim-cli-linux-x64 | grep -oE 'TRIM_CLI_[A-Z_]+|FNOS_[A-Z_]+' | sort -u
grep -rI 'FNOS_HOST\|FNOS_SCHEME' /opt/hermes   # 无输出
```

同文件里的 `TRIM_CLI_SESSION_STORAGE` 和 `TRIM_CLI_CONFIG_DIR` 则**保留**：前者决定 session 存哪个后端，删了有换后端导致重新 OAuth 的风险；后者在 `strings` 里有名字，是新二进制真在读的变量。判断一个 env 该不该留，靠的是"谁在读它"，不是"它看起来新不新"。

## 五、真机验证

连接参数固定 `--host <NAS IP> --port <API端口> --scheme http --allow-insecure-http`，几条只读命令全部 `exit 0`：

| 命令 | 返回 |
| --- | --- |
| `system info` | 设备型号正常 |
| `storage pools` | btrfs 阵列 `clean`，SSD 固件版本可见 |
| `file ls /vol1` | 目录列表正常 |
| `monitor cpu` | 负载、核心数、温度 |
| `app list` | 应用中心列表 |
| `+status` | 聚合状态一次拉齐 |

session 是迁移前授权的，`refresh.lock` 显示临期自动刷新也正常工作——换 skill 没有把登录态搞丢，这是整次迁移里最省事的一环。

## 六、谁该跟着换

基本都该换。文件这块的顾虑最容易打消：批量改名、移动、复制、上传在新包里都是一等命令，`file rename` 一个动作走 HTTP 直连，不用开 SSH，也不用 paramiko 那种自己拼脚本的老路——我这次删掉的两个旧 skill，一个查状态、一个靠 SSH 批量改名，删完并没有留下能力空洞。

真正被裁掉的只有影视（Media）接口本身：包装器在参数解析之前就 `exit 2`。所以影视库元数据、刮削那类操作这包不提供；但「把一批文件改成规范命名」走的是 `file rename`，跟 media 无关，照样不需要 SSH。

一句话：文件操作留在这，影视接口别指望这包。
