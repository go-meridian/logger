# 架构约定

## LogWriter 文件轮转

`writer.go` 是核心，互斥锁（`sync.Mutex`）保护，双重维度轮转：

- **日期轮转**：写入前比较当前日期与 `curDate`，跨天则关闭旧文件、重置 `curSeq`、打开新文件
- **大小轮转**：`curSize + len(p) > maxSize` 时关闭旧文件、`curSeq++`、打开新文件
- **旧文件清理**：仅在日期轮转或初始化时扫描（`openNew(true)`），避免大小轮转时无谓扫描目录；删除超过 `maxAge` 天的文件
- **文件命名**：`{prefix}.{YYYYMMDD}.{seq}`（如 `app.20260920.1`）
- **活跃状态**：`active` 字段标记，`Close()` 后 `Write` 返回 `log writer closed`
- **容错重开**：上次写入失败导致 `file == nil` 时，下次 `Write` 自动重新打开

## 全局单例与原子读取

`logger.go`：

- `logValue` 为 `atomic.Value`，`Get()` 无锁读取，永不返回 nil
- `init()` 阶段预存 `zap.NewNop()`，未初始化时日志为静默丢弃
- `Init()` 仅可调用一次，重复调用会泄漏前一次日志文件句柄
- `Close()` 幂等（`sync.Once`）：`Sync()` 刷新缓冲区 → 关闭 `mainW`、`errW` → 重置为 Nop

## 错误日志独立输出

`Config.ErrorFile` 非空时启用：

- 创建独立 `LogWriter`（前缀为 `ErrorFile`）+ error 级别 enabler（`lvl >= ErrorLevel`）
- 通过 `zapcore.NewTee(mainCore, errorCore)` 组合双 core，error 日志同时写入主文件和错误文件
- `errW` 包级变量持有错误写入器，`Close()` 时一并关闭

## Logger 封装

`zap.go`：

- `Logger` 结构体内部持有 `*zap.Logger`
- `L()` 通过 `Get()` 无锁获取全局实例并包装
- `Wrap(zl)` 包装已有 `*zap.Logger`
- 提供 `With`/`Named` 链式方法，`Z()` 返回底层 `*zap.Logger` 供特殊场景
