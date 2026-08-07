---
title: "搜索体系重建:DeepSeek 服务端搜索 + Firecrawl 提取"
date: 2026-08-06T22:00:00+08:00
tags: ["搜索", "DeepSeek", "Firecrawl", "反爬"]
categories: ["技术实践"]
draft: false
---

# 搜索体系重建:DeepSeek 服务端搜索 + Firecrawl 提取

> 2026-08-05 ~ 08-06 折腾两天,解决"搜索引擎全被反爬"的痛点的完整方案。

## 背景

之前用 curl/浏览器直接抓搜索引擎(Google/Bing/DDG/Startpage/Yandex 等),结果**全被反爬挡住**(CAPTCHA/403/验证页)。需要一个稳定、免反爬、低成本的搜索方案。

## 方案演进

### 阶段 1:发现 DeepSeek 支持服务端 web_search

DeepSeek 官方 **Responses API** 支持 `web_search` 工具——**搜索在 DeepSeek 服务器端执行**,客户端拿到的就是结构化结果,完全绕开反爬。

关键验证:
- `api.deepseek.com/v1/responses` 支持 `tools: [{type: web_search}]`
- 事件流有 `web_search_call.in_progress / searching / completed`
- 返回结果含 title/url/description

### 阶段 2:发现 opencode-go 中转也支持

**意外收获**:opencode-go 的 `/zen/go/v1/responses` 端点**同样支持 web_search 工具**——意味着可以用**已有的 OPENCODE_GO_API_KEY**(和主模型同一账户),不用额外注册 DeepSeek 官方 key。

实测:
```json
{"tools": [{"type": "web_search"}]}
→ output 含 web_search_call(completed),返回真实搜索结果
```

### 阶段 3:Firecrawl 负责提取(分工)

社区共识:**Firecrawl 擅长网页提取(extract),搜索是弱项**(官方也承认"basic search endpoint, not true SERP")。于是分工:

```
web.search_backend:  deepseek   ← 搜索(服务端,免反爬)
web.extract_backend: firecrawl  ← 提取(HTML 转 markdown)
```

## 最终架构

```
┌─────────────────────────────────────────────┐
│ 搜索(web_search) → opencode-go /responses    │
│   → DeepSeek 服务端执行 → 结构化结果           │
├─────────────────────────────────────────────┤
│ 提取(web_extract) → Firecrawl API           │
│   → 网页转 markdown(1 credit/页)             │
├─────────────────────────────────────────────┤
│ 兜底 → cn.bing / 搜狗 / curl 直抓            │
└─────────────────────────────────────────────┘
```

## 关键点

1. **搜索和提取解耦**:搜索走模型服务商(不烧 Firecrawl 额度),提取走 Firecrawl(免费 1000 credits/月)
2. **服务端搜索免反爬**:反爬是客户端直抓的问题,服务端执行彻底绕开
3. **成本极低**:搜索按 token 计费(缓存命中 0.02 元/百万),提取 1 credit/页

## 附:Firecrawl 注册

2026 年 6 月起 Firecrawl 推出 **Keyless 模式**(免 key 每月 1000 credits),但 Hermes 插件仍需要 key——邮箱注册即可拿到 `fc-` 开头的 key,免费额度无需绑卡。

## 参考

- DeepSeek Responses API 文档:https://api-docs.deepseek.com/zh-cn/guides/responses_api
- Firecrawl Keyless 公告:https://www.firecrawl.dev/blog/firecrawl-keyless-launch