---
title: "Hermes Agent Docker 部署踩坑实录:从「能跑」到「为什么我最终放弃容器化」"
date: 2026-08-07T18:00:00+08:00
tags: ["Hermes", "Docker", "部署", "架构"]
categories: ["技术实践"]
draft: false
---


最近折腾了一段时间 Hermes Agent。

最开始我的想法非常简单:

«Hermes 不就是一个 AI Agent 吗?Docker 隔离一下环境,岂不是既干净又方便升级?»

于是我选择了 Docker 部署。

结果一路从"这东西跑起来真简单",折腾到了"我已经把 Docker 部署所有坑都踩完了"。

最后得出的结论反而是:

Hermes 可以很好地运行在 Docker 里,但如果你的目标是让它长期作为一台机器上的自动化 Agent 使用,原生部署反而更加合理。

这篇文章记录一下整个过程。

---

## 一、为什么一开始选择 Docker

我平时本身就大量使用 Docker。

数据库、Home Assistant、各种 Web 服务基本都是容器化部署,因此看到 Hermes 官方提供 Docker 镜像之后,第一反应就是:

```yaml
services:
  hermes:
    image: nousresearch/hermes-agent:latest
    container_name: hermes
    restart: unless-stopped
    command: gateway run

    ports:
      - "9119:9119"    # Dashboard(默认 127.0.0.1,按需暴露)

    volumes:
      - ./data:/opt/data

    environment:
      HERMES_DASHBOARD: "1"
```

然后:

```bash
docker compose up -d
```

看起来非常舒服。

而且 Hermes 官方 Docker 设计本身也很清晰:

- 镜像负责程序和运行环境
- `/opt/data` 保存用户数据
- API Key、配置、Session、Skills、Memory 等持久化到宿主机
- 更新时重新拉取镜像即可

官方文档目前也是这么设计的:容器镜像本身是无状态的,用户数据集中保存到挂载的 `/opt/data` 中,因此升级镜像不会直接丢失 Hermes 数据。

对于普通服务来说,这几乎是完美的容器化方案。

但问题来了:

Hermes 不是普通服务。

---

## 二、第一个坑:容器里的 Hermes,到底是谁在执行命令?

Hermes 最大的特点之一,就是它不是一个单纯的 API 服务。

它会:

- 调用 Shell
- 安装软件
- 创建文件
- 操作 Git
- 使用浏览器
- 执行 Skill
- 调用各种命令行工具
- 根据任务动态改变自己的工作环境

这时候就出现了一个非常关键的问题:

«Hermes 在 Docker 里面执行命令,实际上是在什么环境里执行?»

答案很简单:

它看到的是容器,而不是宿主机。

例如 Hermes 想安装 `apt install xxx`,安装进去的是容器;`pip install xxx`、`npm install xxx`,同样还是容器。

这听起来似乎没什么问题——毕竟容器本来就是用来隔离环境的。

但 Agent 的行为和普通应用不同。

---

## 三、第二个坑:容器重建之后,Hermes 刚刚折腾出来的环境没了

这才是 Docker 部署 Hermes 最让我纠结的地方。

假设 Hermes 今天为了完成任务安装了某个 Python 包、某个 Node.js 工具、某个系统命令、某个 CLI……这些东西如果安装在容器文件系统里面,那么它们属于 **Container Layer**(容器层),而不是 `/opt/data`(挂载卷)。

所以:

- `docker restart hermes` → 通常没问题
- `docker rm -f hermes && docker compose up -d` → 或者升级镜像后重建容器 → **回到干净的镜像环境**

官方文档明确建议升级时拉取新镜像并重新创建容器,同时说明 `/opt/data` 中的数据不会因此丢失。

但这里有一个很容易被忽略的区别:

«数据不会丢,不代表运行环境不会丢。»

这也是 Docker 化 Agent 和 Docker 化普通 Web 服务最大的区别之一。

对于 Nginx:容器重建 → 配置文件还在 → 服务恢复。完全没问题。

但是对于 Agent:容器重建 → 配置还在 → API Key 还在 → Memory 还在 → Session 还在 → **但之前动态安装的软件可能没了**。

这就变得很微妙。

### 补充:部分动态安装其实不丢

需要说明的是,Hermes 有一个 **lazy-packages 机制**:它按需安装的 **Python 依赖会存到 `/opt/data/lazy-packages`**(挂载卷),这部分**不会丢**。

真正会丢的是:

- 系统级安装(`apt install`)
- npm 全局安装(`npm i -g`)
- 编译产物、临时运行环境

所以准确的表述是:动态安装的软件**部分**在挂载卷(不丢),**部分**在容器层(重建即失)。

---

## 四、第三个坑:Hermes 自己就是一个"会改环境的软件"

后来我越来越意识到:

Hermes 和传统意义上的 Docker 服务,其实不是一个类型。

传统 Docker 服务通常是:

```
镜像 → 启动 → 读取配置 → 提供服务
```

