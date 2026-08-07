---
title: "Hermes Dashboard 修改密码:配置热更新技巧"
date: 2026-08-07T17:00:00+08:00
tags: ["Hermes", "Dashboard", "安全", "配置"]
categories: ["技术实践"]
draft: false
---

> 修改 Hermes Dashboard 的登录密码,不需要重启整个容器——改配置 + kill 进程即可,QQ 会话不断线。

## 背景

Hermes Agent 的 Dashboard(Web 管理界面)默认带 basic auth 登录。**Dashboard 页面本身没有"改密码"功能**,需要改配置文件。但很多人以为要重启容器才能生效——其实有更轻的方式。

## 方法

### 1. 改配置文件

编辑 Hermes 配置(config.yaml),找到 dashboard 的 basic_auth 配置:

```yaml
dashboard:
  basic_auth:
    username: admin
    password_hash: "<scrypt 哈希>"   # 或明文 password: "xxx"
```

**两种密码写法**:
- `password_hash`:scrypt 哈希(推荐,不存明文)
- `password`:明文密码(简单但安全性低)

**生成 password_hash**(交互式输入,避免密码出现在命令行历史里):
```bash
# 容器部署的,先进入容器内再执行:
docker exec -it <容器名> bash
# 然后在容器内:
python3 -c "
from plugins.dashboard_auth.basic import hash_password
import getpass
pw = getpass.getpass('输入新密码: ')
print(hash_password(pw))
"
```
用 `getpass` 交互输入——**密码不会显示在终端、不会进 shell history**。
(如果容器里找不到 `plugins` 模块,说明需要在 Hermes 的虚拟环境里执行:`/opt/hermes/.venv/bin/python3`)

### 2. kill Dashboard 进程(热更新)

**关键**:不需要重启整个容器,只需要让 Dashboard 进程重启——Hermes 用 s6 进程管理器,杀掉后会自动拉起新进程并读取新配置:

```bash
kill <dashboard 进程 PID>
# s6 自动拉起新进程(几秒内)
```

### 3. 验证

- 用新密码登录 Dashboard
- 旧密码失效
- **QQ/Telegram 等其他通道不受影响,不需要重新连接**

## 为什么可以这样

Hermes 容器内用 **s6 监督进程**——Dashboard 是独立服务,被杀后 s6 检测到会立即重启,重启时重新读取配置。所以:

```
改 config.yaml → kill dashboard 进程 → s6 自动拉起(读新配置)→ 生效
```

**不需要**:重启整个容器(会影响所有通道、耗时更长)。

## 注意

- 修改前备份 config.yaml(`cp config.yaml config.yaml.bak`)
- password_hash 用 scrypt 生成,别手写
- 如果 dashboard 有多个实例/负载均衡,需要全部重启

## 相关经验

- Hermes 配置热更新的通用思路:**找 s6 管理的子服务,kill 对应进程即可热更新,不用重启容器**
- 同样的方法可用于其他配置变更(dashboard 主题、端口等)
