---
title: "1.8G 可用内存的 NAS 上跑 Hindsight：云端向量方案与备份接入"
date: 2026-10-09T18:00:00+08:00
draft: false
description: "把 LLM 抽取、嵌入、重排全部外包给云 API，本地只留服务骨架和嵌入式 PostgreSQL，再把数据搬进既有备份链路并用空目录演练恢复"
tags: ["Docker", "PostgreSQL", "Hindsight", "备份", "内存优化"]
categories: ["踩坑记录"]
---

想给两个常驻的 AI agent（一个跑在宿主机上的编码 agent，一个跑在容器里的 Hermes）配一套长期记忆服务，卡在两条硬约束上：机器是双核 3855U、总内存 3.8G、可用只剩 1.5G 左右，任何要跑本地 PyTorch 或嵌入/重排模型的方案直接出局；同时这台机器没有桌面环境，数据必须落进现有备份链路，挂了要能恢复。

选型是 Hindsight——它支持把 LLM 抽取、嵌入、重排全部指向远端 API，本地只留服务本体和一个嵌入式 PostgreSQL（pg0）。整件事的思路就一句话：把重活外包给云 API，是低配机器跑带向量检索服务的通用解法。

## 一、先验 API，再动容器

所有模型名都是先 curl 一遍确认存在的，不照文档猜：LLM 走小米 mimo 的 `mimo-v2.6-flash`，嵌入走阿里云百炼 `qwen3.7-text-embedding`（实测 1024 维），重排走 `qwen3.7-text-rerank`。

重排这里有个坑：Hindsight 的 alibaba provider 用的是 `/compatible-api/v1/reranks`——是 `compatible-api`，不是 embeddings 那边的 `compatible-mode`，端点写错了返回 404，很容易误判成"模型不可用"。

## 二、内存预算：把连接池收紧

数据库用镜像内嵌的 pg0，把默认连接池从 100 收到 10：

```yaml
HINDSIGHT_API_DATABASE_URL: "pg0://hindsight?shared_buffers=64MB&max_connections=20"
HINDSIGHT_API_DB_POOL_MAX_SIZE: "10"
mem_limit: 800m
```

实测容器记账 451~576MiB / 800MiB，看着像快满了。拆开 cgroup `memory.stat` 才发现真实进程内存只有约 421MB，另外 326MB 是页缓存——内核随时能回收。**判断这类"快爆了"的假象，看 `anon` 才准，`docker stats` 会骗人。**

## 三、三个坑

**坑一：不设 worker id，容器重建后任务永久卡死。** 首次启动日志有条警告：worker id 默认取容器主机名，而容器一重建主机名就变，旧主机名下处于 `processing` 的任务再也没人回收，consolidation 会静默卡住。短期完全看不出问题，等某次重建后才发作。修复是写死一个：

```yaml
HINDSIGHT_API_WORKER_ID: hindsight
```

**坑二：数据落在 docker 命名卷里，备份根本够不着。** 命名卷在 docker 的数据根目录（`<docker数据根目录>/volumes/`）下，普通用户读不了，也进不了现有备份脚本的管辖范围。迁移到宿主机目录的关键前提是：这个镜像以非 root 的 UID 1000 运行，宿主机用户恰好也是 1000，属主天然对得上——换台机器就得先把目录属主改成容器内用户（`chown -R` 一下 UID:GID），否则报的错是含糊的 `Permission denied`。

不需要 root 的迁移手法是拿镜像自己当拷盘工具，`cp` 加 `diff` 一遍过：

```bash
docker run --rm --entrypoint sh \
  -v hindsight-data:/from:ro -v <数据目录>:/to <hindsight镜像> \
  -c 'cp -a /from/. /to/ && diff -r -q /from /to && echo VERIFY_OK'
```

