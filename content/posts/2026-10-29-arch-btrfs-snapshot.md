---
title: "Arch Linux 根目录快照：pacman 自动打点，翻车用 Live USB 退回"
date: 2026-10-29T12:00:00+08:00
draft: false
description: "snapper + snap-pac 组合：每次 pacman 事务自动生成 pre/post 一对快照，timeline 每小时兜底，升级搞坏用 Arch Live USB 回滚。附保留策略、清理实录、UKI 内核回滚后模块不匹配的处理。"
tags: ["Arch Linux", "btrfs", "snapper", "快照", "snap-pac", "systemd-boot", "回滚"]
categories: ["技术实践"]
---

Arch 滚动更新最大的心理负担是「这次 `-Syu` 会不会把系统搞坏」。我的解法很保守：**每次 pacman 改动系统前自动打一个 btrfs 快照，坏了就用 Live USB 退回旧快照**。刻意没做的三件事：不做可启动快照菜单、不冻结内核、不用 snapper 自带的 `rollback`——原因后面逐条说。这套配置跑了一段时间，把布局、触发、清理和回滚流程记在这里。

## 一、磁盘布局：只有 `@` 进快照

| 项目 | 值 |
| --- | --- |
| 根分区 | `/dev/sdb2`（btrfs） |
| 根子卷 | `@` → `/` |
| ESP | `/dev/sdb1`（vfat，挂在 `/boot`，约 1 G） |
| 启动方式 | UKI：`/boot/EFI/Linux/arch-linux.efi` |
| 快照存储 | 子卷 `@snapshots` → `/.snapshots` |

`/.snapshots` 的 fstab 条目：

```
UUID=<根分区UUID>  /.snapshots  btrfs  rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=/@snapshots  0 0
```

关键取舍是**把不该回滚的东西拆成独立子卷**：

```
/home                  @home
/var/log               @log
/var/cache/pacman/pkg  @pkg
/models                @models
```

日志、包缓存、模型文件不占快照空间，回滚系统也不该动它们——升级失败时日志还得留着查案。

## 二、配置：不用 `create-config`，手动注册

```
snapper      0.13.2-1
snap-pac     3.0.1-3
btrfs-progs  7.1-1
```

| 文件 | 内容 |
| --- | --- |
| `/etc/snapper/configs/root` | 主配置（640 root:root），`SUBVOLUME="/"`、`FSTYPE="btrfs"` |
| `/etc/conf.d/snapper` | `SNAPPER_CONFIGS="root"`（必须，否则 snapper 找不到配置） |
| `/etc/fstab` | `/.snapshots` 挂载条目（备份：`/etc/fstab.bak-snapper`） |
| `/usr/share/libalpm/hooks/` | snap-pac 的三个 pacman hook |

没用 `snapper create-config`：它会自作主张创建 `.snapshots` 子卷，而这个子卷我按自己的布局单独建了——走的是复制官方模板再注册配置名的路子。

## 三、触发：pacman 事务是主力

snap-pac 的三个 hook，触发条件是 `Install / Upgrade / Remove` **任意包操作**，不只 `-Syu`：

```
05-snap-pac-pre.hook     → PreTransaction 打 pre 快照（AbortOnFail）
zz-snap-pac-post.hook    → PostTransaction 打 post 快照
10-snap-pac-removal.hook → 卸载场景
```

- `pacman -Q`、`-Sc` 这类不改系统的操作不触发；yay / paru 最终调用 pacman，同样触发
- 每次事务一对快照，描述就是执行的命令，如 `pacman -S fish`
- pre 快照失败会中止事务（AbortOnFail），坏的包根本进不来

第二层是时间线定时器兜底：

```
snapper-timeline.timer   OnCalendar=hourly   Persistent=no
```

只在开机状态下每个整点 `:00` 触发，关机/休眠错过的小时**不补**——所以快照时间会随开关机出现空洞（例：10 月 28 日 11:00 到 14:00 各一个，之后关机就没了）。要手动打点：

```bash
sudo snapper -c root create --description "手动: 折腾前"
```

## 四、保留策略与清理

```
TIMELINE_CREATE          = yes
TIMELINE_CLEANUP         = yes
TIMELINE_LIMIT_HOURLY    = 5
TIMELINE_LIMIT_DAILY     = 7
TIMELINE_LIMIT_WEEKLY    = 0
TIMELINE_LIMIT_MONTHLY   = 0
TIMELINE_LIMIT_YEARLY    = 0
NUMBER_CLEANUP           = yes
NUMBER_LIMIT             = 10
NUMBER_LIMIT_IMPORTANT   = 10
EMPTY_PRE_POST_CLEANUP   = yes
ALLOW_GROUPS             = wheel
```

读法：**timeline** 留最近 5 个小时段 + 最近 7 天每天 1 个（取并集，理论上限 12 个）；**number** 留 pacman 的 pre/post 最近 10 对，标 `important=yes` 的单独按 10 个算；`empty-pre-post` 清掉内容无变化的 pre/post 对；**`#1` 基线快照没有 cleanup 算法，永远不会被自动删除**。

清理由 `snapper-cleanup.timer` 负责（开机 10 分钟后跑一次，之后每小时一次），依次执行 number、timeline、empty-pre-post 三类清理。它确实在干活：日志里已经实际删除过 `#2 #3 #5 #8 #9 #11 #20`。

