# 代码规范

## 包与命名

- 包名统一为 `logger`，所有文件 `package logger`
- 注释与文档使用中文

## Config 结构体

`config.go` 字段（json/yaml tag 同名）：

| 字段 | tag | 说明 |
|------|-----|------|
| `Level` | `level` | 日志级别：debug/info/warn/error |
| `LogFile` | `logFile` | 日志文件名前缀 |
| `ErrorFile` | `errorFile` | 错误日志前缀，空则不单独输出 |
| `LogDir` | `logDir` | 日志目录，默认 `logs` |
| `MaxSize` | `maxSize` | 单文件最大 MB，默认 500 |
| `MaxAge` | `maxAge` | 旧日志保留天数，默认 30 |

`DefaultConfig(logFile)` 返回默认配置（Level=info、LogDir=logs、MaxSize=500、MaxAge=30）。

修改 Config 时需考虑向后兼容，Lobby 不一定使用所有字段。

## field.go 导出约定

- 字段构造函数与 zap **同名、同签名**，方便消费方迁移（`logger.String(...)` 代替 `zap.String(...)`）
- 类型别名：`Field = zapcore.Field`、`ObjectMarshaler`、`ArrayMarshaler`
- 导出完整集合：基础类型、错误、时间、复合（Any/Reflect/Object/Array/Dict）、指针（`*p` 后缀）、切片（`s` 后缀）
- Lobby 实际高频使用的：`String`、`Uint64`、`Error`、`Int`、`Int64`

## 消费方约定

- Lobby 中**零 zap 直接引用**，全部通过 logger 包操作
- 模块统一接收 `*logger.Logger`，存为包级变量 `var log *logger.Logger`
- **不用** `logger.Get()`（仅 mq 库 vendor 依赖内部使用）；**不用** `InitWithDefault()`，始终 `Init()` + 显式 Config
