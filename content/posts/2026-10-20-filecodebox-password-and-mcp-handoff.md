---
title: "忘了 FileCodeBox 后台口令之后：从 SQLite 里把口令捞出来，再把它改造成博客素材的传送带"
date: 2026-10-20T17:40:00+08:00
draft: false
description: "忘了一个没有找回入口的后台口令，从 SQLite 里把它捞回来，顺带查清『保存设置弹回登录页』的真正原因，再把文件快递柜改造成博客素材的跨机器传送带"
tags: ["FileCodeBox", "SQLite", "口令哈希", "JWT", "MCP", "内网服务", "Docker", "博客工作流"]
categories: ["踩坑记录"]
---

内网跑着一个 FileCodeBox（文件快递柜），一直当匿名分享工具用。今天要进它的后台，管理员口令死活想不起来——而这个程序既没有「忘记密码」入口，也没有任何邮件/短信找回路径，口令只存在于它自己的 SQLite 里。

更麻烦的是，登录失败按 IP 计数（默认 `error_count=10` / `error_minute=1`），连续试错会被临时锁定，「一顿乱试」这条路本身也走不通：

```
POST /admin/login  →  401 {"detail":"密码错误"}
```

于是就有了这次从库里把口令捞回来的排查。本来到「字典命中」就该结束了，结果中途又栽进一个更值得写的岔子——找回口令 46 秒后，会话全没了，症状和「口令错」一模一样。最后还顺手把这个快递柜改造成了博客素材的跨机器传送带。

## 一、排查：从环境变量翻到 SQLite

**假设 1：口令应该在环境变量或 compose 里。**

```
$ docker inspect <fcb容器> --format '{{json .Config.Env}}' | python3 -m json.tool
["PATH=...", "LANG=C.UTF-8", "APP_VERSION=2.7.1", "HOST=0.0.0.0", "PORT=12345",
 "WORKERS=1", "APP_ENV=production", "LOG_LEVEL=warning", "ACCESS_LOG=false",
 "FORWARDED_ALLOW_IPS="]
```

只有运行时参数，没有任何凭据。**假设推翻。**

**假设 2：口令在挂载的数据卷里，而且不是不可逆的。**

```
$ docker exec <fcb容器> ls -la <数据卷挂载点>
-rw-r--r-- 1 root root 81920 filecodebox.db
-rw-r--r-- 1 root root 32768 filecodebox.db-shm
-rw-r--r-- 1 root root 12392 filecodebox.db-wal
```

拿到 SQLite。它的配置存在 `keyvalue` 表里：

```sql
CREATE TABLE keyvalue (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  key VARCHAR(255) NOT NULL UNIQUE,
  value JSON,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL
)
```

```sql
SELECT value FROM keyvalue WHERE key='settings';
-- → {... "admin_token": "sha256$<16字节hex盐>$<64位hex哈希>", "jwt_secret": "<随机串>" ...}
```

**关键观察：`admin_token` 是 `sha256$salt$hash` 三段式**——不是 bcrypt/scrypt/argon2，是**单轮无迭代的快哈希**。翻程序的 `/app/core/utils.py`：

```python
PASSWORD_SCHEME = "scrypt"   # 现行方案；sha256/明文只是「仍可验证」的历史格式

def verify_password(password, hashed):
    if hashed.startswith("scrypt$"):
        scheme, n, r, p, salt, stored_hash = hashed.split("$")
        password_hash = hashlib.scrypt(password.encode(), salt=salt.encode(),
                                       n=int(n), r=int(r), p=int(p), maxmem=...).hex()
        return hmac.compare_digest(password_hash, stored_hash)

    # sha256 格式: sha256$salt$hash（历史数据，验证逻辑保持原样）
    if hashed.startswith("sha256$"):
        _, salt, stored_hash = hashed.split("$")
        password_hash = hashlib.sha256(f"{salt}{password}".encode()).hexdigest()
        return hmac.compare_digest(password_hash, stored_hash)
```

盐固定、算法快、没有迭代次数——**这条记录的实际强度约等于明文**，字典里有的口令秒出。

**假设 3（验证）：直接拿字典试。**

```bash
python3 - <<'PY'
import hashlib, hmac
salt, target = "<从 settings 里取的盐>", "<从 settings 里取的哈希>"
for p in (line.rstrip("\n") for line in open("/path/to/wordlist.txt", encoding="utf-8", errors="ignore")):
    if hmac.compare_digest(hashlib.sha256((salt+p).encode()).hexdigest(), target):
        print("HIT:", p); break
PY
→ HIT: <那个自设的 11 位串>
```

