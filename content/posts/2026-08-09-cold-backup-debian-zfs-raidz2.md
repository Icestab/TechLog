---
title: "从零搭建一台低频冷备服务器:Debian + ZFS RAIDZ2"
date: 2026-08-09T10:30:00+08:00
tags: ["ZFS", "Debian", "RAIDZ2", "冷备", "存储"]
categories: ["技术实践"]
draft: false
---

## 一、为什么要单独做一台冷备机

对于一些重要数据,单纯依赖主服务器上的 RAID、快照或者定期备份并不够。

我准备搭建一台专门用于冷备的机器:

- 4 块 4TB 硬盘
- 实际冷备数据量不会超过 8TB
- 一个月甚至更久才开机一次
- 完成备份、检查后直接关机
- 平时整机断电
- 主要目标是防止硬盘故障,同时保证数据长期完整性

这台机器并不是传统意义上的 NAS。

它大部分时间都是:

```
关机
 ↓
开机
 ↓
备份
 ↓
检查
 ↓
关机
 ↓
断电
```

因此设计目标不是高性能,也不是提供 7×24 小时文件共享,而是尽可能简单、可靠。

## 二、为什么选择 RAIDZ2

4 块 4TB 硬盘一共有约 16TB 原始容量。

由于实际需要的冷备容量不到 8TB,因此没有必要为了容量去选择 RAID5。

最终选择:

4 × 4TB → ZFS RAIDZ2

RAIDZ2 相当于双校验,可以容忍同时损坏两块硬盘。

大致结构:

```
4 × 4TB HDD
      │
      ▼
   RAIDZ2
      │
      ▼
   ≈ 8TB
   可用容量
```

对于冷备场景来说,容量利用率并不是第一目标。

相比 RAID5,RAIDZ2 可以在一块硬盘已经损坏、正在更换和恢复的情况下,再承受一块硬盘故障,因此更适合这种低频使用、长期存放数据的场景。

## 三、为什么选择 ZFS,而不是 mdadm RAID6

RAIDZ2 和传统 RAID6 在容错能力和可用容量上非常接近。

如果只考虑 RAID 功能,Debian + mdadm RAID6 + ext4 完全可以满足需求。

最终还是选择 ZFS RAIDZ2,主要原因是 ZFS 不只是 RAID。

它还提供:

- 数据校验和 checksum
- scrub
- 数据自愈
- Copy-on-Write
- 快照
- 存储池管理

其中最重要的是数据完整性。

冷备机器一个月甚至更久才开机一次,因此除了防止硬盘直接损坏,还希望在重新开机时能够主动检查长期保存的数据。

ZFS 的 scrub 正好适合这个场景。

## 四、为什么不安装 NAS 系统

虽然 TrueNAS、OpenMediaVault 等 NAS 系统可以降低 ZFS 的管理门槛,但这台机器并不是真正意义上的长期运行 NAS。

它绝大多数时间都是关机状态。

因此没有必要为了一个偶尔使用的冷备机,引入完整的 NAS 管理系统。

最终选择:

Debian Stable + ZFS

这样系统本身非常简单:

```
Debian Stable
    │
    ▼
   ZFS
    │
    ▼
 RAIDZ2
    │
    ├── 4TB
    ├── 4TB
    ├── 4TB
    └── 4TB
```

系统盘与数据盘完全分离。

## 五、为什么不使用 LUKS

最初考虑过使用 LUKS 加密。

但实际需求并不是防止别人获得全部数据,而是:

«如果硬盘因为保修等原因离开自己手里,单独拿到其中一块硬盘时,无法直接读取完整的备份数据.»

RAIDZ2 本身就会把数据和校验分布到多块成员盘上。

因此单独拿走一块 RAIDZ2 成员盘,无法像普通 ext4 硬盘一样直接挂载并看到完整目录和文件。

对于目前的需求,这已经足够。

同时不使用 LUKS,也避免了额外的密钥管理问题。

尤其这是一个低频冷备系统,几年以后重新恢复数据时,最不希望出现的情况就是:

«数据还在,但加密密钥找不到了.»

因此目前最终方案是不使用 LUKS。

需要明确的是:

RAIDZ2 不是加密。

它只能解决单独拿走一块 RAID 成员盘无法直接读取完整文件的问题。

如果未来需求升级为即使拿到完整存储池也不能读取数据,再考虑 ZFS 原生加密即可。

## 六、安装后的初始化

系统安装完成、4 块硬盘接入以后,首先确认磁盘:

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL
```

同时查看稳定的设备路径:

```bash
ls -l /dev/disk/by-id/
```

创建 ZFS RAIDZ2 时,不直接使用 `/dev/sdX`,而使用 `/dev/disk/by-id/`,避免设备名称在重启后发生变化。

例如:

```bash
sudo zpool create \
  -o ashift=12 \
  cold \
  raidz2 \
  /dev/disk/by-id/ata-DISK1 \
  /dev/disk/by-id/ata-DISK2 \
  /dev/disk/by-id/ata-DISK3 \
  /dev/disk/by-id/ata-DISK4