**坑三：手工起 pg0 的 PostgreSQL 缺 ICU 库。** 为了验证热拷贝出来的目录能不能恢复，手工 `pg_ctl` 起库直接报 `libicuuc.so.70: cannot open shared object file`。库其实就在 pg0 自己的安装目录里，只是不在搜索路径，加 `LD_LIBRARY_PATH` 指过去即可。另外拷出来的数据目录带着 `postmaster.pid`，要先删掉否则锁冲突。顺带一个冷知识：`pg_dump` 是客户端工具，不链接 ICU，不需要这个环境变量——只有服务端要。

## 四、备份：热拷贝能救急，但主路径是逻辑备份

新数据目录落在既有 rsync 备份的管辖范围内，默认会被原样拷走——但那是一个正在写入的 PostgreSQL 目录。实测热着 rsync 一份、删掉 pid、用镜像自带的 PG 起库：**能起来，数据完整**。不过这是在近乎空载的库上，写入压力大时 WAL 轮转撞上撕裂版本的几率会上升，所以它只当兜底。

主路径用官方自带的备份命令，不停库：

```bash
docker exec hindsight hindsight-admin backup /tmp/hs-backup.zip
docker cp hindsight:/tmp/hs-backup.zip ./
```

实测 6.7 秒、52KB、22 行数据跨 23 张表。脚本里必须用"先写 `.tmp` 再改名"的写法，否则备份命令失败时会用半截文件覆盖上一次的好备份——这个失败路径我是真跑过的：停掉容器再执行，旧 zip 的 sha256 分毫未变。

## 五、空目录演练，才是备份的验收

"备份命令返回 0"不等于"能恢复"。用**空目录**起一个全新实例再灌回去：健康检查 200（约 40 秒，期间自动拉 74MB 的 PG 二进制）→ 恢复 22 行 23 表 → bank 在、事实条数对、recall 命中、向量完好。

演练还顺带优化了备份体积：pg0 的 PG 二进制不在镜像里，是首次启动按需拉的，所以备份排除掉 `installation` 目录，真正的数据只有实例目录（约 65MB）加那份 zip。

## 六、根因对照

| 问题 | 根因 |
|---|---|
| 异步任务永久卡死 | worker id 默认取容器主机名，重建即变，旧任务无人回收 |
| 数据没法备份 | 默认落在 docker 命名卷，不在宿主机文件系统里 |
| 手工起 PG 失败 | pg0 的服务端依赖自带 ICU 库，需手动指定搜索路径 |
| 恢复流程没底 | 备份"在跑"不等于"能恢复"，必须空目录演练 |
| 控制台用不了 | 默认只绑回环，headless 机器上等于没开 |

最后一行是反着的坑：控制台锁着回环，API 端口却绑在 `0.0.0.0` 对整段内网开放，而我当时（错以为）这个服务没有任何鉴权开关，于是端口怎么绑就成了唯一防线。改法是让控制台监听全网段（浏览器只调它的同源 `/api/*`，由它在服务端转发给 API，所以反代只需要开控制台一个端口），数据面反而可以收得更紧。

> **订正（2026-10-10）**：上一段「没有任何鉴权开关」是错的——鉴权功能一直存在，只是配置项名字里不含 `auth`（数据面叫 `TENANT_EXTENSION`，控制台叫 `CP_ACCESS_KEY`），默认未启用而已。已在[《「这服务没有鉴权」——一句话错了两天》](/posts/2026-10-10-hindsight-data-plane-auth/)里完整纠正并开启。

## 七、谁该这么干

适合的：内存吃紧（可用 1~2G）但有云 API 额度的机器，本地只留服务骨架 + 嵌入式库，一样能跑带向量检索和自动整合的记忆服务。该留在本地方案的：内存 8G 以上、或者明确不想让记忆内容出门到第三方 API 的场景——本地方案的嵌入维度、模型选择都自己控，代价是那点内存和折腾成本。

最后一条经验最便宜也最值钱：**备份一定要用空目录演练一次**。只验证"命令跑通了"，等于只验证了备份的存在性，没验证恢复能力。
