---
name: hilog
description: >
  查询鸿蒙 HarmonyOS 设备上的应用日志。当用户需要：(1) 查看应用实时日志，
  (2) 按 Tag/关键词/级别过滤日志，(3) 排查应用崩溃或异常，(4) 查看指定应用的日志，
  (5) 导出日志到文件，(6) 清空日志缓冲区，
  或提到 hilog、hdc、日志、log、crash、崩溃、报错 等关键词时触发。
argument-hint: "[bundleName 或 filter 关键词]"
---

# HarmonyOS 应用日志查询

查询鸿蒙设备上运行中应用的日志信息。支持筛选、导出、崩溃抓取。

## 前置条件

- hdc 已安装且在 PATH 中（`hdc version` 可执行）
- 设备已通过 USB 或无线连接（`hdc list targets` 可看到设备）

## 关键规则

1. **所有 hilog 命令必须用 `hdc shell "hilog ..."` 双引号包裹**，避免中文和特殊字符的 Shell 转义问题
2. **每次执行都必须重新检测设备和 PID**，不要缓存旧值。应用重新部署后 PID 会变，用旧 PID 会静默无输出
3. **用户只需提供 bundleName**，skill 内部自动完成 pidof → hilog 的串联，不需要用户手动获取 PID

## 执行流程

### 1. 自动检测设备（每次都执行）

```bash
hdc list targets
```

- **一台设备：** 直接使用，后续命令不加 `-t`
- **多台设备：** 列出列表让用户选择，本次会话记住选择。后续所有命令加 `-t <序列号>`
- **没有设备：** 提示检查 USB 连接和调试开关，建议 `hdc kill -r` 重启服务
- 设备断连重连后序列号可能变化，所以每次都要重新检测

### 2. 确定目标应用

如果用户提供了 bundleName，直接使用。否则：

```bash
hdc shell "aa dump -a | grep -E 'bundle name|ability name' | head -20"
```

### 3. 获取应用 PID（每次都重新获取）

```bash
hdc shell pidof <bundleName>
```

- 如果 PID 与上次不同，输出提示：`应用已重启，PID 从 xxx 变更为 yyy`
- 如果返回空（应用未运行），提示用户启动应用或帮助启动：

```bash
hdc shell aa start -a <abilityName> -b <bundleName>
# 等待1秒后重新获取
hdc shell pidof <bundleName>
```

### 4. 查看日志

**推荐排查流程（用户没有明确指定过滤条件时，按此顺序）：**

```
第一步：看错误和警告（快速定位）
  hdc shell "hilog -P <pid> -L W,E,F -z 100"

第二步：按 Tag 过滤（缩小范围）
  hdc shell "hilog -P <pid> -T <tag1>,<tag2> -z 200"

第三步：按关键词搜索（精确查找）
  hdc shell "hilog -P <pid> -e 'keyword1|keyword2' -z 200"

第四步：看全量应用日志（全面排查）
  hdc shell "hilog -P <pid> -t app -z 500"
```

如果第一步无错误/警告输出，主动告知用户并建议执行第二步。

**各类过滤方式：**

查看最近 N 行日志：
```bash
hdc shell "hilog -P <pid> -z <N>"
```

按日志级别过滤：
```bash
hdc shell "hilog -P <pid> -L W,E,F"
# 级别: D(DEBUG) I(INFO) W(WARN) E(ERROR) F(FATAL)
```

按 Tag 过滤：
```bash
hdc shell "hilog -P <pid> -T <tag1>,<tag2> -z <N>"
```

按关键词正则过滤：
```bash
hdc shell "hilog -P <pid> -e '<正则表达式>' -z <N>"
```

组合过滤：
```bash
hdc shell "hilog -P <pid> -L E,F -T <tag> -z <N>"
```

只看应用日志：
```bash
hdc shell "hilog -P <pid> -t app -z <N>"
```

### 5. 崩溃日志抓取

当用户反馈应用崩溃、闪退、无响应时：

