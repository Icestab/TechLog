---
title: "zram 部署踩坑：memory-limit 不是你以为的那个 limit"
date: 2026-10-15T11:00:00+08:00
draft: false
description: "4GB 小主机 swap 吃满，想上 zram 分层却一个单元文件都没生成——根因是 memory-limit 是 host-memory-limit 的废弃别名，语义是「机器内存超过这个数就别创建设备」。"
tags: ["zram", "swap", "内存", "小主机", "Debian", "systemd", "lz4", "sysctl", "vm.swappiness", "运维"]
categories: ["踩坑记录"]
---

这台家用 NAS 的配置是 Intel Celeron 3855U（2 核 2 线程 @1.6GHz，Skylake-U）+ 4GB 内存，系统 Debian GNU/Linux 12 (bookworm)、内核 6.1.0-53-amd64，服务全部 Docker 化（11 个容器），swap 是装系统时建的 `/dev/sda5`，约 975MB，和根文件系统同一块盘。起因不是报错，而是「感觉卡」。本文记一次从内存排查到 zram 分层落地的完整过程，以及那个让我卡最久的坑：一行 `memory-limit`。

## 一、现象：三个信号同时出现

```
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.7Gi       3.4Gi       105Mi        84Mi       617Mi       349Mi
Swap:          974Mi       974Mi        20Ki
```

1. `available` 只剩 349Mi
2. **swap 974Mi 全部用满，只剩 20Ki** —— 等于完全没有换出缓冲
3. 内存压力计数器已经起来了

```
$ cat /proc/pressure/memory
some avg10=0.52 avg60=0.43 avg300=0.73 total=244865863
full avg10=0.47 avg60=0.35 avg300=0.58 total=204163955
```

`full` 行的意思是「所有任务都在等内存」，avg300=0.58 说明过去 5 分钟里平均有 0.58% 的时间整机完全卡在内存上。

CPU 同时被排除：

```
$ uptime
 09:41:03 up 9 days, 33 min,  1 user,  load average: 0.04, 0.07, 0.08
```

2 核机器 load 0.04，所以问题纯粹是内存 + 换出，跟算力无关。

## 二、排查：先排除泄漏，再找大头

### 2.1 假设有进程在泄漏内存 —— 推翻

按 RSS 排序、隔 30 秒采样两次对比增长，所有大进程的 RSS 都原地不动甚至略降：

```
1300031  hermes        383.6 ->  382.8 MB  (Δ+0.0)
1300009  hermes        338.9 ->  337.6 MB  (Δ+0.0)
1273114  hindsight-api 296.1 -> 295.3 MB   (Δ+0.0)
124762   python3       174.9 -> 174.6 MB   (Δ+0.0)
```

没有增长中的泄漏，纯粹是**总量超了**。

### 2.2 按容器归类：一户占了一半机器

`docker stats --no-stream`，问题最严重那一刻：

```
hermes        1.974GiB / 3.745GiB   52.70%     ← 一家占了半台机器
hindsight     503.1MiB / 800MiB     62.89%
hacs          197.5MiB / 3.745GiB    5.15%
postgres       50.83MiB / 3.745GiB   1.33%
vaultwarden    30.2MiB
portainer      27.75MiB
caddy          22.59MiB
filecodebox    19.45MiB
ddns-go        6.602MiB
speedtest      5.934MiB
dufs             788KiB
```

`hermes` 容器一户 52.7%，其余 10 个容器加起来不到 0.9GiB。

### 2.3 拆开 hermes：4 个「幽灵 TUI 会话」

列出容器内进程并回溯父进程链：

```
380 MB  ... python -c ... hermes gateway run --replace
327 MB  ... python -c ... hermes dashboard --host 0.0.0.0 --port 9119 --no-open
 68 MB  /opt/data/bin/hugo server --gc --bind 0.0.0.0 --port 1313
--- 以下是 4 组配对出现的进程 ---
243 MB  python -m tui_gateway.entry      +  70 MB  node .../ui-tui/dist/entry.js
240 MB  python -m tui_gateway.entry      +  66 MB  node .../ui-tui/dist/entry.js
244 MB  python -m tui_gateway.entry      +  67 MB  node .../ui-tui/dist/entry.js
245 MB  python -m tui_gateway.entry      +  63 MB  node .../ui-tui/dist/entry.js
```

父进程链显示这 4 组的父进程都是 dashboard 那个 python：

```
tui_gateway.entry  ← node entry.js  ← hermes dashboard (port 9119)
```

也就是说，这是**浏览器里打开的 hermes dashboard Chat 标签页**拉起来的托管 TUI，每组约 310MB。dashboard 日志能还原时间线：

