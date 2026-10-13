---
title: "Oracle RAC 集群关机加盘并添加 ASM 磁盘完整实录"
date: 2026-10-13T13:00:00+08:00
draft: false
description: "停机加物理磁盘，开机后先起库恢复业务，再在线把新盘加入 ASM 磁盘组并扩容表空间：crsctl 停启集群、UDEV 裸设备绑定、ALTER DISKGROUP ADD DISK 自动重平衡、表空间三种扩法"
tags: ["oracle", "rac", "asm", "udev", "表空间", "运维", "存储扩容"]
categories: ["技术日志"]
---

**环境**：Oracle RAC + Grid Infrastructure + ASM，Linux 裸设备，不使用分区。

**目标**：停机加物理磁盘，开机后将新盘加入 ASM 磁盘组，并扩容表空间。

**顺序原则**：开机后先启动数据库恢复业务，再从从容容地做加盘——ASM 的 `ADD DISK` 是在线操作，此时新盘还没进磁盘组，不影响现有磁盘组和数据库运行。

## 一、停机与加盘

### 1. 停止数据库和集群服务

在任意节点以 oracle 用户停止数据库（`-o immediate` 立即中断并回滚未提交事务，避免正常关闭等业务排空太久）：

```bash
srvctl stop database -d <db_unique_name> -o immediate
```

确认数据库已停止：

```bash
srvctl status database -d <db_unique_name>
```

然后以 root 用户停止集群服务（所有节点都要执行，或按顺序）：

```bash
crsctl stop crs
```

确认集群已停止：

```bash
crsctl check crs
```

最后关闭操作系统，进行物理加盘。

### 2. 硬件加盘

将新磁盘映射给所有 RAC 节点，并记录存储工程师提供的识别号（WWID / SCSI ID）。

开机后，确认操作系统已识别新盘：

```bash
lsblk
fdisk -l
```

新盘应显示为裸设备（如 `/dev/sdd`），且没有有效分区表。

## 二、开机启动集群与数据库

以 root 用户启动集群服务：

```bash
crsctl start crs
```

检查集群状态：

```bash
crsctl check crs
```

确认 ASM 实例已启动：

```bash
srvctl status asm
```

如果 ASM 未启动，手动启动：

```bash
srvctl start asm
```

集群就绪后，**立刻启动数据库恢复业务**，不要等加盘做完：

```bash
srvctl start database -d <db_unique_name>
```

检查状态：

```bash
srvctl status database -d <db_unique_name>
```

连接数据库确认已打开：

```sql
sqlplus / as sysdba
SELECT status FROM v$instance;
```

到这里业务已经恢复。新盘尚未加入任何磁盘组，`DATA` 仍是原来的容量和成员，后续加盘全程在线、不需要再停库。

## 三、配置 UDEV 绑定 ASM 磁盘

RAC 环境下，所有节点都必须配置相同的 UDEV 规则。

### 1. 获取新盘的 SCSI ID

假设新盘为 `/dev/sdd`，执行：

```bash
/usr/lib/udev/scsi_id -g -u -d /dev/sdd
```

输出应与存储提供的识别号一致。

如果系统路径不同，可用 `which scsi_id` 查找。

### 2. 编写 UDEV 规则

编辑 `/etc/udev/rules.d/99-oracle-asmdevices.rules`，添加类似内容：

```bash
KERNEL=="sd*", ENV{DEVTYPE}=="disk", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id -g -u -d %N", RESULT=="你的识别号", SYMLINK+="asm-diskX", OWNER="grid", GROUP="asmadmin", MODE="0660"
```

说明：

- `ENV{DEVTYPE}=="disk"` 表示匹配整块裸盘，不分区。
- `PROGRAM` 通过 SCSI ID 匹配，不依赖设备名，适合 RAC 多节点。
- `SYMLINK+="asm-diskX"` 创建固定别名，如 `asm-data01`。
- `OWNER`、`GROUP`、`MODE` 必须让 grid 用户和 asmadmin 组可读写。

### 3. 重载规则并触发

在所有节点执行：

```bash
udevadm control --reload-rules
udevadm trigger --action=change --sysname-match=sdd
udevadm settle --timeout=30
```

验证权限和别名：

```bash
ls -l /dev/asm-diskX
```

正常应显示：

```text
brw-rw---- 1 grid asmadmin ... /dev/asm-diskX
```

