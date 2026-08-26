---
title: "在 Android/Termux 上使用 Adreno 840 + Vulkan 部署 llama.cpp 本地大模型"
date: 2026-08-25T20:00:00+08:00
tags: ["llama.cpp", "Vulkan", "Android", "Termux", "Adreno", "本地大模型"]
categories: ["技术实践"]
draft: false
---

> 随着移动端 SoC 性能越来越强,现在已经可以直接在 Android 手机上运行 GGUF 格式的大语言模型。本文记录一次实际部署过程:使用 Termux + llama.cpp + Mesa Freedreno/Turnip + Vulkan,让高通 Adreno GPU 参与 LLM 推理。
>
> 本文以 Adreno 840 为例,并使用 LFM2.5-2.6B-Q8_0 进行测试。

---

## 一、环境

本次测试环境:

| 项目 | 配置 |
|---|---|
| 系统 | Android |
| SoC | Qualcomm Snapdragon |
| GPU | Adreno 840 |
| 推理框架 | llama.cpp |
| llama.cpp | b10516 |
| GPU Backend | Vulkan |
| Mesa | 26.0.6 |
| Vulkan 驱动 | Freedreno / Turnip |
| 模型 | LFM2.5-2.6B-Q8_0 |
| 模型大小 | 约 2.7 GB |

> 手机使用的是统一内存架构,因此 `vulkaninfo` 显示的 GPU memory 并不是传统 PC 独立显卡意义上的 VRAM。

---

## 二、安装 Termux

建议使用官方 Termux 项目提供的版本,不建议使用来源不明的第三方修改版。

进入 Termux 后首先更新软件包:

```bash
pkg update
pkg upgrade
```

然后安装 llama.cpp:

```bash
pkg install llama-cpp
```

---

## 三、安装 Vulkan Backend

llama.cpp 本身只是推理框架,GPU 后端需要单独安装。

Termux 当前提供 Vulkan backend:

```bash
pkg install llama-cpp-backend-vulkan
```

同时安装 Vulkan 工具:

```bash
pkg install vulkan-tools
```

---

## 四、安装 Adreno Vulkan 驱动

对于高通 Snapdragon + Adreno GPU,可以安装 Mesa 的 Freedreno Vulkan ICD:

```bash
pkg install mesa-vulkan-icd-freedreno
```

这里的关键组件是:

```text
llama.cpp
    ↓
Vulkan Backend
    ↓
Vulkan Loader
    ↓
Mesa Freedreno / Turnip
    ↓
Adreno GPU
```

`mesa-vulkan-icd-freedreno` 是让 Mesa/Turnip 为 Adreno 提供 Vulkan 支持的重要组件。

---

## 五、验证 Vulkan

安装完成后,可以使用:

```bash
vulkaninfo --summary
```

查看 Vulkan 是否正常工作。

也可以直接:

```bash
vulkaninfo --summary | grep -i devicename
```

正常情况下应该能够看到 Adreno GPU,而不是:

```text
llvmpipe
```

例如本次测试环境能够识别:

```text
Adreno (TM) 840
```

---

## 六、验证 llama.cpp 是否识别 GPU

仅仅 `vulkaninfo` 能看到 GPU 还不够,最好直接让 llama.cpp 检查。

执行:

```bash
llama-cli --list-devices
```

本次测试得到:

```text
Available devices:
  Vulkan0: Adreno (TM) 840 (11313 MiB, 7105 MiB free)
```

这意味着 llama.cpp 已经成功发现 Vulkan GPU。

至此:

> Android → Termux → llama.cpp → Vulkan → Adreno 840

整个 GPU 推理链路已经打通。

---

## 七、准备 GGUF 模型

llama.cpp 使用 GGUF 格式模型。

本次使用:

```text
LFM2.5-2.6B-Q8_0.gguf
```

文件大小约:

```text
2.7 GB
```

对于 Adreno 840 来说,这个模型非常适合作为测试模型。

Q8_0 的优势是量化损失较小,同时模型体积又明显低于 FP16,因此非常适合移动端测试。

将模型放到 Termux 可以访问的目录,例如:

```bash
mkdir -p ~/models
```

然后确认:

```bash
ls -lh ~/models/
```

> 提示:如果模型在 PC 上,可以通过 `scp` 或 Termux 内置的 `termux-setup-storage` 配合存储权限直接拉取,避免手机流量。