```
23:19:16 pty accepted peer=<docker网关IP> mode=gated cred=ticket
23:20:16 tui_gateway.ws: ws closed reason=client_disconnect(code=1006)
23:25:35 pty accepted ...
23:26:31 tui_gateway.ws: ws closed reason=client_disconnect(code=1006)
```

关键发现：每次 `pty accepted` 之后大约 1 分钟浏览器就断连了（`client_disconnect`），**但进程还在**。

### 2.4 为什么断开连接后进程不退出

读容器里的实现，`hermes_cli/web_server_chat.py` 写死：

```python
# /api/pty spawns `hermes --tui` behind a pseudo-terminal and forwards bytes +
# resize escapes to xterm.js.
PTY_REGISTRY = PtySessionRegistry(
    ttl=30 * 60, max_sessions=16, buffer_cap=1 * 1024 * 1024, read_timeout=_PTY_READ_CHUNK_TIMEOUT)
```

而 `hermes_cli/pty_session.py` 开头的注释解释了这个设计：

> A PTY process outlives the WebSocket that created it: a single drain task always reads the PTY into a bounded RingBuffer and forwards to the attached socket when present. Reconnecting with the same opaque token replays the buffer and resumes live.

**PTY 是 keep-alive 的**：浏览器断开后进程不退出，要等 `ttl=30*60`（30 分钟）才被 reaper 回收，目的是「重连能续上」。所以关掉标签页 ≠ 释放内存，得等 30 分钟。

顺带确认，在托管模式下你**没办法手动退出它**：

- `/quit`（别名 `/exit`）被显式禁用，只会提示用 `/new` 开新会话
- 空闲时按 Ctrl+C 也不退出，代码里 `handleIdleHotkeyExit()` 在 dashboard 模式下改成调 `requestDashboardNewSession()`

### 2.5 验证：第二天早上再测

```
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.7Gi       2.3Gi       145Mi       136Mi       1.7Gi       1.4Gi
Swap:          974Mi       897Mi        77Mi

$ ps -eo cmd | grep -c '[t]ui_gateway.entry'
0

$ docker stats --no-stream | grep hermes
hermes   1.019GiB / 3.745GiB
```

4 个 TUI 会话被 30 分钟 TTL 自动回收，hermes 容器从 **1.97GiB 降到 1.02GiB**，`available` 从 **349Mi 回到 1.4Gi**。

**所以这台机器的毛病是「峰值憋死」型，不是稳态就撑不住。**

## 三、顺手挖到的两件事

### 3.1 容器不设内存上限，框架的自保机制等于没有

读 `gateway/agent_cache_pressure.py`：

```python
# Shed well under the limit: once cgroup `memory.high` throttling kicks in (swap full),
# a SIGTERM flush cannot finish inside systemd's stop timeout.
_AUTO_BUDGET_FRACTION = 0.65
```

配置项 `agent.agent_cache.memory_high_mb` 默认是 `auto`，算法是：

```
budget = (cgroup memory.high 或 memory.max，都没有则用总内存) × 0.65
```

- 容器**没设限制**时：`0.65 × 3835MB ≈ 2492MB` —— 意思是「匿名 RSS 涨到 2.5GB 才开始丢 LRU transcript 自救」。可全机才 3.8GB，等到那时候宿主早死了
- 设 `mem_limit: 2048m` 之后：`0.65 × 2048 ≈ 1331MB`，自保机制才真正生效

**给小内存机器上的容器设内存上限，最大收益不是「防止它吃满」，而是「给它装上本来就该有的自保等级」。**

用一次性容器实测了 Docker 在 cgroup v2 下的映射（结论和直觉不同）：

```
$ docker run --rm -d --memory 2048m --memory-reservation 1800m --memory-swap 2048m alpine sleep 60
memory.max        2147483648      ← --memory 落在 memory.max
memory.high       max             ← 没有暴露！
memory.low        1887436800      ← --memory-reservation 落在这里（不是 memory.high）
memory.swap.max   0               ← --memory-swap == --memory 表示禁止额外 swap
```

### 3.2 这颗 CPU 没有 AVX，本地 reranker 走不通

`/proc/cpuinfo` 里指令集只有 `sse4_2 popcnt`，**没有 `avx` / `avx2` / `fma` / `f16c`，连 `bmi2` 都没有**。CPU 标识是 `cpu family 6 / model 78 / stepping 3`（真·Skylake-U 核心），但 Intel 在 Skylake 世代的 Pentium/Celeron（含 3855U）上砍掉了 AVX 系列指令。实测 fp32 GEMM：