`udevadm settle` 用于等待 udev 事件处理完成，确保符号链接和权限已生效。如果已经看到设备且权限正确，可跳过。

## 四、将磁盘加入 ASM 磁盘组

以 grid 用户连接 ASM 实例：

```bash
su - grid
export ORACLE_HOME=/u01/app/19.0.0/grid   # 替换为实际 Grid Home
export ORACLE_SID=+ASM1                    # RAC 节点 1，节点 2 为 +ASM2
export PATH=$ORACLE_HOME/bin:$PATH
sqlplus / as sysasm
```

SYSASM 是权限，不是数据库账户。以 grid 用户通过操作系统认证即可。

检查新盘是否被识别为候选盘：

```sql
SELECT name, path, header_status, mode_status FROM v$asm_disk;
```

`HEADER_STATUS` 应为 `CANDIDATE`。

加入磁盘组（假设磁盘组名为 DATA）：

```sql
ALTER DISKGROUP DATA ADD DISK '/dev/asm-diskX';
```

ASM 会自动启动重新平衡，无需额外执行 REBALANCE 语句。**这一步在数据库运行中执行即可**，重平衡按 extent 搬移数据，期间读写不受阻塞，只是有一定 I/O 开销。

可选指定速度：

```sql
ALTER DISKGROUP DATA ADD DISK '/dev/asm-diskX' REBALANCE POWER 5;
```

监控进度：

```sql
SELECT * FROM v$asm_operation;
```

确认磁盘组容量已增加：

```sql
SELECT name, total_mb, free_mb FROM v$asm_diskgroup;
```

## 五、增加表空间

### 1. 新建表空间

```sql
CREATE TABLESPACE app_data
DATAFILE '+DATA' SIZE 10G
AUTOEXTEND ON NEXT 1G MAXSIZE 32G;
```

### 2. 扩展现有表空间

追加数据文件：

```sql
ALTER TABLESPACE app_data
ADD DATAFILE '+DATA' SIZE 10G
AUTOEXTEND ON NEXT 1G MAXSIZE 32G;
```

调整现有数据文件自动扩展：

```sql
ALTER DATABASE DATAFILE '+DATA/ORCL/DATAFILE/app_data01.dbf'
AUTOEXTEND ON NEXT 1G MAXSIZE 32G;
```

直接扩大数据文件：

```sql
ALTER DATABASE DATAFILE '+DATA/ORCL/DATAFILE/app_data01.dbf' RESIZE 20G;
```

初始大小 10G 并非固定标准，可根据业务增长预估调整。关键是启用 AUTOEXTEND 并设置合理的 MAXSIZE。

### 3. 验证

```sql
SELECT tablespace_name, status FROM dba_tablespaces;
SELECT file_name, bytes/1024/1024 MB, autoextensible, maxbytes/1024/1024 MAX_MB
FROM dba_data_files
WHERE tablespace_name = 'APP_DATA';
```

## 六、注意事项与踩坑总结

1. **RAC 所有节点都要配置 UDEV**，规则文件必须一致，否则 ASM 可能无法识别或权限报错。
2. **新盘不要分区**，直接使用整块裸设备，配合 UDEV 的 `ENV{DEVTYPE}=="disk"` 即可。
3. **SYSASM 不是账户**，而是通过 grid 用户和 asmadmin 组获得的系统权限。
4. **`ALTER DISKGROUP ... ADD DISK` 会自动重平衡**，无需手动执行 REBALANCE。
5. **加盘和重平衡建议在业务低峰期进行**，减少 I/O 影响。
6. 如果新盘有旧分区或旧 ASM 头，可用 `wipefs -a /dev/sdd` 清除后再触发 UDEV。
7. **多路径环境需针对聚合后的 `/dev/mapper/mpath*` 或 `dm-*` 设备编写 UDEV 规则**，不能直接用物理盘路径。
8. 操作前备份，尤其是磁盘组和表空间变更，确保有回退方案。

## 七、总结

整套流程可概括为：

```text
停数据库 → 停集群 → 关机加盘 → 开机 → 启集群 → 启动数据库（先恢复业务）
→ 确认新盘 → 配置 UDEV（所有节点）→ 重载规则 → ASM 加盘（在线）→ 自动重平衡
→ 增加表空间 → 验证
```

核心是把「恢复业务」和「存储扩容」拆开：数据库该多早起就多早起，`ADD DISK`、重平衡、建表空间这三步全程在线，慢慢做不影响已经恢复的业务。