---

## 八、使用 Vulkan 运行模型

最简单的运行方式:

```bash
llama-cli \
  -m ~/models/LFM2.5-2.6B-Q8_0.gguf \
  -ngl 999
```

其中:

```text
-ngl 999
```

表示尽可能将模型层全部 offload 到 GPU。

如果 Vulkan GPU 正常工作,启动日志中可以看到 Vulkan 设备被使用。

---

## 九、推荐配置

实际使用时,可以使用下面这组参数:

```bash
llama-cli \
  -m ~/models/LFM2.5-2.6B-Q8_0.gguf \
  -ngl 999 \
  -c 4096 \
  -b 512 \
  -ub 512 \
  --cache-type-k q8_0 \
  --cache-type-v q8_0 \
  --threads 6 \
  --threads-batch 8 \
  --temp 0.6 \
  --jinja \
  -fa on
```

下面解释几个比较重要的参数。

---

### `-ngl 999`

让 llama.cpp 尽可能把模型层放到 GPU。

移动端 Vulkan 推理基本应该优先尝试:

```text
-ngl 999
```

而不是手动设置一个很小的值。

---

### `-c 4096`

设置上下文长度为 4096 tokens。

上下文越大,KV Cache 占用的内存越多。

如果手机内存比较紧张,可以降低到:

```text
-c 2048
```

如果内存充足,则可以进一步尝试:

```text
-c 8192
```

但上下文长度并不是越大越快。

---

### `-b 512`

设置 batch size。

它主要影响 Prompt Processing。

较大的 batch 通常可以提高 GPU 对长 Prompt 的处理效率,但同时会增加内存需求。

---

### `-ub 512`

这是 unified batch size,也就是 ubatch。

同样主要影响批处理过程。

移动设备上不建议一开始就把数值设置得特别大,256~512 是比较合理的测试范围。

---

## 十、KV Cache 量化

本次测试使用:

```text
--cache-type-k q8_0
--cache-type-v q8_0
```

KV Cache 量化的主要目的不是提高模型智力,而是:

> 降低 KV Cache 的内存占用。

特别是在 8K、16K 甚至更长上下文时,KV Cache 会越来越大。

如果只是测试模型性能,也可以使用默认 KV Cache 设置进行对比。

---

## 十一、CPU 线程

配置:

```text
--threads 6
--threads-batch 8
```

其中:

- `--threads` 主要影响生成阶段
- `--threads-batch` 主要影响 Prompt Processing

移动端 CPU 通常采用大小核架构,因此线程数并不是越高越快。

例如可以实际测试:

```text
4 threads
6 threads
8 threads
```

找到最适合当前 Snapdragon SoC 的配置。

---

## 十二、Jinja

本次使用:

```text
--jinja
```

它用于让 llama.cpp 使用模型对应的 Jinja Chat Template。

对于支持聊天模板的模型,建议开启。

否则模型可能无法按照官方预期的方式组织 `system / user / assistant` 等消息。

不过它本身不会让 GPU 变快。

---

## 十三、Flash Attention

当前版本 llama.cpp 的参数格式是:

```text
-fa on
```

而不是简单的 `-fa`。

可以使用:

```text
-fa on
```

强制开启。

也可以:

```text
-fa off
```

关闭。

默认:

```text
-fa auto
```

---

## 十四、实际性能测试

使用:

```text
LFM2.5-2.6B-Q8_0.gguf
```

在 Adreno 840 + Vulkan 环境下进行测试。

关闭 Flash Attention 时:

```text
Prompt:      9.6 t/s
Generation: 10.2 t/s
```

开启 `-fa on` 之后:

```text
Prompt:     34.7 t/s
Generation: 10.3 t/s
```

可以看到一个比较有意思的现象:

- **Prompt Processing**:从 9.6 t/s 提升到 34.7 t/s,提升非常明显
- **Generation**:基本没有变化(10.2 → 10.3 t/s)

也就是说:

> Flash Attention 对本次测试的 Prompt Processing 有明显帮助,但对逐 token Generation 几乎没有提升。

这也是移动端 GPU 推理中比较值得注意的一点。

---

## 十五、为什么 Generation 只有约 10 t/s?