```
$ docker exec hindsight /app/api/.venv/bin/python -c "...torch matmul..."
GEMM fp32 512x512:  22.7 ms  -> 11.8 GFLOPS
GEMM fp32 1024x1024: 615.3 ms  ->  3.5 GFLOPS
```

（只有 2MB L2、无 L3，一超出缓存就崩。）

hindsight 的 HF 缓存里其实已经有本地 reranker 的权重（`BAAI/bge-small-en-v1.5` 129M、`cross-encoder/ms-marco-MiniLM-L-6-v2` 88M），但真跑最小的那个（22M 参数）→ **直接被 OOM kill**：

```
$ docker exec hindsight cat /sys/fs/cgroup/memory.events
oom 1
oom_kill 1
$ docker exec hindsight cat /sys/fs/cgroup/memory.peak
838979584          ← 正好是 compose 里给的 800MB 限额
```

而当时宿主 `SwapFree = 6MB`，连换出的余地都没有。**加内存条也救不了这条路**（内存只是门票，CPU 才是门槛），所以继续用云端 rerank API。

## 四、转向 zram：先量这颗弱 CPU 扛不扛

既然目标是「给 swap 加个快的层」，zram 是自然选择。但 3.2 已经证明这颗 CPU 很弱，所以**先测压缩算法的实际吞吐**，而不是想当然。

实测条件：16MB 合成 JSON 数据、128KB 分块模拟 zram 的簇压缩、单核、`nice -n 19`、`ulimit -v 160MB` 限死地址空间。

| 算法 | 压缩 | 解压 | 压缩比 |
|---|---|---|---|
| **lz4** | **315.5 MB/s** | **1127.4 MB/s** | 2.83x |
| zstd -3 | 65.2 MB/s | 153.1 MB/s | 5.19x |

（zstd -1 只留了解压 202.4 MB/s，压缩吞吐当时没落盘，不列进表。lz4 是用 `ctypes` 直接调 `LZ4_compress_default` / `LZ4_decompress_safe` 测的，zstd 用系统自带 CLI。内核里的 lz4 是标量实现，实际可能比这个略慢，心里留个折扣。）

换算成代价：

| 动作 | lz4 | zstd |
|---|---|---|
| 换出 100MB 内存 | 0.32 秒 CPU | 1.5 秒 CPU |
| 换入 100MB 内存（压缩后 35MB） | 0.09 秒 CPU | 0.5~0.7 秒 CPU |

**必须选 lz4。** zstd 的压缩比虽然高（5.19x vs 2.83x），但在这颗无 AVX 的 CPU 上压缩只有 65MB/s，会真的把机器拖慢。

### 4.1 可行性核查，顺带撞见一个必然会踩的坑

```
$ find /lib/modules/$(uname -r) -name "*.ko*" | grep -E "zram|zsmalloc|lz4|lzo|zstd"
  ✓ zram            kernel/drivers/block/zram/zram.ko
  ✓ zsmalloc        kernel/mm/zsmalloc.ko
  ✓ lz4             kernel/crypto/lz4.ko
  ✓ lz4_compress    kernel/lib/lz4/lz4_compress.ko
  ✓ lz4hc           kernel/crypto/lz4hc.ko
  ✓ zstd / zstd_compress
  ✗ lzo            不存在
  ✗ lzo_rle        不存在

$ grep -E "^CONFIG_(ZRAM|ZSMALLOC|ZRAM_DEF_COMP)" /boot/config-$(uname -r)
CONFIG_ZRAM=m
CONFIG_ZSMALLOC=m
CONFIG_ZRAM_DEF_COMP_LZORLE=y
CONFIG_ZRAM_DEF_COMP="lzo-rle"        ← 内核默认算法，但 lzo 模块不在！
```

内核默认 `comp_algorithm` 是 `lzo-rle`，而这台机器的 `/lib/modules` 里没有 lzo/lzo_rle 模块。**不显式指定算法的话，`modprobe zram` 能成功，但写 `disksize` 时会因 crypto API 找不到 lzo-rle 而失败。**

其余核查：没有被 blacklist、没有现成的 zram 配置/服务（fnOS 也没有自己的 swap/zram 服务，无冲突）、`/sbin/{zramctl,swapon,mkswap,modprobe}` 都在（一开始 `which` 报缺是 PATH 不含 `/sbin`，虚惊）、apt 里有现成包 `systemd-zram-generator 1.1.2-2+b2` 与 `zram-tools`、zram 是纯内存块设备不落盘（对只剩 7.6G 的根分区零影响）。

## 五、翻车：一行配置让 zram 凭空消失

第一版配置（**这一版是错的**）：

```ini
[zram0]
zram-size = 1536
memory-limit = 512          # ← 错在这里
compression-algorithm = lz4
swap-priority = 100
```