顺带一个容易踩的点：程序内置的**出厂默认**口令是 `FileCodeBox2023`（源码常量 `LEGACY_DEFAULT_ADMIN_TOKEN` 由 `FileCodeBox` 和 `2023` 拼接而成），而实际这条**不是**默认值，是当年自己设的 11 位串。所以「试试默认口令」和「拿字典爆破」是两条不同的路。

拿接口确认一次：

```
POST /admin/login {"password":"<命中的串>"}
→ 200，返回 is_admin 的 JWT，expires_in: 2592000   # 30 天
```

到这里，按理说就该收工了。

## 二、岔子：找回口令 46 秒后，会话全没了

排查过程中我做了一次只读的「现状快照」，几分钟后这个快照就对不上了：`settings` 里的 `admin_token` 换成了一个**新的 scrypt 哈希**，`jwt_secret` 也变了。

先查程序自己记的管理活动日志（还是 `keyvalue` 表，`admin_activity_events`）：

```
17:02:55  config.update  fields 含 admin_token     count=1
17:04:21  config.update  fields 不含 admin_token   count=1
```

再把反代的访问日志按 `/admin` 过滤，时间线是这样的：

```
16:56:23  GET   /admin/verify          → 200   # 旧会话还在，能进后台
16:56:26  POST  /admin/logout          → 200
17:02:09  POST  /admin/login           → 200   # 用刚找回的口令重新登录成功
17:02:36  GET   /admin/config/get      → 200
17:02:55  PATCH /admin/config/update   → 200   # ← 这一次提交里带了 admin_token
17:03:04  GET   /admin/config/get      → 401   # 会话突然失效
17:03:08  POST  /admin/login           → 200   # 用新口令又登进来了
17:04:21  PATCH /admin/config/update   → 200
```

来源是内网的一台移动设备——也就是**自己在后台「系统设置」里改了管理员口令**。

## 三、根因：保存设置会顺手轮换 jwt_secret

**一句话**：不是「密码错了」，是保存「系统设置」会顺手轮换 `jwt_secret`，把签发过的所有会话一次性作废；这两种故障的表象完全一样（都是被弹回登录页），所以极易误判成口令错。

看源码 `apps/admin/config_service.py` 的 `update_config`：

```python
admin_pw = update_data.get("admin_token")
admin_password_changed = False
if admin_pw is None or admin_pw == "":
    update_data.pop("admin_token", None)
elif not is_password_hashed(admin_pw):
    update_data["admin_token"] = hash_password(admin_pw)
    admin_password_changed = True
else:
    admin_password_changed = True        # ← 提交的值「已经是哈希」也算「改了口令」

...
if admin_password_changed:
    next_config["jwt_secret"] = generate_jwt_secret()     # ← 会话全部作废
```

两处设计值得记下来：

1. **保存设置 = 换会话密钥**。这是「改了密码就该重新登录」的合理意图，但副作用是：任何一次设置保存都会让当前这个 tab 掉线，症状和「密码错」一模一样。日志里 `17:03:04 → 401` 就是这个，紧接着 `17:03:08` 用新口令又登进去了。
2. **「提交的 token 已经是哈希」也走同一条分支**。也就是说，如果前端把已经哈希过的值原样回传，口令没变也会轮换 `jwt_secret`，白踢一次。

另外，`17:02:55` 那次提交**带** `admin_token` 字段，等于把密码框里的内容哈希后写库——所以现在生效的是那一刻表单里的值。这解释了「明明刚才还能登、现在登不上了」。

## 四、两套解法

### 4.1 恢复原口令（只想找回，不改库）

口令是 `sha256$salt$hash` 快哈希时可以字典恢复，只要能读到数据库：

```bash
# 取盐和目标哈希
docker exec <fcb容器> python3 -c "
import sqlite3,json
c=sqlite3.connect('<数据卷挂载点>/filecodebox.db')
h=json.loads(c.execute(\"select value from keyvalue where key='settings'\").fetchone()[0])['admin_token']
_,salt,target=h.split('\$'); print(salt); print(target)"
```

然后接上文那段字典比对脚本（python3 就够，不需要 hashcat）。

### 4.2 直接重置（口令已经变成 scrypt，爆不动了）

scrypt（`n=16384, r=8, p=1`，约 50ms/次）不适合爆破，只能在库层面重置。**必须先停容器再写库**，改完启动，让它重新加载配置：