```

这里:

- `zpool create` :创建 ZFS 存储池
- `-o ashift=12` :按照 4K 对齐
- `cold` :存储池名称
- `raidz2` :使用 RAIDZ2
- 后面四个路径:四块实际硬盘

创建完成后,再创建备份数据集:

```bash
sudo zfs create cold/backup
```

## 七、日常维护

这台机器的维护并不复杂。

### 1. 查看存储池状态

最重要的命令:

```bash
sudo zpool status
```

正常情况下应该看到:

```
state: ONLINE
```

如果出现:

```
DEGRADED
FAULTED
UNAVAIL
```

就需要进一步检查。

### 2. 查看存储池容量

```bash
zpool list
```

查看 ZFS 数据集:

```bash
zfs list
```

### 3. 执行 scrub

ZFS scrub 可以对池中的数据进行完整性检查。

执行:

```bash
sudo zpool scrub cold
```

查看进度:

```bash
sudo zpool status
```

如果看到类似:

```
scan: scrub repaired 0B
```

说明本次 scrub 已完成,并且没有发现需要修复的数据。

对于这台冷备机,可以把 scrub 作为定期维护任务,或者根据实际情况在每次冷备周期进行检查。

### 4. 检查硬盘 SMART

安装:

```bash
sudo apt install smartmontools
```

查看硬盘健康状态:

```bash
sudo smartctl -a /dev/sdX
```

重点关注:

- SMART overall-health
- Reallocated_Sector_Ct
- Current_Pending_Sector
- Offline_Uncorrectable

ZFS RAID 状态正常,并不意味着硬盘本身没有潜在问题,因此 SMART 检查同样重要。

## 八、硬盘损坏后的处理

首先:

```bash
sudo zpool status
```

确认具体是哪一块硬盘发生故障。

更换物理硬盘后,查看新硬盘的稳定设备路径:

```bash
ls -l /dev/disk/by-id/
```

然后执行:

```bash
sudo zpool replace cold 旧盘ID 新盘ID
```

例如:

```bash
sudo zpool replace cold \
  ata-old-disk \
  ata-new-disk
```

之后:

```bash
sudo zpool status
```

观察 resilver 进度。

完成以后,存储池应该重新恢复到:

```
state: ONLINE
```

## 九、如果系统盘坏了怎么办

这是采用独立系统盘的一个重要优势。

假设:

```
Debian 系统盘
    ↓
损坏
```

而 4 块数据盘完全正常。

重新安装 Debian,然后安装 ZFS 工具。

插回原来的 4 块硬盘后:

```bash
sudo zpool import
```

即可发现原来的存储池。

然后:

```bash
sudo zpool import cold
```

检查:

```bash
sudo zpool status
sudo zfs list
```

即可重新访问原来的数据。

甚至不一定需要原来的主机。

如果原主机主板损坏,也可以把 4 块硬盘转移到另一台 Linux 机器,在安装 ZFS 后重新导入原来的 pool。

这也是把系统盘和数据盘分离的重要原因:

«系统坏了,可以重装;数据池不依赖原来的系统盘.»

## 十、关机和断电

这台机器不是 7×24 小时运行的 NAS。

备份完成以后正常关机:

```bash
sudo shutdown -h now
```

或者:

```bash
sudo poweroff
```

关机后完全断电不会影响正常使用。

下次重新通电后,ZFS 可以重新识别和导入存储池。

因此整个使用周期可以非常简单:

```
通电
 ↓
启动 Debian
 ↓
检查 zpool status
 ↓
执行备份
 ↓
数据校验
 ↓
必要时执行 scrub
 ↓
确认状态正常
 ↓
shutdown
 ↓
断电
```

真正需要避免的是在数据正在写入时直接拔电。

正常关机后再断电则没有问题。

## 十一、最终方案

最终确定的冷备架构:

```
┌──────────────────────────────┐
│          冷备服务器           │
│                              │
│  Debian Stable               │
│       │                      │
│       └── ZFS                │
│            │                 │
│         RAIDZ2               │
│            │                 │
│       4 × 4TB HDD            │
│            │                 │
│         ≈ 8TB                │
│            │                 │
│       cold/backup            │
└──────────────────────────────┘
```

核心设计原则:

- Debian Stable
- 独立系统盘
- 4 × 4TB HDD
- ZFS RAIDZ2
- 不使用 LUKS
- 不安装 NAS 系统
- 使用 `/dev/disk/by-id/`
- 定期进行 SMART 检查
- 定期执行 ZFS scrub
- 备份完成后正常关机
- 平时整机断电

这台机器的定位不是 NAS,而是一个低频使用的数字保险柜。

相比堆叠大量服务,更重要的是保持系统简单,让它在一个月甚至更久不开机之后,下一次通电依然能够可靠地恢复数据。