症状是「什么都没生成」：

```
$ systemctl status systemd-zram-setup@zram0.service --no-pager -l
× systemd-zram-setup@zram0.service - Create swap on /dev/zram0
     Active: failed (Result: exit-code)
    Process: 1426370 ExecStart=/lib/systemd/system-generators/zram-generator --setup-device zram0 (code=exited, status=1/FAILURE)

$ systemctl is-enabled dev-zram0.swap
Failed to get unit file state for dev-zram0.swap: No such file or directory

$ ls /run/systemd/generator/ | grep zram
（空）

$ cat /sys/block/zram0/disksize
0
```

**根因定位的关键手法：非 root 直接干跑 generator，不改系统**：

```bash
$ /lib/systemd/system-generators/zram-generator /tmp/gt1 /tmp/gt1 /tmp/gt1
zram0: system has too much memory (3835MB), limit is 512MB, ignoring.
生成的文件: 0 个
```

一句话结案。对照 man page：

```
OPTIONS
   host-memory-limit=
       Sets the upper limit on the total usable RAM (as defined by MemTotal in
       /proc/meminfo, confer proc(5)) above which the device will not be created.
       This takes a nonnegative number, representing that limit in megabytes, or
       the literal string none, which can be used to override a limit set earlier.
       Defaults to none.

OBSOLETE OPTIONS
   memory-limit=
       Compatibility alias for host-memory-limit.
```

（`man zram-generator.conf`）

## 六、根因：它不是设备级上限，是「整机准入门槛」

**`memory-limit=` 不是「zram 设备能占用的内存上限」，而是 `host-memory-limit=` 的废弃别名，语义是「系统内存超过这个数就别创建 zram 设备」。**

在 3835MB 的机器上写 `memory-limit = 512`，等于告诉 zram-generator「这台机器内存 3835MB > 512MB，忽略」→ 一个单元文件都不生成。

而之前的假设错得很自然：这个参数名**太像**一个设备级的内存上限了，直觉上就该是 `/sys/block/zram0/mem_limit` 的配置入口。实际上 zram-generator **根本没有提供**设置设备级 `mem_limit` 的配置项——完整选项只有 `host-memory-limit` / `zram-size` / `zram-fraction` / `max-zram-size` / `compression-algorithm` / `writeback-device` / `swap-priority` / `mount-point` / `fs-type` / `options`。

连锁反应链条：

```
配置里有 memory-limit = 512
  → generator 判定「机器内存超标」，产出 0 个单元文件
    → /run/systemd/generator/dev-zram0.swap 被删除（daemon-reload 重跑 generator 会清掉旧产物）
      → dev-zram0.swap 单元消失 → systemctl restart 它报 "No such file or directory"
        → systemd-zram-setup@zram0.service 以退出码 1 失败
          → zram 完全没有启用，swap 退回只有 sda5 的状态
```

## 七、解决方案

### 7.1 最终配置（`/etc/systemd/zram-generator.conf`）

```ini
[zram0]
zram-size = 1024
compression-algorithm = lz4
swap-priority = 100
```

三个参数的理由：

- **`zram-size = 1024`**：zram 的 `disksize` 是**懒分配的虚拟上限**（没有页换出时占 0 真实内存），所以它不是「预留」。但**最坏情况真实占用 ≈ disksize**（数据完全不可压缩时）。注意 `disksize` 限的是**未压缩逻辑容量**（ArchWiki 原话：controls the maximum *uncompressed* amount of data it can store, not the maximum compressed size），与压缩比无关，真实内存占用 = 逻辑量 ÷ 压缩比。按实测 2.83x 折算：

  | disksize | 能吸收的原始页（= disksize，与压缩比无关） | 真实内存占用：典型(2.83x) / 最坏(不可压缩) |
  |---|---|---|
  | **1024 MB（最终采用）** | **~1.0 GB** | **~362 MB / 1024 MB** |
  | 1536 MB | ~1.5 GB | ~543 MB / 1536 MB |

  当时 swap 只用了 942MB（逻辑量），1024 的容量绰绰有余，同时把最坏情况的风险砍掉三分之一。

- **`compression-algorithm = lz4`**：见第四节实测，lz4 在无 AVX 的 CPU 上是唯一划算的选择。**同时这也避开了「内核默认 lzo-rle 但 lzo 模块缺失」那个坑。**

- **`swap-priority = 100`**：现有 `/dev/sda5` 的优先级是 `-2`。内核优先使用高优先级 swap，只有 zram 满了才溢出到磁盘。**sda5 保留、成本为 0，是一张免费的安全网。**

### 7.2 应用（顺序有讲究）