```bash
docker stop <fcb容器>
docker run --rm -v <数据卷>:/d python:3.12-alpine python3 - <<'PY'
import sqlite3, json, hashlib, os
new_password = "改成你要的新口令"
salt = os.urandom(16).hex()
h = hashlib.scrypt(new_password.encode(), salt=salt.encode(),
                   n=2**14, r=8, p=1, maxmem=64*1024*1024).hex()
hashed = f"scrypt$16384$8$1${salt}${h}"
db = sqlite3.connect("/d/filecodebox.db")
cfg = json.loads(db.execute("select value from keyvalue where key='settings'").fetchone()[0])
cfg["admin_token"] = hashed
db.execute("update keyvalue set value=? where key='settings'", (json.dumps(cfg),))
db.commit(); print("admin_token 已重置")
PY
docker start <fcb容器>
```

> `scrypt$<n>$<r>$<p>$<salt>$<hash>` 六段，参数要和 `core/utils.py` 里的 `SCRYPT_N/R/P` 一致，否则验证路径会直接判失败。

## 五、顺手改造：把快递柜变成博客素材的传送带

背景：原有的博客素材交接是「Pi 往宿主目录写 md 文件，同机的容器读文件」。**换一台机器就断了**——投递箱是宿主机上的目录。既然 FileCodeBox 本来就在内网跑着、又支持 REST 上传和取件码，就把它当传输层。

**先做事实核查（这一步别省）：**

1. **版本**：本地镜像是 2.7.1，而 MCP 支持是上游 2.8.0 才加的。`docker compose pull` 到 2.8.3 后：

   ```
   /health → {"code":200,"message":"ok","detail":{"status":"ok","version":"2.8.3",
             "storage":"local","theme":"themes/2024"}}
   ```

2. **MCP 的能力边界**（握手 + `tools/list` 实测）：

   ```
   initialize → {"result":{"protocolVersion":"2025-03-26",
                            "serverInfo":{"name":"filecodebox","version":"1.0.0"}}}
   tools/list → share_text, share_file
   ```

   **只有「创建分享」两个工具，没有「凭取件码取件」的工具。** 所以 MCP 只能承担「投」，取件必须走 REST。

3. **REST 端点在 2.7.1 上就已经全都有**（实测）：`POST /share/text/`、`POST /share/file/`、`POST /share/select/`、`GET /share/download`——这条通道根本不需要升级，升级只是为了多一个 MCP 客户端面。

4. **它到底在不在公网**：别靠「域名 + 真 TLS 证书」推断，查公共 DNS 的实际应答：

   ```
   $ curl -s 'https://dns.google/resolve?name=<反代域名>&type=A' | ...
   "data": "<RFC1918 地址>"     # 公网 DNS 记录本身就返回内网地址 → 外网根本路由不到
   ```

5. **消费端能不能访问它**：同样实测，别猜：

   ```
   $ docker exec <消费端容器> curl -s -o /dev/null -w '%{http_code} (%{remote_ip})' https://<反代域名>/
   200 (内网地址)
   ```

   容器虽然在另一个 bridge 网络上、fcb 也没发布宿主端口，但通过反代域名是通的，**不需要改任何 compose**。

**落地形态：fcb 只当传送带，账本留在消费端。**

```
Pi 上传到 fcb ──取件码──▶ 消费端取件后自己写进
                         …/blog-inbox/inbox/<素材名>.md
                                   │
                         之后照旧：写成文章、发布、把素材挪进 done/
```

这样「已发布」这个状态仍然由文件系统的 `done/` 表达，不用为一个易失的分享服务另造一套记账约定。上传用 `share_file` 而不是 `share_text`，因为前者保留原始文件名（素材 id 就是文件名，这个信息得跟着走）。

具体命令：

```bash
# 投：拿到取件码（-F 表单，注意是 /share/file/，末尾有斜杠）
curl -s -X POST https://<反代域名>/share/file/ \
  -F "file=@2026-10-20-xxx.md" -F expire_value=7 -F expire_style=day
# → {"code":200,"message":"ok","detail":{"code":"<取件码>","name":"2026-10-20-xxx.md"}}

# 取：每次先 select 现换一个 key 再下载——直链只有约 17～33 分钟寿命，
# 不能缓存复用（key 按 1000 秒时间窗签名，机制见第六节）
U=$(curl -s -X POST https://<反代域名>/share/select/ \
      -H 'Content-Type: application/json' -d '{"code":"<取件码>"}' \
    | python3 -c 'import sys,json;print(json.load(sys.stdin)["detail"]["download_url"])')
# → /share/download?key=<64位hex>&code=<取件码>
curl -fsS -o /tmp/material.md "https://<反代域名>$U"
```