```bash
# 崩溃相关日志
hdc shell "hilog -e 'FATAL|crash|signal|backtrace|abort|SIGABRT|SIGSEGV|NativeCrash' -z 200"

# 应用 FATAL 日志
hdc shell "hilog -P <pid> -L F -z 100"

# ANR/卡死
hdc shell "hilog -P <pid> -e 'timeout|ANR' -z 100"
```

如果应用已崩溃（pidof 无输出），在全局日志中搜索：
```bash
hdc shell "hilog -e '<bundleName>' -L E,F -z 200"
```

### 6. 导出日志到文件

**优先使用重定向方式导出**（避免 `hdc file recv` 在 Git Bash 下的路径转义问题）：

```bash
# 导出到用户指定路径
hdc shell "hilog -P <pid> -z 500" > <输出路径>

# 导出特定 Tag
hdc shell "hilog -P <pid> -T <tag> -z 500" > <输出路径>

# 导出错误日志
hdc shell "hilog -P <pid> -L E,F -z 500" > <输出路径>
```

**默认文件名：** 用户未指定路径时，使用 `<bundleName最后一段>_<yyyyMMdd_HHmmss>.log` 保存到当前工作目录。

**导出后自动读取分析：** 使用 Read 工具读取导出文件，统计 Tag 分布、错误信息、关键事件。

### 7. 其他操作

```bash
# 清空日志缓冲区
hdc shell "hilog -r"

# 查看缓冲区大小
hdc shell "hilog -g"

# 设置日志级别（临时，重启失效）
hdc shell "hilog -b D"
hdc shell "hilog -b D -T <tag>"
```

## hilog 参数速查

| 参数 | 说明 | 示例 |
|------|------|------|
| `-P <pid>` | 按进程 ID 过滤 | `-P 52178` |
| `-T <tag>` | 按 Tag 过滤（最多 10 个） | `-T MyTag,OtherTag` |
| `-L <level>` | 按级别过滤 | `-L W,E,F` |
| `-D <domain>` | 按 domain 过滤（最多 5 个） | `-D 0xD0A0510` |
| `-t <type>` | 按日志类型过滤 | `-t app` |
| `-e <regex>` | 正则表达式过滤 | `-e "error\|exception"` |
| `-z <n>` | 显示最近 n 行 | `-z 100` |
| `-a <n>` | 显示最早 n 行 | `-a 50` |
| `-v color` | 彩色输出 | `-v color` |
| `-r` | 清空日志 | `-r` |
| `-g` | 查看缓冲区大小 | `-g` |
| `-b <level>` | 设置可输出级别 | `-b D` |

## 日志格式说明

```
月-日 时:分:秒.毫秒  PID  TID  级别  Domain/BundleName/Tag: 内容
03-19 20:43:07.283  52178 52178 I A04510/com.example.app/MyTag: Hello World
```

- **级别**: D(调试) I(信息) W(警告) E(错误) F(致命)
- **Domain**: app 范围 `0x0~0xFFFF`，系统范围 `0xD000000~0xD0FFFFF`（过滤时加 `0xD0` 前缀）

## 注意事项

- `-z`（tail）和 `-a`（head）不能同时使用
- 实时日志是阻塞式的，会持续输出直到 Ctrl+C
- Tag 最多 10 个，PID 和 Domain 最多 5 个
- 日志缓冲区默认 512K，日志量大时会被快速覆盖
- **所有命令用 `hdc shell "hilog ..."` 双引号包裹**

## $ARGUMENTS 处理

如果用户提供了参数：
- 如果参数像 bundleName（含 `.`），自动 pidof 获取 PID 后查看日志
- 如果参数像 Tag 或关键词，用作 `-T` 或 `-e` 的过滤条件
- 如果参数是 `clear`/`清空`，执行 `hilog -r`
- 如果参数是 `export`/`导出`，导出日志到文件
- 如果参数是 `crash`/`崩溃`，执行崩溃日志专项抓取
