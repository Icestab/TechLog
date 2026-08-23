---
title: "从零打造 Linux TUN 全局代理客户端:sscli 架构与踩坑记录"
date: 2026-08-22T20:00:00+08:00
tags: ["Go", "Shadowsocks", "TUN", "网络", "代理"]
categories: ["技术实践"]
draft: false
---

> 一个轻量、原生 Linux、无 GUI 的 Shadowsocks 分流工具的诞生过程。从需求分析到架构选型,从核心实现到踩过的每一个坑。

## 为什么要做这个

市面上的代理工具很多,但很少有同时满足以下条件的:

1. **纯命令行、无 GUI**——跑在服务器/WSL 上,没有桌面环境
2. **全局接管**——所有应用流量自动分流,不需要每个程序单独设置代理环境变量
3. **规则分流**——国内直连、被墙的走代理,而不是一刀切全代理或全直连
4. **轻量可控**——不依赖 Clash/mihomo 那套大而全的框架,只要核心功能

Clash 系太重,simple-tun2socks 没有分流,tun2socks 只做协议转换不做路由决策。于是决定自己造一个。

## 架构设计

核心思路: **TUN 网卡 + 用户态 TCP/IP 栈 + 规则引擎 + Shadowsocks 子进程**

```text
Linux / WSL
     │  所有应用流量（无需设置任何代理环境变量）
     ▼
   TUN 网卡 (sscli0)
     │
     ▼
sscli 路由引擎（gvisor 用户态 TCP/IP 栈 + 规则匹配）
     │
     ├──── DIRECT ──→ 直接出网（fwmark 防回环）
     │
     └──── PROXY ───→ 本地 SOCKS5 → sslocal → VPS
```

### 关键选型

| 组件 | 选型 | 理由 |
|---|---|---|
| TUN 设备 | `wireguard/tun` (MIT) | WireGuard 同款,稳定可靠,MIT 许可 |
| 用户态 TCP/IP 栈 | **gvisor netstack** (Apache-2.0) | sing-box/mihomo/tun2socks 同款,200 行就能桥接 TUN |
| SOCKS5 客户端 | `x/net/proxy` | Go 标准库扩展,支持 UDP ASSOCIATE |
| Shadowsocks 加密 | **shadowsocks-rust 的 sslocal** | 不自己实现加密,复用官方预编译二进制,安全可靠 |

为什么用 gvisor netstack 而不是直接处理 IP 包?因为 gvisor 提供了完整的 TCP/IP 协议栈——你只需要处理"这个连接走代理还是直连",TCP 三次握手、拥塞控制、重传这些脏活累活全由 gvisor 在用户态搞定。

### 三种分流模式

| 模式 | 行为 | 适用场景 |
|---|---|---|
| `gfw` (默认) | 只代理 GFW List 命中目标,其余直连 | 日常使用,最省流量 |
| `bypass` | 默认代理,绕过中国域名/IP | 需要大部分走代理的场景 |
| `global` | 除局域网外全代理 | 调试/临时全代理 |

规则优先级:Private/LAN > 用户自定义 > 中国域名 > 中国 IP > GFW List > 默认行为。

## 核心实现细节

### 1. TUN ↔ gvisor 桥接

这是整个项目的基石。IP 包的流向:

```go
// TUN 设备读取原始 IP 包
n, err := tunDevice.Read(buf)

// 写入 gvisor channel endpoint（链路层）
endpoint.InjectIndirect(buf[:n], gvisorEnvelope)

// gvisor 处理 TCP/IP 协议栈
// → 触发 TCPForwarder 或 UDPForwarder 回调
// → 我们的 router 做分流决策
```

看起来简单,但有两个大坑:

**坑 1:写偏移 offset 必须留 10 字节头空间**

```go
// 错误写法——offset=0,内核返回路径全灭,看起来"启动就断网"
tunDevice.Write(rawIPPacket, 0)

// 正确写法——预留 virtioNetHdr (10 字节)
tunDevice.Write(rawIPPacket, virtioNetHdrLen)
```

这个 bug 的症状是:启动后所有连接卡在 SYN-SENT,看起来像断网,但 TUN 接口是 up 的。排查了两天才发现是写偏移问题。