```bash
# zram 当前是空的（disksize=0），所以改配置不丢数据 —— 这是改 size 的最佳时机
sudo tee /etc/systemd/zram-generator.conf <<'EOF'
[zram0]
zram-size = 1024
compression-algorithm = lz4
swap-priority = 100
EOF

sudo systemctl daemon-reload
sudo systemctl reset-failed systemd-zram-setup@zram0.service
sudo systemctl start dev-zram0.swap
```

- `daemon-reload` 会重跑 generator，把正确的单元生成到 `/run/systemd/generator/`
- `reset-failed` 清掉之前的 failed 状态（否则 start 可能被拦）
- `start dev-zram0.swap` 会通过 `Requires=` 先把 `systemd-zram-setup@zram0.service` 拉起来（它执行 `zram-generator --setup-device zram0`，负责写 sysfs：算法 → disksize），然后 `swapon`

### 7.3 第二轮：配套的内核 vm 调参（发布后追加）

zram 只把「快的层」建起来；下面这组 sysctl 决定**压力来临时内核怎么用它**。

来源是 ArchWiki «zram» → "Optimizing swap on zram" 的标准四件套（该节注明 "These values are what Pop!_OS uses"）：

```ini
vm.swappiness = 180          # ← ArchWiki 原值
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
vm.page-cluster = 0
```

**本机刻意把 swappiness 从 180 改成 100**（理由见下）。最终 `/etc/sysctl.d/99-zram.conf`：

```ini
vm.swappiness = 100
vm.page-cluster = 0
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
```

逐条机理与依据：

| 参数 | 本机默认 | 设为 | 依据 |
|---|---|---|---|
| `vm.swappiness` | 60 | **100** | 内核文档：定义「swap 与文件系统分页的相对 IO 成本」，0-200。**"At 100, the VM assumes equal IO cost"** |
| `vm.page-cluster` | 3 | **0** | 内核文档：默认 3 = 一次换入 2³=8 页=32KB。**"Zero disables swap readahead"** |
| `vm.watermark_boost_factor` | 15000 | **0** | Ubuntu upstream 补丁 "prevent swap storms by disabling high watermark boosting" |
| `vm.watermark_scale_factor` | 10 | **125** | 拉开 min/low/high 水位间距，让 kswapd 提前、平滑回收 |

**`page-cluster = 0` 为什么对 zram 特别有意义**：zram 没有寻道成本，一次批量读 8 页只会让 CPU 白解压用不到的页（本机 lz4 解压 1127 MB/s，省下的都是净赚）。Fedora 的讨论串说得直白：`page-cluster=3` 是给 HDD/SSD 设计的，套在内存压缩块设备上就是「做多余的读、白烧 CPU 解压」。

**`watermark_boost_factor = 0` 是这四条里对本机价值最高的**，因为背书最硬。Ubuntu 的 SRU 补丁描述：

> High watermark boosting can cause **large swap activity** under certain memory intensive workloads, making the system **very unresponsive (screen does not refresh, keyboard not responding, etc.)**. This large swap activity seems to be prevented disabling high watermark boosting.

Debian 邮件列表里另有一条实测：

> Tuning vm.watermark_boost_factor to 0 (disable) on Debian significantly improves performance on memory-intensive tasks that utilise SWAP space, by stopping preemptive kswapd freeing of memory, and subsequent **page thrashing**.

**——这恰好就是本次事故的症状**（load average 冲到 37、所有服务请求超时约 15 分钟，见第一节与第九节）。Ubuntu 后来把它做成了默认值。

**为什么 swappiness 用 100 而不是 ArchWiki 的 180**：180 是在赌「swap IO 几乎免费」，前提是 **swap 全是 zram**。本机是「zram 1G 快层 + 慢盘 975M 溢出层」的两层结构，而 kernel 的 `swappiness` 是**全局的、分不清 swap 设备**。一旦 zram 满了、溢出到那块低端 SSD，180 会让内核继续积极地把页往慢盘推。100 的含义是「swap 和文件系统的 IO 成本相当」，在「快层有限量 + 慢层兜底」的拓扑下是更稳的假设。

**应用**（我用的是 fish，fish 没有 heredoc，所以是「先落文件再 cp」）：

```fish
sudo cp 99-zram.conf /etc/sysctl.d/99-zram.conf && sudo sysctl --system
```

另一个容易被静默跳过的细节：**systemd-sysctl 会忽略「非 root 属主」或「他人可写」的配置文件**。

```fish
sudo chown root:root /etc/sysctl.d/99-zram.conf; sudo chmod 644 /etc/sysctl.d/99-zram.conf
```

**验证**（`sysctl` 接多个 key 会一起打印）：

