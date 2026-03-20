# harmonyos-hilog-skill

HarmonyOS 应用日志查询 Skill，用于 [Claude Code](https://claude.ai/code)。

通过 `hdc` + `hilog` 命令查询鸿蒙设备上的应用日志，支持按 Tag/级别/关键词筛选、崩溃日志抓取、日志导出到文件。

## 安装

将 `skills/hilog/` 目录复制到以下任一位置：

```bash
# 个人全局（所有项目可用）
cp -r skills/hilog ~/.claude/skills/hilog

# 项目级别（仅当前项目可用）
cp -r skills/hilog .claude/skills/hilog
```

## 前置条件

- `hdc` 已安装且在 PATH 中
- 设备已通过 USB 或无线连接

## 使用方式

在 Claude Code 中直接使用：

```
/hilog com.example.myapp          # 查看指定应用日志
/hilog crash                      # 抓取崩溃日志
/hilog export                     # 导出日志到文件
/hilog clear                      # 清空日志缓冲区
/hilog ChatVM                     # 按 Tag 筛选
```

也可以用自然语言触发：
- "查看应用日志"
- "应用崩溃了，帮我看看日志"
- "把日志导出到桌面"
- "筛选错误级别的日志"

## 功能

| 功能 | 说明 |
|------|------|
| 日志查看 | 按 PID 查看指定应用的最近 N 行日志 |
| Tag 筛选 | `-T` 按 Tag 过滤，支持多个 Tag |
| 级别筛选 | `-L` 按 D/I/W/E/F 级别过滤 |
| 关键词搜索 | `-e` 正则表达式匹配日志内容 |
| 崩溃抓取 | 自动搜索 FATAL/crash/signal/backtrace 等关键词 |
| 导出文件 | 导出到自定义路径，导出后自动读取分析 |
| 多设备支持 | `-t` 指定目标设备 |
| 缓冲区管理 | 清空日志、查看/设置缓冲区大小 |

## 许可

MIT
