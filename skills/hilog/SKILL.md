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

## 执行流程

### 1. 确认设备连接

```bash
hdc list targets
```

如果有多个设备，**所有后续 hdc 命令都需加 `-t <设备序列号>`**：
```bash
# 示例：指定设备执行命令
hdc -t 9CN0123529000015 shell hilog -z 50
hdc -t 127.0.0.1:5555 shell pidof com.example.app
```

如果只有一个设备，无需加 `-t`。

### 2. 确定目标应用

如果用户提供了 bundleName，直接使用。否则：

```bash
# 列出正在运行的前台应用
hdc shell "aa dump -a | grep -E 'bundle name|ability name' | head -20"
```

### 3. 获取应用 PID

```bash
hdc shell pidof <bundleName>
```

如果应用未运行（无输出），提示用户先启动应用，或帮助启动：

```bash
hdc shell aa start -a <abilityName> -b <bundleName>
# 启动后等待1秒再获取 PID
hdc shell pidof <bundleName>
```

### 4. 查看日志

根据用户需求选择合适的命令：

**查看最近 N 行日志（最常用）：**
```bash
hdc shell hilog -P <pid> -z <N>
```

**按日志级别过滤：**
```bash
# 只看警告和错误
hdc shell hilog -P <pid> -L W,E,F
# 级别: D(DEBUG) I(INFO) W(WARN) E(ERROR) F(FATAL)
```

**按 Tag 过滤：**
```bash
hdc shell hilog -P <pid> -T <tag1>,<tag2> -z <N>
```

**按关键词正则过滤：**
```bash
hdc shell hilog -P <pid> -e "<正则表达式>" -z <N>
```

**组合过滤（PID + 级别 + Tag）：**
```bash
hdc shell hilog -P <pid> -L E,F -T <tag> -z <N>
```

**只看应用日志（排除系统日志）：**
```bash
hdc shell hilog -P <pid> -t app -z <N>
```

### 5. 崩溃日志抓取

当用户反馈应用崩溃、闪退、无响应时，使用以下命令抓取关键信息：

```bash
# 抓取崩溃相关日志（FATAL + 堆栈 + 信号）
hdc shell "hilog -e 'FATAL|crash|signal|backtrace|abort|SIGABRT|SIGSEGV|NativeCrash' -z 200"

# 抓取应用级别的 FATAL 日志
hdc shell hilog -P <pid> -L F -z 100

# 抓取 ffrt 超时（ANR/卡死相关）
hdc shell "hilog -P <pid> -e 'timeout|ANR' -z 100"
```

如果应用已崩溃（pidof 无输出），不带 `-P` 在全局日志中搜索 bundleName：
```bash
hdc shell "hilog -e '<bundleName>' -L E,F -z 200"
```

### 6. 导出日志到文件

将日志导出到本地文件，支持用户自定义路径：

```bash
# 导出到指定路径（用户指定或默认当前目录）
hdc shell hilog -P <pid> -z 500 > <输出路径>

# 导出特定 Tag 的日志
hdc shell hilog -P <pid> -T <tag> -z 500 > <输出路径>

# 导出错误日志
hdc shell hilog -P <pid> -L E,F -z 500 > <输出路径>

# 导出全部应用日志
hdc shell hilog -P <pid> -t app -z 1000 > <输出路径>
```

**导出后自动读取文件内容进行分析：**
导出完成后，使用 Read 工具读取导出的日志文件，帮助用户分析日志内容（统计 Tag 分布、错误信息、关键事件等）。

**默认文件名规则：** 如果用户未指定路径，使用 `<bundleName简称>_<时间戳>.log` 保存到当前工作目录。

### 7. 其他操作

**清空日志缓冲区：**
```bash
hdc shell hilog -r
```

**查看日志缓冲区大小：**
```bash
hdc shell hilog -g
```

**设置日志级别（临时，重启失效）：**
```bash
# 全局设置为 DEBUG 级别
hdc shell hilog -b D
# 针对特定 Tag
hdc shell hilog -b D -T <tag>
```

## hilog 参数速查

| 参数 | 说明 | 示例 |
|------|------|------|
| `-P <pid>` | 按进程 ID 过滤 | `-P 52178` |
| `-T <tag>` | 按 Tag 过滤（最多 10 个） | `-T MyTag,OtherTag` |
| `-L <level>` | 按级别过滤 | `-L W,E,F` |
| `-D <domain>` | 按 domain 过滤（最多 5 个） | `-D 0xD0A0510` |
| `-t <type>` | 按日志类型过滤 | `-t app`（只看应用日志） |
| `-e <regex>` | 正则表达式过滤 | `-e "error\|exception"` |
| `-z <n>` | 显示最近 n 行 | `-z 100` |
| `-a <n>` | 显示最早 n 行 | `-a 50` |
| `-v color` | 彩色输出 | `-v color` |
| `-v long` | 详细格式 | `-v long` |
| `-r` | 清空日志 | `-r` |
| `-g` | 查看缓冲区大小 | `-g` |
| `-b <level>` | 设置可输出级别 | `-b D`（设为 DEBUG） |

## 日志格式说明

```
月-日 时:分:秒.毫秒  PID  TID  级别  Domain/BundleName/Tag: 内容
03-19 20:43:07.283  52178 52178 I A04510/com.example.app/MyTag: Hello World
```

- **级别**: D(调试) I(信息) W(警告) E(错误) F(致命)
- **Domain**: 十六进制标识，app 类型范围 `0x0~0xFFFF`，系统类型范围 `0xD000000~0xD0FFFFF`
- 过滤系统日志的 domain 时需加 `0xD0` 前缀

## 注意事项

- `-z`（tail）和 `-a`（head）不能同时使用
- `-z`/`-a` 和 `-x`（非阻塞退出）不能同时使用
- 实时日志是阻塞式的，会持续输出直到 Ctrl+C
- Tag 最多 10 个，PID 和 Domain 最多 5 个
- `hilog -b` 设置的级别是临时的，重启后恢复默认
- 日志缓冲区默认 512K，日志量大时会被快速覆盖，需要及时抓取

## $ARGUMENTS 处理

如果用户提供了参数：
- 如果参数像 bundleName（含 `.`），用它获取 PID 后查看日志
- 如果参数像 Tag 或关键词，用它作为 `-T` 或 `-e` 的过滤条件
- 如果参数是 `clear`/`清空`，执行 `hilog -r` 清空日志
- 如果参数是 `export`/`导出`，导出日志到文件
- 如果参数是 `crash`/`崩溃`，执行崩溃日志专项抓取