## 五、当前状态：编号快照已到上限

截至 2026-10-29：磁盘 `68G / 223G（31%）`，剩余 154G。未启用 btrfs quota，所以 `snapper list` 的 used space 列为空（不影响功能）。

编号快照正好 10 个，**已到 `NUMBER_LIMIT=10` 上限**：

```
#6/#7    pacman -Syu ... cuda ...（装 CUDA 那次）
#13/#14  pacman -S fish
#15/#16  pacman -S rsync
#17/#18  pacman -S zoxide
#21/#22  pacman -S tesseract ...
```

下次 pacman 操作会生成新的一对，随后清理删掉最旧的 `#6/#7`。想保住 CUDA 之前的快照，把 pre/post 一起标 `important`，它们就改走 `NUMBER_LIMIT_IMPORTANT` 名额、不占普通 10 个：

```bash
sudo snapper -c root modify 6 --userdata important=yes
sudo snapper -c root modify 7 --userdata important=yes
```

timeline 快照另有 9 个（`#4 #10 #19 #23 #24 #25 #26 #27 #28`），在 5+7 并集范围内。

## 六、回滚：Live USB 三步走

平时不用想它，真要用的时候在 Arch Live USB 里跑脚本：

```bash
lsblk                                                          # 确认根分区，如 /dev/sdb2
sudo bash ~/btrfs-snap/restore-from-live.sh /dev/sdb2          # 列出快照
sudo bash ~/btrfs-snap/restore-from-live.sh /dev/sdb2 6        # 回滚到 #6
```

脚本做的事（也是手动等价操作）：以 `subvolid=5` 挂载根分区 → 当前 `@` 改名成 `@broken-<时间戳>`（不删除）→ 把 `@snapshots/6/snapshot` 快照成新 `@` → 确保可写、`sync` 后卸载；快照创建失败会自动把旧 `@` 改回来。手动版：

```bash
sudo mount -o subvolid=5 /dev/sdb2 /mnt
ls /mnt                        # 应看到 @ @home @snapshots ...
sudo mv /mnt/@ /mnt/@broken
sudo btrfs subvolume snapshot /mnt/@snapshots/6/snapshot /mnt/@
sudo btrfs property set /mnt/@ ro false
sync && sudo umount /mnt && sudo reboot
```

进去确认系统正常后，再回来删掉旧根（`@broken-<时间戳>`），别急着删——它是你的第二道保险：

```bash
sudo mount -o subvolid=5 /dev/sdb2 /mnt
sudo btrfs subvolume delete /mnt/@broken-XXXXXXXX-XXXXXX
sudo umount /mnt
```

## 七、UKI 内核的坑：快照里没有内核

内核和 initramfs 打包在 ESP 的 UKI 里，**不在根快照内**；根快照里只有 `/usr/lib/modules`。所以回滚后若 ESP 上的内核比快照里的模块新：

- 能正常启动 → 进系统后 `sudo pacman -Syu` 对齐两者
- 启动报模块版本不匹配 → Live USB 里 chroot 重建：

```bash
sudo mount -o subvol=@ /dev/sdb2 /mnt     # 根子卷直接挂到 /mnt
sudo mount /dev/sdb1 /mnt/<boot目录>      # ESP 挂进 chroot 的 boot 目录
sudo arch-chroot /mnt mkinitcpio -P
```

因此内核升级后建议尽快重启一次，让运行内核、`/usr/lib/modules`、ESP 上的 UKI 三者保持一致，别让「回滚」这件事顺带处理三个版本的内核。

## 八、几个别踩的坑

1. **不要用 `snapper rollback`**：内核 cmdline 和 fstab 写死了 `rootflags=subvol=@`，它依赖的「切换默认子卷」机制在这里不适用。
2. **快照只覆盖 `@`**：`/opt`（CUDA 约 5.3G）在快照内，大文件建议放 `/models` 子卷——模型不该被每次回滚带着走。
3. **回滚不等于回滚 pacman 数据库**：snap-pac 的 pre 快照在 pacman 同步数据库**之后**才拍，所以 `-Syu` 翻车退回后，本地库已是新状态——此时再装新包就是不受支持的 partial upgrade，要先再同步一次。
4. 想临时关时间线（保留 pacman 快照）：`sudo systemctl disable --now snapper-timeline.timer`；立刻按策略清理：`sudo snapper -c root cleanup number`（timeline 同理）。

## 九、取舍：谁该这么干

- **该上这套**：单盘 btrfs 的滚动发行版（Arch 及其衍生）、每次 `pacman -Syu` 都心里打鼓、愿意花半小时把布局一次搭对的——换来的是「任意一次包事务都可退」。
- **没必要**：不折腾系统、发行版自带健全快照方案（如 openSUSE 的 snapper 全家桶）、或盘不是 btrfs 的——先解决文件系统，再谈快照。
- **要加码**：多机器/要异地容错的，快照只是本地时间机器，重要数据仍要单独备份链路（[另一篇 ZFS 冷备份](/posts/2026-08-09-cold-backup-debian-zfs-raidz2/)就是另一条腿）。

一句话：这套方案的收益不在省事，在于**把「升级」从不可逆动作降级成可逆动作**——代价只是每次事务一对快照和 10 个名额的清理规则。
