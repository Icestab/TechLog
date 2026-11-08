---
title: "coscmd 迁移 coscli：密钥终于不明文了，还白嫖了增量同步（附两个坑）"
date: 2026-11-08T20:00:00+08:00
draft: false
description: "每周备份上传从 coscmd 迁到腾讯云 coscli：密钥从明文变默认加密，sync 增量同步省掉每周 461MB 重传；附两个默认行为的坑与修法。"
tags: ["腾讯云COS", "coscli", "coscmd", "备份", "云存储"]
categories: ["踩坑记录"]
---

这周把每周备份上传链路从 coscmd 迁到了 coscli。不是故障，是一次主动换工具——旧的 coscmd 用着有三个不满意，而新工具 coscli 恰好一个一个解决了。迁移过程里踩了两个「默认行为」的坑，都不算 bug，但只要照老习惯写脚本就必翻车，值得记一笔。

## 一、为什么要迁：coscmd 的三个不满意

旧脚本每周日 03:30 把 compose 备份包传腾讯云 COS，用的是 pip 装的 coscmd：

1. **过时**：Python 包，基本停更；
2. **不支持同步**：只有 `upload` 全量传，每周重传 461MB 的 tar 包——哪怕它一个字节没变过；
3. **密钥明文**：`~/.cos.conf` 里 SecretId/SecretKey 明文躺着，谁 cat 一下就看见了。

官方新工具 coscli 是 Go 单二进制，支持 `sync` 增量同步。后两条正好对着打。

## 二、迁移前的认知反转：coscli 密钥默认是加密的

动手前查文档，差点得出「换工具也解决不了明文问题」的结论——网上不少旧文写 coscli 的 `~/.cos.yaml` 同样是明文存密钥。结果在官方 config 文档里看到这么一行：

> `--disable-encryption`：设置是否关闭密钥加密。**默认为 false**，若设置为 true，则不加密配置文件中的密钥相关信息。

也就是说 `coscli config init` / `config set` 写入的 SecretId/SecretKey **默认加密存储**。差别就藏在一行文档的默认值里。这才确认迁移值得做。

## 三、安装与配置

查最新版并安装（v1.0.9，2026-08-25 发布，至今仍是 latest）：

```bash
curl -s https://api.github.com/repos/tencentyun/coscli/releases/latest | grep tag_name
# "tag_name": "v1.0.9",

curl -fSL -o ~/.local/bin/coscli https://github.com/tencentyun/coscli/releases/download/v1.0.9/coscli-v1.0.9-linux-amd64
chmod +x ~/.local/bin/coscli && ~/.local/bin/coscli --version
# coscli version v1.0.9
```

`coscli config init` 是交互式逐项提问，密钥不落 shell 历史。各项建议答案：

| 提示 | 建议 |
|---|---|
| Mode | SecretKey（在腾讯云 CVM 上可选 CvmRole 免密钥） |
| Disable Encryption | **false（默认）**——保持密钥加密 |
| DisableAutoFetchBucketType | false（默认），密钥有 `cos:HeadBucket` 权限即可 |
| CloseAutoSwitchHost | false（默认），保留备用域名自动切换容错 |
| ServiceDomain | 回车用默认 `service.cos.myqcloud.com` |
| Bucket Name | `<bucketname>-<appid>` |
| Region / 自定义 Endpoint | 地域照实填；标准桶 endpoint 不用自定义（false） |

## 四、坑一：日志默认路径把自己拦死

把旧脚本里的 `coscmd upload "$FILE" "$COS_PATH"` 换成 `coscli sync`，第一次运行就翻车：

```
INFO[2026-11-08 19:11:25] Upload ~/backuptocos/ to cos://…/compose/ start
ERRO[2026-11-08 19:11:25] failOutputPath ~/backuptocos/coscli_output/ is subdirectory of ~/backuptocos/
```

一开始以为是日志输出目录配置写错了。查 `coscli sync --help` 才发现：`--fail-output-path` 与 `--process-log-path` 的默认值都是相对路径 `coscli_output`，相对**当前工作目录**解析。而旧脚本第一句就是 `cd "$WORK_DIR"`（为了后面 `sha256sum -c` 校验），相对路径于是解析进了同步源目录——coscli 又有条自检「日志输出目录不能是同步源目录的子目录」（防止日志目录下一轮被当数据同步上去），自己把自己拦死了。

**根因：默认相对路径 + 脚本 cd 进源目录。** 它默认把日志放哪儿的规则，和它自己的目录校验规则天然冲突——不改参数必踩。修法是把两个日志目录显式指到源目录之外：

```bash
COSCLI_LOG_DIR="/tmp/coscli_output"   # 不能放源目录里，否则 sync 报错
coscli sync "$WORK_DIR/" "$COS_PATH" \
    --fail-output-path "$COSCLI_LOG_DIR" \
    --process-log-path "$COSCLI_LOG_DIR"
```

顺带一提：这个坑只在「工作目录 = 源目录」时出现。在别的目录跑裸命令 `coscli sync -r src/ cos://…` 没事，日志落在当前目录的 `coscli_output/`。嫌烦也可以干脆关掉两个日志：`--process-log=false --fail-output=false`。