```fish
sudo sysctl vm.swappiness vm.page-cluster vm.watermark_boost_factor vm.watermark_scale_factor
vm.swappiness = 100
vm.page-cluster = 0
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
```

**验证方式是「看压力下的表现」，不是「看稳态数字」** —— 这四条只在内存压力出现时才起作用，而应用当时 `/proc/pressure/memory` 三个窗口全是 0.00，所以各项稳态指标纹丝不动（预期内，不代表没生效）。

## 八、验证

### 8.1 分层是否生效

```
$ swapon --show
NAME       TYPE      SIZE     USED PRIO
/dev/sda5  partition  975M  942.4M   -2
/dev/zram0 partition    1G     0B   100
```

zram 优先级 100 > sda5 的 -2，总 swap 从 975M 变成约 2.0G（975M 磁盘 + 1G zram）。

### 8.2 开机自启有没有真的生效

zram-generator 是 **systemd generator**，不是 service，所以 `enable` 是无关动作。证据链：

```
$ systemctl is-enabled dev-zram0.swap
generated              ← 「这是开机生成的，没什么可 enable 的」
$ systemctl is-enabled systemd-zram-setup@zram0.service
static                 ← 无 [Install] 段，只由依赖拉起

$ cat /run/systemd/generator/dev-zram0.swap
# Automatically generated by /usr/lib/systemd/system-generators/zram-generator
[Unit]
Requires=systemd-zram-setup@zram0.service
After=systemd-zram-setup@zram0.service
[Swap]
What=/dev/zram0
Priority=100
Options=discard

$ systemctl list-dependencies swap.target
swap.target
● ├─dev-disk-by\x2duuid-<uuid>.swap      ← sda5
● └─dev-zram0.swap                       ← zram0
```

原理：generator 在**每次开机**的早期阶段无条件执行，读 `/etc/systemd/zram-generator.conf`，往 `/run/systemd/generator/`（tmpfs）写单元文件。重启后这些文件消失、再被重新生成——所以去 `enable` 它们毫无意义，**配置文件本身就是开关**。

### 8.3 副产品：以后改配置先「干跑预检」

以普通用户身份干跑 generator，不改系统、不需要 sudo：

```bash
mkdir -p /tmp/gt && /lib/systemd/system-generators/zram-generator /tmp/gt /tmp/gt /tmp/gt
find /tmp/gt -type f -o -type l
```

- 配置有问题 → 直接打印原因（例如 `zram0: system has too much memory (3835MB), limit is 512MB, ignoring.`）
- 配置正常 →
  ```
  Creating unit file dev-zram0.swap (/dev/zram0 with 1536MB)
  /tmp/gt/dev-zram0.swap
  /tmp/gt/swap.target.wants/dev-zram0.swap
  /tmp/gt/systemd-zram-setup@zram0.service.d/bindings.conf
  ```

**看到 `Creating unit file` 再去 `daemon-reload`。** 后来还用 `ZRAM_GENERATOR_ROOT=<假根>` 在沙箱里验证了「去掉 memory-limit 就能生成」，两边都复现了。

### 8.4 观察真实压缩比

`free` 的 Swap 行是**逻辑用量**（换出页数 × 4K，未压缩），zram 的真实内存代价是压缩后的量，两者差一个压缩比。看真实代价要读 `mm_stat`：

```bash
awk '{printf "逻辑 %.1f MB   实际占RAM %.1f MB   压缩比 %.2fx\n", $1/1048576, $3/1048576, $1/$2}' \
  /sys/block/zram0/mm_stat
```

（字段序：`orig_data_size` / `compr_data_size` / `mem_used_total` / `mem_limit` / `mem_used_max` / …）

当时只有 1 页数据，比值还不可采信：

```
4096   69   4096   0   4096   0   0   0   0
```

顺带一个细节：`mem_used_total = 4096` 而 `compr_data_size = 69` —— zsmalloc 按整页分配，69 字节也占满一页。数据量上来后会摊薄，但意味着**不能拿早期的 `mem_used_total` 去推算压缩比**。

### 8.5 真实运行数据（发布后补充——回收 8.4 的悬念）

早先「只有 1 页数据、压缩比不可采信」的悬念可以回收了。zram 稳定运行后：

```
$ swapon --show
NAME       TYPE      SIZE   USED PRIO
/dev/sda5  partition  975M 845.4M   -2
/dev/zram0 partition    1G 417.6M  100

$ awk '...' /sys/block/zram0/mm_stat
逻辑 407.6 MB  压缩后 136.0 MB  实占RAM 144.5 MB  压缩比 3.00x
```

**三个结论：**

