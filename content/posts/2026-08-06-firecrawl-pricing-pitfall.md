---
title: "Firecrawl 计费陷阱:PDF 提取按页计费"
date: 2026-08-06T22:30:00+08:00
tags: ["Firecrawl", "计费", "PDF", "省钱"]
categories: ["技术实践"]
draft: false
---

> 一次 82 页 PDF 提取烧掉 84 credits(月度免费额度的 8.4%),血的教训。

## 计费规则(官方确认)

| 操作 | 消耗 |
|------|------|
| 搜索 | 2 credits / 10 条结果(向上取整) |
| HTML 提取 | 1 credit / 页 |
| **PDF 提取** | **按页计费**(82 页 ≈ 84 credits) |
| 搜索+自动抓正文 | 搜索 2 + 每结果 1 credit |

免费额度:1000 credits/月。

## 教训

**PDF 绝对不要直接扔给 Firecrawl 提取**——按页计费,一个几十页的 PDF 就能烧掉月度额度的百分之几到十几。

## 正确做法

```
PDF  → curl 下载到本地 → 本地解析(pymupdf/read_file) → 免费
HTML → Firecrawl(1 credit/页) 或 curl
搜索 → DeepSeek 服务端(不耗 Firecrawl 额度)
```

实测对比:
- 82 页 PDF 扔 Firecrawl:84 credits
- 同 PDF curl + 本地解析:0 credits

## 为什么 Firecrawl 搜索也烧额度

Firecrawl 的搜索**默认会抓每个结果的正文**("search + extraction in one call")——一次搜索 10 条结果 = 2(搜索)+ 10(抓取)= 12 credits。所以搜索密集场景要小心,最好用纯搜索服务(如 DeepSeek web_search)。

## 总结

1. **PDF 永远走本地解析**(curl + pymupdf),零成本
2. **Firecrawl 只用于 HTML 提取**(1 credit/页,1000 次/月用不完)
3. **搜索交给模型服务端**(不耗 Firecrawl 额度)