## 五、坑二：目录同步强制要求 -r

日志参数加上，第二次运行换来一个新报错：

```
ERRO[2026-11-08 19:12:25] localPath:~/backuptocos/ is dir, please use --recursive option
```

我原以为目录同步不需要额外参数——**被推翻**。coscli 对目录同步强制要求 `-r/--recursive`，不做「自动判断」，要求显式声明递归意图。这是防误递归上传大目录的设计，但对 rsync 用户是个认知陷阱：rsync 不加 `-r` 也能跑（`-a` = `-rlptgoD`，递归一直藏在 archive 里），coscli 没有 `-a` 全家桶，拒绝得干脆利落。加上 `-r` 后跑通。

## 六、顺带发现：--exclude 是子串匹配

同步脚本目录验证排除规则，要求把含敏感信息的 `.env` 挡在外面：

```bash
coscli sync -r ~/scripts/ cos://…/scripts/ --exclude ".env" …
# Succeed: Total num: 5, size: 25,598 Byte (25.00 KB). OK num: 5(upload 5 files).

coscli ls cos://…/scripts/
#   scripts/back_compose.sh   3.86 KB
#   scripts/back_exclude.txt  505.00 B
#   scripts/check_monthly.py  18.79 KB
#   scripts/update_pac.sh     393.00 B
#   scripts/upload_cos.sh     1.46 KB
```

本地 7 个文件只上了 5 个：`.env` 被排除 ✅，但 **`.env.example` 也一起被排掉了**——`--exclude ".env"` 是前缀/子串匹配，不是精确文件名匹配。这次被排掉的只是无密钥模板，无损失，但行为要知道。要精确区分得自己调整模式，并用 `coscli ls` 复核结果。

## 七、最终脚本与验证

最终可用的每周备份 + 脚本同步核心段：

```bash
COSCLI="$HOME/.local/bin/coscli"
COSCLI_LOG_DIR="/tmp/coscli_output"   # 不能放源目录里，否则 sync 报错

# 1) 备份包同步（CRC64 增量，未变化的文件直接跳过）
"$COSCLI" sync -r "$WORK_DIR/" "cos://<bucket>/compose/" \
    --fail-output-path "$COSCLI_LOG_DIR" \
    --process-log-path "$COSCLI_LOG_DIR"

# 2) 脚本目录同步（排除含密钥的 .env*）
"$COSCLI" sync -r "$HOME/scripts/" "cos://<bucket>/scripts/" \
    --exclude ".env" \
    --fail-output-path "$COSCLI_LOG_DIR" \
    --process-log-path "$COSCLI_LOG_DIR"
```

手动临时传单个文件不需要 `-r` 和日志参数：`coscli cp /path/to/file cos://<bucket>/compose/`。已写入 crontab，每周日 03:30 自动执行。

首次完整跑通（compose 包当天早晨刚由旧工具传过，正好检验增量）：

```
INFO[…] Upload …/backuptocos/ to cos://…/compose/ Succeed: Total num: 2, size: 483,810,374 Byte (461.40 MB).
       OK num: 2(upload 1 files, skip 1 files), Skip size: 483,810,272 Byte (461.40 MB).
cost 1.243000(s)
```

- 461MB 的 tar 包被 **CRC64 对比直接跳过**，零传输，全程 1.2 秒；
- scripts 目录只传了 1 个变化文件（脚本自己），4 个跳过——增量判断精准；
- `coscli ls` 确认桶内对象齐全、`.env` 不在列。

## 八、注意点备忘

- **别手改 `~/.cos.yaml`**：默认是加密的，手改会破坏密文；改配置走 `coscli config set`（`--secret_id`/`--secret_key` 更新后自动加密），别直接编辑（除非用 `--disable-encryption` 关掉加密——不建议）。
- **回滚很简单**：旧脚本和 coscmd 保留一周，crontab 切回去即可。建议观察一个 cron 周期再删旧工具。
- **收尾别忘轮换密钥**：旧 SecretId/Key 在明文配置里躺了很久，迁移稳定后去 CAM 轮换一遍，新的用 `coscli config set` 更新。
- `--exclude` 的完整 glob 语义官方文档没写明，涉及密钥/敏感文件的排除规则，重要场景都用 `coscli ls` 复核。脚本目录嵌套场景这次没测。

## 九、可迁移的结论

1. **工具自带的「默认相对路径」要放进调用方的工作目录语境里检查**——`coscli_output` 这类默认值，在 cd 过的脚本里会落到意想不到的位置，还可能和工具自身的目录校验打架。脚本里调工具，日志/输出路径一律显式给绝对路径。
2. **换云存储 CLI 前先核对「密钥存储」这栏**：coscmd 明文 vs coscli 默认加密，差别就藏在 `--disable-encryption` 默认值一行文档里。同类工具（rclone 的 `obscure`、AWS CLI 的 credential 文件权限）各有各的口径，值得单独查。
3. **排除规则的语义不写在文档里时，用 `ls` 复核是唯一可靠的验证**——尤其涉及密钥文件的排除，「以为排掉了」和「真排掉了」必须用结果确认。