1. **生产环境真实压缩比 `2.98x ~ 3.00x`**，与第四节用合成 JSON 数据实测的 2.83x 很接近、略好。这个数字直接验证了 `zram-size = 1024` 选得对：
   - 3x ⇒ 1024MB 的 zram 能吸收约 **1GB 逻辑页**（= disksize），对应真实内存占用典型 **~341MB**、最坏（不可压缩）≈1024MB
   - 实际只用了 144.5MB 真实内存就装着 407.6MB 逻辑页，还有充裕余量
2. **zram 确实在工作**，不是摆设：`USED 417.6M`、逻辑 407.6MB，都在动。
3. **sda5 从 942MB 掉到 845MB** —— 坑清单第 4 条预测的「自然迁移」应验了：被访问过的页换入内存后，下次换出会走优先级更高的 zram，所以旧页在**不需要任何手动操作**的情况下慢慢往 zram 迁。这比预期更好，因为第 4 条当时预测的迁移触发条件是重启。

### 8.6 仍然待观察

- **验收点**：下次重启后 `swapon --show` 里 sda5 的 USED 应接近 0。如果它又涨到几百 MB，说明 zram 满了，要调大 `zram-size`。
- **vm 调参的效果无法归因**：7.3 节那四条只在内存压力出现时才起作用，要验证只能挂长期采样记录下一次压力事件的表现。但即便变好，也很难归因到具体哪一条 —— 所以本文只把它们记成「有依据的预防性调整」，不声称「已验证有效」。
- 压力事件是瞬时的，事后查不出来。当时考虑过挂一个每 5 分钟采样 pressure / swap 分层 / load / zram 压缩比的 systemd timer，最后没做。

## 九、我在同一天造的第二场事故

为了让 lz4 测试更真实，先写了 200MB 随机文件、又生成 190MB 数据文件，然后在 ctypes 测试里把这份数据复制了多份（src/dst/out + 每个 128KB 分块的缓冲）→ **匿名内存约 600~800MB**。而当时 `MemAvailable` 只有 1.4GB、**`SwapFree` 只有 6MB**：

```
$ uptime
 10:36:46 up 9 days,  1:28,  1 user,  load average: 0.60, 31.64, 37.90
                                                      ↑ 2 核机器，15 分钟均值 37.9
```

内核被迫疯狂回收 + 往廉价 SSD 上换页 → 大量进程卡在 D 状态（不可中断 I/O）→ 所有服务请求超时约 15 分钟。**但没有任何容器被杀**：`docker ps -a` 全部 Up、`restarts=0`、40 分钟内的 docker 事件只有测试容器自身。

重跑时按下面这套规则做，才拿到了可信数据：

- 数据集 ≤16MB
- 跑在 `ulimit -v` 限死地址空间的进程里
- `nice -n 19` + `taskset` 单核
- 宿主上不跑超过 100MB 匿名内存的东西；要跑就丢进 `docker run --memory 128m --memory-swap 128m` 的临时容器
- **动手前先看 `SwapFree`，个位数 MB 就一切内存测试免谈**

## 十、坑清单

1. **主坑：`memory-limit` 是废弃别名**，语义是「整机内存超过它就不创建设备」，不是设备级上限（第六节）。教训：**别从二进制里拼出来的字符串猜配置语义，读系统上现成的 man page。**
2. **内核默认算法 `lzo-rle` 的模块可能不存在**，不显式指定算法就不会生效。这条在别的发行版/内核上也可能撞到。
3. **改了配置后 `daemon-reload` 会删掉旧的生成产物**。新配置产出为空时，`/run/systemd/generator/` 里的旧单元会被清掉 → 原本存在的 swap 单元变成 `No such file or directory`。这不是「配置改动导致服务崩溃」，而是「产物被收回」，别往错误方向排查。
4. **加 zram 不会让已有的磁盘 swap 页迁移过去**。已经躺在 sda5 上的 942MB 会原地不动；想搬只能 `swapoff /dev/sda5`，但那会把 942MB 先全部拉回内存，瞬间吃光余量（当时 `MemAvailable` 1.5GB，风险很高）。**不要手动 swapoff，让它自然冷在那里**——新换出的页自然走 zram，重启后是干净起跑。
5. **`free` 的 Swap 数字会误导你**：只有一个合并池，看不出哪个快哪个慢，也看不出 zram 的真实内存代价（逻辑 vs 压缩后差一个压缩比）。
6. **dashboard 的 Chat 标签页 = 一个 310MB 常驻进程**：PTY keep-alive、断开后 30 分钟才回收，托管模式下 `/quit` 被禁用、空闲 Ctrl+C 只会开新会话，`max_sessions=16` 是硬编码的。要立刻回收只能 `docker exec hermes kill <pid>`（宿主机上这些进程 uid=10000，普通用户没权限直接 kill）。
7. **zram 不落盘、可回滚**：删掉或改名 `/etc/systemd/zram-generator.conf` → `daemon-reload` → `swapoff /dev/zram0` → `systemctl stop systemd-zram-setup@zram0.service`；也可以用内核参数 `systemd.zram=0` 或把 generator 软链到 `/dev/null`。