顺手把它注册成 coding agent 自己的 MCP server（streamable HTTP，免鉴权）：

```bash
pi mcp add filecodebox --url https://<反代域名>/mcp --exposure direct --description "..."
# pi mcp list → filecodebox: connected, 2 tools (direct, global)
```

## 六、验证

**口令恢复**：

```
POST /admin/login {"password":"<命中串>"} → 200
detail: {"id":"admin","username":"admin","token":"eyJ…","token_type":"Bearer",
         "expires_in":2592000}
```

**升级无损**：`/health` 返回 2.8.3；`keyvalue` 三条记录齐全；站点名、主题、上传开关都没变。

**端到端过一遍快递柜通道**（这是关键，单元测试不算数）：

```
# 上传 2203 字节的素材后，在消费端容器里取件
$ docker exec <消费端容器> curl -fsS -o /tmp/material.md '<直链>'
bytes=2203
$ md5sum /tmp/material.md   → d5a1e0bdbc26eef7aaf252037e5f2561
$ md5sum 原件               → d5a1e0bdbc26eef7aaf252037e5f2561     # 逐字节一致

# 权限：容器里写投递箱、再挪进 done/
$ cp /tmp/material.md …/blog-inbox/inbox/2026-10-20-xxx.md   # 写成功
$ mv  …/blog-inbox/inbox/2026-10-20-xxx.md …/blog-inbox/done/   # 挪成功
```

**边界情况试过的**：

- `POST /share/select/` **必须 JSON body**；用 form 编码会 422：

  ```
  HTTP 422 {"detail":[{"type":"model_attributes_type","loc":["body"],
            "msg":"Input should be a valid dictionary or object to extract fields from",
            "input":"code=<取件码>"}]}
  ```

- `GET /share/download` **缺 key 会 422**（`{"type":"missing","loc":["query","key"]}`）——所以不能只凭取件码 wget，得先 select 拿 key。
- **`select` 不消耗下载次数**：`expire_style=count, expire_value=1` 时，select 三次后 `used_count` 仍是 0、`remaining_downloads` 仍是 1；真正消耗的是 `download`。
- **下载直链里的 `key` 是限时的——这正是最初判错、本次订正的地方。** 起初在同一分钟内连调三次 select，拿到的 key 每次都一模一样，于是我得出「key 稳定、直链可缓存」的结论，还写进了交接说明；第二天再拿同一条直链去取，得到 `403 {"code":403,"detail":"下载鉴权失败"}`（旧直链隔天实测仍是 403）。

  翻源码才看到真相（`core/utils.py`）：

  ```python
  async def get_select_token(code: str, offset: int = 0):
      jwt = getattr(settings, "jwt_secret", "")     # 会话密钥
      # 每个窗口约 1000 秒；offset 允许校验上一窗口，避免边界竞态
      time_factor = int(time.time() / 1000) - max(0, int(offset))
      return hashlib.sha256(f"{code}{time_factor}000{jwt}".encode()).hexdigest()
  ```

  而下载入口只接受**当前与上一个窗口**的 token（`apps/base/views.py`）：

  ```python
  valid_keys = [
      await get_select_token(normalized_code, offset=0),
      await get_select_token(normalized_code, offset=1),
  ]
  if not any(hmac.compare_digest(key.encode(), c.encode()) for c in valid_keys):
      ip_limit["error"].add_ip(ip)
      raise HTTPException(status_code=403, detail="下载鉴权失败")
  ```

  拿真实数据把窗口算回来也对得上：上传时刻 17:38:57 落在窗口 1792489，用当时的取件码反算出的 sha256 与旧直链印的 key 完全一致；次日当前窗口 1792569 算出的是另一个值，旧 key 已不在任何有效窗口。

  **结论：一条直链的寿命是 1000～2000 秒（约 17～33 分钟），到期必须重新 select 换 key；取件码本身存在 `filecodes` 表里，长期有效，不受时间窗影响。** 同窗内重复 select 看到同一个 key 属正常——为什么归纳会骗人，见第八节第 5 条。

## 七、坑与注意点