**坑 2:读必须批量化处理 GSO 超级包**

Linux 内核会把多个小包合并成一个 GSO 超级包发给 TUN。读取时可能遇到 `ErrTooManySegments`——这不是错误,而是"只读了部分"的信号,必须 `continue` 而不是退出。

### 2. 防代理死循环

TUN 代理最怕的就是:代理流量又被 TUN 接住 → 无限循环 → 断网。

解决方案是三层防护:

1. **fwmark**:所有直连流量打标记,TUN 只接受没有标记的包
2. **独立路由表**:代理出站走独立的默认路由,不经过 TUN
3. **VPS 主机路由豁免**:服务器域名解析出的**所有** IP 都加主机路由,避免 DNS 轮询回环

```text
# 伪代码:VPS 地址全部豁免
for _, ip := range resolveAll(vpsDomain) {
    addHostRoute(ip, directGateway)  // 不走 TUN
}
```

### 3. DNS 分流

DNS 是分流的命脉——搞不定 DNS,域名分流就是空谈。

```text
应用查询 DNS
     │
     ▼
sscli DNS 劫持（iptables REDIRECT 53 端口）
     │
     ├─ 中国域名 → 国内上游（TCP 传输,防 LAN 伪造应答）
     │
     ├─ GFW 域名 → DoH（RFC 8484,防 ISP 明文窥探）
     │
     └─ 启动引导解析 → DoH → 回退系统 DNS（带告警）
```

一个细节:直连上游强制走 **TCP 传输**,而不是 UDP。为什么?因为局域网里可能有路由器/ISP 劫持 DNS 应答(插入广告),TCP 传输能防这种伪造。

### 4. sslocal 子进程管理

Shadowsocks 加密完全由 `shadowsocks-rust` 的预编译 `sslocal` 二进制承担,sscli 不实现任何加密协议。密码通过 **0600 权限的临时配置文件**传递给子进程,不进命令行(防 `/proc` 泄露)。sslocal 以 **nobody** 降权运行。

### 5. 远程会话不中断

这是运维场景的刚需——你 SSH 到服务器上 `sscli start`,不能把自己的 SSH 断了。

实现方式:iptables `mangle OUTPUT` 对 `conntrack ESTABLISHED/RELATED` 连接打 fwmark 直连——既有 TCP 连接(比如当前 SSH 会话)走原路径,新出站连接照常分流。

## 安全退出与崩溃恢复

信号处理是代理工具最容易出问题的地方。sscli 的策略:

- **SIGINT/SIGTERM/SIGHUP**:全量恢复网络状态(TUN 删除、路由表清理、iptables 规则移除)
- **崩溃后**:手动执行 `sudo sscli stop` 清扫残留(PID 验身,不误杀复用 PID 的进程)
- **幂等性**:stop 可以重复执行,不会误操作
- **不依赖 systemd**:手动管理为唯一方式,适配各种环境

## 测试覆盖

```bash
go test ./internal/... -count=1        # 单元测试
go test -race ./internal/tun/ ./internal/dns/ ./internal/daemon/  # 并发测试
./scripts/e2e-test.sh                  # 端到端测试(需 root)
```

端到端测试覆盖:启动后 TUN/策略路由就位 → 直连不被破坏 → 停止后 TUN/路由/ip rule 完整恢复且网络可用。

## 已知限制

- WSL2 mirrored 网络模式下策略路由可能与宿主共享栈冲突,建议 NAT 模式
- 崩溃(`kill -9`)后的残留清扫需手动执行一次 `sudo sscli stop`
- 不支持 Fake-IP、DoT(架构已预留,第一阶段不做)
- 不支持 IPv6 完整代理(默认阻断防泄漏)

## 开源地址

**GitHub**: [Icestab/shadowsocks-linux-cli](https://github.com/Icestab/shadowsocks-linux-cli)

MIT 许可,欢迎试用和反馈。

---

*如果你也在 Linux/WSL 上需要一个轻量的全局代理分流工具,希望这篇文章对你有帮助。有问题欢迎提 Issue。*