8. **`sysctl --system` 只负责「应用」，不负责「恢复」**。改完 `/etc/sysctl.d/*.conf` 后 `sysctl --system` 会立刻把值写进内核，但**回滚时删掉文件再跑一次 `sysctl --system` 是没用的**——它只应用「现在存在的配置」，不会把运行时值还原成内核默认。真正回滚要显式写回：

   ```fish
   sudo rm /etc/sysctl.d/99-zram.conf
   echo 60    | sudo tee /proc/sys/vm/swappiness
   echo 3     | sudo tee /proc/sys/vm/page-cluster
   echo 15000 | sudo tee /proc/sys/vm/watermark_boost_factor
   echo 10    | sudo tee /proc/sys/vm/watermark_scale_factor
   ```

   同类陷阱：`sysctl --system` 也**不会报错**告诉你某个文件被跳过了。如果值没生效，先检查属主和权限（见 7.3）。

## 十一、可迁移的结论

1. **小内存机器上，`mem_limit` 是「给应用装自保机制」的开关，不只是围栏。** 很多框架（这里的 hermes 就是）的「内存压力 → 自我卸载」阈值是从 cgroup 限制按比例推导的。容器不设限制 → 阈值按物理内存算 → 在共享机器上永远触发不了 → 自保机制等于没有。**先设限制，再做别的优化。**
2. **选压缩算法要按 CPU 实测，不能按压缩比选。** 在这类「阉割 AVX 的 Skylake 世代 Pentium/Celeron」上，zstd 的压缩比优势（5.19x vs 2.83x）抵不过它的 CPU 代价（65 MB/s vs 315 MB/s）。判断标准应该是「换出速率 × CPU 单核时间」。**顺带：选任何 SIMD 敏感的软件栈前，先查 `/proc/cpuinfo` 里有没有 AVX。**
3. **systemd generator 可以「干跑预检」。** `/lib/systemd/system-generators/<name> <dir> <dir> <dir>` 以普通用户身份就能跑，输出到临时目录、不碰系统。改配置前先跑一遍，等于给配置改动加了个零风险编译期检查——对 zram-generator、systemd-gpt-auto-generator 等所有 generator 都适用。
4. **`free` 的 Swap 行只给总量和逻辑用量。** 要做分层（快/慢 swap）诊断，必须看 `swapon --show` 的 per-device 用量，加上设备自己的统计（zram 看 `mm_stat`）。

5. **给 zram 配的 sysctl 是一组「压力下的行为参数」，不是「稳态性能参数」。** 它们只在内存压力出现时才起作用，所以应用前后稳态数字不变是**正常的**，不能据此判断「没用」。验证它们的唯一方式是记录压力事件本身 —— 而压力事件是瞬时的、事后查不出来，所以如果不预先挂采样，就等于放弃了验证机会。（本文就是没挂采样，所以对效果只能存疑。）

6. **抄 zram 调参时要注意「单层还是多层 swap」。** ArchWiki / Pop!_OS 那套值（`swappiness=180`）的前提是 swap 全是 zram；一旦你的 swap 是「zram + 磁盘」两层，`swappiness` 是全局的、分不清设备，180 会把页积极推向那个可能很慢的磁盘层。改成 100（内核文档的 "equal IO cost"）在多层级更稳。

## 十二、谁该这么干，谁不该

**适合**：内存 4GB 上下、swap 已经长期吃满、CPU 弱到加磁盘 swap 也只是慢磨的小机器（家用 NAS、软路由旁的盒子）。zram 是免费的快层，磁盘 swap 留着当安全网，两边不冲突，回滚只要删一个配置文件。

**不适合/要先想清楚**：内存本身就够用的机器（swap 基本不动，加 zram 纯属画蛇添足）；以及指望 zram「变出内存」的场景——**zram 本质上是一种超发**，它让你在 4GB 里塞进更多东西，代价是「zram 满 + 磁盘 swap 满」时 OOM 来得更直接（在磁盘 swap 上至少还能慢慢磨）。zram 的压缩页躺在 zsmalloc 里，是**不可回收的内核内存**，不能被换出、不能被回收——这就是为什么最坏占用要按 `disksize` 算，而不是按压缩比算。
