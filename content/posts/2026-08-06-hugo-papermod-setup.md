---
title: "Hugo + PaperMod 博客搭建记录(含踩坑)"
date: 2026-08-06T21:30:00+08:00
tags: ["Hugo", "PaperMod", "博客", "搭建"]
categories: ["技术实践"]
draft: false
---

# Hugo + PaperMod 博客搭建记录(含踩坑)

> 本博客即本文的产物。记录完整搭建过程和踩过的坑,供以后重建参考。

## 技术选型

- **Hugo** v0.164.0(extended)— Go 单二进制,极快
- **PaperMod** 主题 — 极简技术风,内置搜索/归档/标签
- 目标:按日期更新的技术日志流

## 搭建步骤

### 1. 安装 Hugo(容器内,免 root)

```bash
# 下载官方 release 的 tar.gz 到用户目录(路径按实际环境替换)
curl -L "https://github.com/gohugoio/hugo/releases/download/v0.164.0/hugo_extended_0.164.0_linux-amd64.tar.gz" -o /tmp/hugo.tar.gz
mkdir -p ~/bin && tar -xzf /tmp/hugo.tar.gz -C ~/bin hugo
# 验证
~/bin/hugo version
```

### 2. 建站点 + 装主题

```bash
cd ~ && ~/bin/hugo new site techlog
git clone --depth 1 https://github.com/adityatelange/hugo-PaperMod ~/techlog/themes/PaperMod
```

### 3. 配置 config.toml(中文)

关键配置点:
- `theme = "PaperMod"`
- 菜单:归档 / 搜索 / 标签
- 输出:HTML + RSS + JSON(搜索需要)

### 4. 创建归档/搜索页面

```markdown
<!-- content/archives.md -->
---
title: "归档"
layout: "archives"
url: "/archives/"
---

<!-- content/search.md -->
---
title: "搜索"
layout: "search"
url: "/search/"
---
```

### 5. 构建

```bash
hugo --minify    # 输出到 public/
```

## 踩坑记录

### 坑 1:hugo.toml vs config.toml 并存

Hugo 0.164 的 `hugo new site` 默认生成 **hugo.toml**,如果又写了 config.toml,**两个文件并存时 hugo 优先读 hugo.toml**(还是默认内容),你的配置全部不生效。

**解决**:删除 hugo.toml,只留 config.toml。

### 坑 2:TOML 语法几个易错点

- `[params.homeInfoParams]` 必须用等号 `Title = "..."`(不是冒号)
- `[menu.main]` 和 `[[menu.main]]` 不能共存——定义表后再当数组表用会报 "Cannot overwrite a value"
- **正确写法**:直接 `[[menu.main]]` 数组表,每项一个

### 坑 3:日期显示英文 "August 6, 2026"

PaperMod 默认英文日期。改 config.toml:
```toml
[params]
  dateFormat = "2006-01-02"    # 数字格式
```

### 坑 4:归档页月份也是英文

主题模板 `layouts/archives.html` 硬编码 `GroupByDate "January"`。

**正确改法(覆盖模板,升级不丢)**:复制主题模板到站点 layouts/ 下再改:
```bash
cp themes/PaperMod/layouts/archives.html layouts/archives.html
# 改 GroupByDate "January" → GroupByDate "1月"(中文月份)
```
站点 layouts/ 优先级高于主题,主题升级不会覆盖。

### 坑 5:归档页想"年份 + 中文月份"两级

默认模板是"年份分组 → 月份分组"两级。月份显示成"8月"用 `GroupByDate "1月"`(Go layout 里 `1` = 数字月份,加"月"字就是中文)。

## 本地预览(容器场景注意)

容器内起 hugo server,容器外访问需要**端口映射**或**复用已映射端口**:

```bash
# baseURL 必须用外部可访问的地址,否则生成的链接是 localhost 打不开
# <port> 用实际端口(如 8642),<宿主机IP> 用实际 IP,勿在文章中写死
hugo server --bind 0.0.0.0 --port <port> --baseURL http://<宿主机IP>:<port>/
```

**坑**:baseURL 写成 localhost → 页面链接全是 localhost,容器外访问跳转失败。必须用宿主机 IP(具体 IP 按实际环境替换,勿在文章中暴露真实内网地址)。

## 结论

Hugo + PaperMod 搭建快(30 分钟内),坑集中在:配置文件冲突、TOML 语法、日期格式、容器端口。记录在案,下次 20 分钟搞定。