而 Hermes 更接近:

```
镜像 → 启动 Agent → 接收任务 → 执行 Shell → 安装工具 → 创建文件
    → 调用 Git → 运行 Skill → 修改环境 → 继续工作
```

也就是说:

**Hermes 的运行环境本身就是 Agent 工作空间的一部分。**

这时候 Docker 的"不可变基础设施"理念,和 Agent 的"动态修改环境"理念开始产生冲突。

---

## 五、第四个坑:Docker 里的权限问题

这是我实际部署时遇到的另一个典型问题。

宿主机 `./data` 挂载到 `/opt/data`,但容器里的 Hermes 并不是简单地以宿主机当前用户运行。

于是就可能出现 `Permission denied`——例如 `/opt/data/config.yaml` 或其他数据目录无法写入。

后来发现官方 Docker 镜像实际上已经针对这个问题做了不少处理:当前官方镜像启动时会通过初始化脚本处理 `/opt/data` 的权限,并且 Hermes 主进程本身使用非 root 的 `hermes` 用户运行。官方 Docker 文档也明确说明 Dashboard 进程是在容器内以非 root 的 `hermes` 用户运行。

因此这里最容易犯的错误就是:

«看到权限问题之后直接 `chmod -R 777`。」

我现在不推荐这么做。

应该首先搞清楚:

```bash
docker exec hermes id
ls -ln ./data
```

看看宿主机 UID/GID 和容器里的运行用户到底是什么关系。

---

## 六、第五个坑:Dashboard 和 Gateway 不是一回事

这个坑我前后也绕了一圈。

一开始看到 Compose:

```yaml
ports:
  - "8642:8642"
  - "9119:9119"
```

很自然会理解成:8642 = Gateway,9119 = Dashboard。

这个理解大方向没错,但真正重要的是:

**8642 并不是"只要运行 Gateway 就一定需要对外开放的 Web 管理端口"。**

当前官方 Docker 文档明确说明:

- **8642**:对应 Hermes 的 OpenAI-compatible API server 和 health endpoint,它只有在启用了 API Server 时才真正需要对外提供。如果只使用 Telegram、Discord 等聊天平台,则可以不暴露这个端口。
- **9119**:是 Dashboard。Dashboard 默认监听 `127.0.0.1:9119`,官方这么设计是为了避免把管理界面直接暴露到网络上。

所以不能简单理解成"8642 = 网页,9119 = 网页",更准确的关系是:

```
Hermes Gateway ── API Server ── 8642
Hermes Dashboard ── HTTP ── 9119
```

---

## 七、第六个坑:Dashboard 根本没必要单独跑一个容器

这个问题我也折腾过。

后来查官方最新 Docker 实现才发现:

Dashboard 本来就是 Gateway 容器里的 side-process。

只需要:

```yaml
environment:
  HERMES_DASHBOARD: "1"
```

然后 `command: gateway run` 启动容器即可。

官方 entrypoint 会在主 Gateway 启动之前启动 Dashboard,并由 **s6-overlay** 进行监督。如果 Dashboard 崩溃,s6 会自动重新拉起它。

所以没必要为了 Dashboard 再搞一个 `hermes` + `hermes-dashboard` 两个容器。

官方目前甚至明确写了:

«Dashboard 单独运行成一个容器是不支持的。»

原因是 Dashboard 的 Gateway 存活检测依赖共享 PID namespace。

这也解释了为什么之前一些网上的旧教程会让人越配越复杂。

---

## 八、第七个坑:不要看到 9119 就直接绑定 0.0.0.0

如果你想从局域网访问 `http://服务器IP:9119`,第一反应可能是设置 `HERMES_DASHBOARD_HOST: 0.0.0.0`。

但官方默认不这么做。

因为 Dashboard 里面涉及:

- API Key
- Hermes 配置
- Session
- Agent 管理

因此默认绑定 `127.0.0.1` 其实是非常合理的。

官方也明确警告,允许 Dashboard 绑定非 localhost 地址会暴露敏感信息,需要自行提供可信网络边界和认证。

如果一定要远程访问,更合理的是:

```
浏览器 → Caddy / Nginx → 127.0.0.1:9119
```

而不是直接 `0.0.0.0:9119` 裸奔。

---

## 九、第八个坑:Docker 不是"完全隔离",但也不是"宿主机环境"

这点特别容易产生误解。

Hermes Docker 容器里虽然可以执行大量 Linux 操作,但:

容器 ≠ 宿主机

例如 Hermes 运行 `uname -a` 看到的是容器对应的环境;`apt install` 修改的是容器;`systemctl` 也不是在操作宿主机 systemd。

所以如果你的目标是:

«让 Hermes 像一个真正的 Linux 用户一样使用这台机器。»

Docker 就会变得比较别扭。

---

## 十、但是 Docker 也不是一无是处

折腾完之后,我反而觉得 Hermes Docker 有非常明确的适用场景:

1. **测试 Hermes**:`docker run` → 测试 → 删掉,干净利落。
2. **临时体验**:也非常适合。
3. **不希望 Hermes 接触宿主机**:Docker 反而是更好的方案。
4. **把 Hermes 当成一个固定服务**(如 Telegram → Hermes → 固定 Skills → 固定环境):Docker 也完全没问题。

官方目前甚至建议至少给 Hermes 1 GB 内存,推荐 2~4 GB;如果启用浏览器自动化,则建议至少 2 GB。

---

## 十一、但我的 Hermes 使用方式恰好相反

我的 Hermes 并不是单纯拿来聊天。

我希望它能够真正成为一台机器上的自动化 Agent:

```
Hermes
 ├── Git
 ├── GitHub
 ├── 博客
 ├── 自动化脚本
 ├── OpenWrt 构建
 ├── 文件处理
 ├── Web Search
 ├── Browser
 ├── 各种 CLI
 └── 自己安装所需依赖
```

这时候我更希望:

«Hermes 看到的 Linux 环境,就是我给它准备的那台 Linux。»

而不是:

«Hermes 看到的是一个随时可能被重新创建的容器。»

于是事情就开始变得很明确。

---

## 十二、我最终决定:Hermes 原生部署

原生部署之后,整个模型简单很多。

假设创建一个专门的 Linux 用户:

```bash
useradd -m hermes
```

然后使用 `hermes` 这个用户运行 Hermes。

这样 Hermes 的权限边界非常清楚:

```
root
 ├── 系统管理
 └── hermes 用户
       ├── Hermes
       ├── Git
       ├── Python
       ├── Node
       ├── CLI
       ├── Skills
       ├── 工作目录
       └── Agent 自动化任务
```

Hermes 执行 `pip install`、`npm install`、`git clone` 安装和修改的就是真正属于 `hermes` 这个用户的环境,而不是某个容器层。

这其实更符合 Agent 的工作方式。

---

## 十三、原生部署还有一个隐藏优势:升级模型更自然

Hermes 官方目前 Linux/macOS/WSL2/Termux 的安装方式就是官方安装脚本:

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

官方安装文档也把这种方式作为标准 CLI 安装路径。

原生安装以后:

```
系统 → hermes 用户 → Hermes(配置/Skills/Git/Python/Node/工作环境)
```

整个结构更像一个普通 Linux 用户工作站。

而 Hermes 本身就是一个 Agent,所以:

让它拥有一个属于自己的 Linux 用户环境,比让它生活在一个不断重建的容器里更自然。

---

## 十四、Docker 和原生部署,我现在这样看

| 项目 | Docker | 原生 |
|------|:------:|:----:|
| 初次部署 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 环境隔离 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 升级镜像 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 数据持久化 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 动态安装软件 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 操作宿主机 | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Agent 长期运行 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 调试方便程度 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 环境可控性 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 适合测试 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 适合我的使用方式 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

所以我最终并不是认为:

«Docker 部署 Hermes 是错的。»

而是:

«Docker 更适合把 Hermes 当成一个应用;原生部署更适合把 Hermes 当成一个 Agent。»

这两种定位不同,答案自然也不同。

---

## 十五、如果现在重新部署,我会怎么选?

如果只是想体验 Hermes:

**Docker,直接上。**

官方镜像已经把 Python、Node.js、Playwright/Chromium、Git、ripgrep、ffmpeg 等常用依赖准备好了,部署成本非常低。

如果希望长期运行,并且让 Hermes 自己安装工具、修改工作环境、操作 Git、写博客、运行脚本、做自动化任务、长期积累 Skills、深度使用 Linux,那么我的选择会是:

```
专用 Linux 主机 → hermes 用户 → Hermes Agent
```

必要的时候再使用 Docker:

```
Hermes → 本机环境 + Docker(数据库/构建环境/临时服务/特定工具)
```

也就是说:

不是 Hermes 在 Docker 里面,而是 Hermes 在 Linux 上,需要什么再调用 Docker。

这反而是我现在觉得最舒服的架构。

---

## 十六、最后的总结

这几天折腾 Docker Hermes,最大的收获并不是学会了怎么写 `docker-compose.yml`,而是搞清楚了一个更重要的问题:

«一个 AI Agent 到底应该被当成"服务",还是被当成"用户"?»

如果只是服务:Docker 非常合适。

如果是一个真正会思考 → 执行命令 → 安装软件 → 修改环境 → 创建文件 → 使用 Git → 调用工具 → 持续工作的 Agent,那么它其实更像一个拥有自己 Linux 环境的自动化用户。

所以经过这一轮折腾,我最终决定:

«Hermes 原生部署,Docker 留给 Hermes 去使用。»

这可能才是最符合我实际使用方式的方案。

而 Docker 部署这一路踩过的坑,也算没有白踩——至少现在以后再看到 `image: xxx`,我不会再条件反射地认为:

«"能 Docker 化,就一定应该 Docker 化"。»