- **最容易误判的一条**：后台「保存设置」成功之后被弹回登录页，**不是口令错了**，是 `jwt_secret` 被轮换了。17:03:04 的 401 和 17:03:08 的登录成功只差 4 秒，光看错误码根本分不出来——排查时把「认证失败」和「配置写入」的时间戳对起来看。
- **「提交已哈希的 token」也会触发轮换**（`else: admin_password_changed = True`），原样保存也可能白踢一次。
- **`sha256$salt$hash` 格式的历史口令等于明文强度**。程序在登录成功时会透明升级成 scrypt（日志：「管理员口令已升级为 scrypt 哈希」），但升级前它一直躺在库里。
- **FileCodeBox 出错也返回 HTTP 200**：`GET /share/download` 遇到过期/不存在的码，返回的是 `200` + body `{"code":404,"message":"ok","detail":"文件已过期"}`。实测第二次下载：

  ```
  1st fetch  http=200  bytes=12   ← 正文
  2nd fetch  http=200  bytes=54   ← {"code":404,...,"detail":"文件已过期"}
  ```

  **所以 `curl -o material.md <url>` 会把这段错误 JSON 原样写进素材文件，`curl -f` 也不会失败。** 取完必须确认拿到的是正文。这是这条通道最阴的坑。
- **403 `{"detail":"下载鉴权失败"}` = key 过期，不是素材丢了**——重新 select 换一条直链就能下。而且 403 会计入 IP 限流（`ip_limit["error"]`，默认 `error_count=10` / `error_minute=1`），拿过期链接反复重试等于自己把自己短暂拉黑，别这么做。
- **`share_text` 没有下载直链**：它的 `download_url` 是 `null`，文本只能走 `POST /share/select/` 取 `detail.text`，wget/curl 拉不了。要可下载、要保留文件名，就必须用 `share_file`。
- **容器里通常没有 wget**：`curl` 有、`wget` 没有，给消费端的取件命令要写 curl。
- **升级前先备份**：fcb 的数据在**命名卷**里，不在 compose 目录，任何「按目录备份」的脚本都覆盖不到它：

  ```bash
  docker run --rm -v <数据卷>:/d -v <备份目录>:/b \
    alpine tar czf /b/fcb-data-$(date +%F-%H%M).tgz -C /d .
  ```

- **`latest` 是浮动标签**：2.8.0 加过一个 HTML 预览端点，2.8.3 又把它删了，小版本之间接口会动，生产用建议 pin 具体版本。
- **通道的适用边界**：因为域名的公网 DNS 就指向内网地址，这条通道**只在同一内网内有效**。消费端/发起端真的在异地（外网笔记本、VPS）时它和任何内网服务一样不通，需要 VPN/组网，不是这个方案能覆盖的。
- **回滚**：走库的话改动只在 `settings` 这一个 JSON 里，恢复旧值即可；走通道的话素材在 fcb 里是过路副本，设了 TTL 会自己过期，不构成持久化负担。

## 八、可迁移的结论

1. **口令哈希的格式就是它的强度**。看到 `sha256$salt$hash`、`md5(...)` 这类单轮快哈希，直接当明文处理——字典几天内就能出；反之 scrypt/argon2 就别想在库层面爆破，只能重置。判断成本的方法不是「有没有盐」，而是「是不是 memory-hard + 有没有迭代」。
2. **「保存设置」触发会话密钥轮换，是把用户锁在门外的经典设计坑**。它和「凭证错误」的表象完全重合——排这类问题时，先建立「写操作」与「认证失败」的时间相关性，再下结论，靠错误码会一直猜错。
3. **别用间接信号推断服务的可达性**。域名带真 TLS 证书 ≠ 在公网（本例公网 DNS 直接返回内网地址）；有 compose 有容器 ≠ 消费端能连上（本例跨 bridge 网络、没发布端口，但经反代是通的）。两个方向都得实测：查公共 DNS 的实际 A 记录，再从目标容器里带 `%{http_code} %{remote_ip}` 真跑一次。
4. **把「传输」和「账本」分开设计**。传输层可以选任何易失的东西（取件码、TTL、一次性），账本（已发布/未发布、修订历史）留在需要长期持有的那一侧的文件系统里。这样换传输方式时不需要重做状态机，也不会出现「两边记账不一致」。
5. **同一时间窗内的重复观测，会骗你相信一个值是恒定的。** 连续三次 select 拿到同一个 `key`，就断定「key 稳定、直链可缓存」，还写进了交接说明；真相是它每 1000 秒换一次，第二天直接 403。教训有两层：凡是看起来像「常量」的令牌/签名/链接，背后一定有窗口或过期参数——**先去源码找那个参数**，而不要靠多次观测归纳；要验证时效性，唯一有效的办法是**跨过时间边界再测一次**，同窗内的样本再多也没有信息量。还有一个附带结论：这类 token 常常拿服务端密钥签名（本例是 `jwt_secret`），于是「改个无关设置」就能把在途链接全作废，排查时要想到这条连带关系。