虽然 Adreno 840 是非常强的移动 GPU,但不能简单按照 PC 显卡的思路来理解。

LLM Generation 与 Prompt Processing 的工作特征不同。

**Prompt Processing** 可以进行大量并行计算:

```text
大量 token
   ↓
GPU 并行计算
   ↓
较高吞吐
```

而 **Generation** 是:

```text
生成一个 token
       ↓
读取模型权重
       ↓
计算
       ↓
生成下一个 token
       ↓
重复
```

因此 Generation 很容易受到:

- GPU 内存带宽
- Kernel 实现
- Vulkan backend 优化程度
- 量化格式
- KV Cache
- Android GPU 驱动

等因素影响。

所以:

```text
Prompt 34.7 t/s
Generation 10.3 t/s
```

并不意味着 Adreno 840 的计算能力只有 10 tok/s。

这是当前这套 `llama.cpp + Vulkan` 软件栈下的实际推理吞吐。

---

## 十六、作为服务使用:llama-server(Open WebUI 可选)

单次的 `llama-cli` 适合交互式测试和跑 benchmark,但如果你想让它变成一个可以随时打开的本地 AI 服务,建议用 **llama-server**(llama.cpp 自带):

```bash
pkg install llama-cpp-server
```

启动服务:

```bash
llama-server \
  -m ~/models/LFM2.5-2.6B-Q8_0.gguf \
  -ngl 999 \
  -c 4096 \
  --jinja \
  -host 127.0.0.1 \
  -port 8080
```

启动后,`llama-server` 会暴露一个 **OpenAI 兼容的 HTTP API**(`/v1/chat/completions` 等),手机上的任何 OpenAI 兼容客户端都能直接调用:

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "LFM2.5-2.6B",
    "messages": [{"role": "user", "content": "你好"}]
  }'
```

llama.cpp 的 server 还自带一个极简的 Web 聊天界面(`http://127.0.0.1:8080/`),不装任何东西就能先聊起来。

### 可选:Open WebUI

如果你想要更完整的 Web 聊天体验,可以让 `llama-server` 和 **Open WebUI** 搭配使用:

```bash
# Termux 里跑 OpenAI 兼容的 llama-server(如上)
# 再在另一台设备/容器上跑 Open WebUI 前端
docker run -d -p 3000:8080 \
  -e "OPENAI_API_BASE_URL=http://手机IP:8080/v1" \
  ghcr.io/open-webui/open-webui:main
```

> 注意:Open WebUI 本身对 Termux/Android 不是原生安装,通常建议在 NAS / PC 上跑前端,通过局域网指向手机的 llama-server。手机上直接用自带的 Web 界面就够用了。

---

## 十七、总结与注意事项

到这里,一套完整的"Android 手机本地大模型"方案就落地了:

```text
Android + Termux
    ↓
llama.cpp + Vulkan backend
    ↓
Mesa Freedreno/Turnip 驱动
    ↓
Adreno GPU 加速
    ↓
GGUF 模型(本地推理,完全离线)
```

### 值得记住的几点

1. **离线可用**:模型完全在本地,不需要联网,数据不出手机——这是相比云端最大的优势。
2. **Prompt 快、Generation 慢是常态**:移动端 GPU 受内存带宽和驱动栈限制,单 token 生成速度通常远低于 PC 独显。别只看峰值。
3. **Flash Attention 值得开**:对长 Prompt 输入有明显提升,且对生成速度几乎无损,默认建议 `-fa on`。
4. **上下文大小按内存调**:2.6B 模型 8K 上下文是可行的,但更小的模型(likely 1.5B)会更从容,适合跑在手机上长期待机。
5. **功耗与发热**:持续 Generation 会让手机发热、掉电快,长任务建议接电源并注意散热。
6. **版本差异**:llama.cpp 迭代很快,参数格式(如 `-fa on`)在不同版本间会有变化,以 `llama-cli --help` 和你安装的版本文档为准。

### 写在最后

在 Adreno 840 + Vulkan 这套软件栈下,LFM2.5-2.6B-Q8_0 能跑到 Prompt 34.7 t/s、Generation 10.3 t/s,已经足够支持日常的对话、翻译、简单编码辅助等轻量场景。如果只是想测试手机 GPU 的推理能力,或者需要一个完全离线的本地助手,这套方案非常值得一试。