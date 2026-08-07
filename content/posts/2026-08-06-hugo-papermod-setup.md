---
title: "Hugo + PaperMod 博客搭建记录(含踩坑)"
date: 2026-08-07T03:30:00+08:00
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

**坑**:hugo server 的 baseURL 用线上域名时,本地预览的链接会跳转到线上——本地预览必须用本地 baseURL,线上构建用线上 baseURL,两者分开。

## 上线 GitHub Pages(自定义域名)

### 1. 推送流程

```bash
git init && git add -A && git commit -m "init"
git remote add origin git@github.com:<用户>/<仓库>.git
git push -u origin main
```

**坑**:`git clone` 拉的主题(PaperMod)会被当作 embedded git repository(子模块,160000 模式)——删除 `themes/PaperMod/.git` 后重新 add,主题才作为普通文件提交。

### 2. 自动部署(GitHub Actions)

`.github/workflows/deploy.yml`:
- `actions/checkout@v4` + `peaceiris/actions-hugo@v3`(hugo-version 指定)
- `hugo --minify --baseURL "<线上URL>"`
- `actions/upload-pages-artifact@v3` + `actions/deploy-pages@v4`
- 触发:push main

**坑**:首次运行前必须在 GitHub **Settings → Pages → Source 选 GitHub Actions**,否则 deploy 步骤报 Environment 不存在。

### 3. 自定义域名

- DNS 加 CNAME 记录:`<子域>` → `<用户>.github.io.`
- GitHub Pages 设置里填自定义域名 + 勾选 **Enforce HTTPS**(自动签发证书,首次等几分钟)
- **⚠️ 改域名必须同步改 baseURL**(config.toml + Actions workflow 里两处),否则 CSS/JS 链接指向旧域名,页面只有文字没有样式

### 4. Deploy Key(推荐)

SSH key 加到仓库 **Settings → Deploy keys**(不是个人 SSH keys)——只授权单仓库,更安全。**必须勾选 Allow write access**,否则 push 被拒。

**坑**:容器内 hermes 用户的 SSH 家目录是 `/opt/data/.ssh`(passwd 定义),不是 `$HOME`——key 放错位置会 Permission denied。

## 界面定制

### 中文界面

config.toml 加 `defaultContentLanguage = "zh"`(Hugo 0.158+ 废弃 languageCode,用这个才启用中文 i18n):
- 界面文案(Home→主页、阅读时间等)自动中文化
- 自定义翻译加到 `themes/PaperMod/i18n/zh.yaml`(如 `posts → 文章`)

### 面包屑定制(覆盖模板)

站点 `layouts/` 下建同名模板优先级高于主题,主题升级不丢:
- `layouts/_partials/breadcrumbs.html`:自定义面包屑结构
- `layouts/single.html`:控制单页是否显示面包屑(如 About 页排除)

**坑**:用 `.Section` 判断目录 + i18n 键,不要硬编码英文标题字符串(脆弱);判断页面用 `.RelPermalink` 而非 `.Title`。

### 导航菜单

config.toml 的 `[[menu.main]]` 按 weight 排序(小在前),可加外部链接(如 GitHub 仓库)。

## 结论

Hugo + PaperMod 搭建快(30 分钟内),坑集中在:配置文件冲突、TOML 语法、日期格式、容器端口、GitHub Pages 部署、模板定制。记录在案,下次 20 分钟搞定。