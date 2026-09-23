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

## 日志输出格式（encoder.go）

自定义 `keyValueEncoder` 实现完整 `zapcore.Encoder`（含 ObjectEncoder 全部 AddXXX），替代 zap 默认 console encoder（tab 分隔、小写 level），输出单行、grep 友好的格式：

```
DATE[2026-09-23T13:10:36.426+0800]  INFO  RouteCmd  cmd=xxx uid=123 requestId=abc-123
```

- 头部固定：`DATE[时间] 大写级别 消息`，双空格分隔
- 字段用 `key=value`，空格分隔；含空格/引号/反斜杠的值用 `strconv.Quote` 包裹（如 `msg="hello world"`）
- 基础类型（String/Int/Uint/Bool/Float/Duration/Time/Error/Binary 等）直接序列化
- 复合类型（Any/Object/Array/Reflect）退化为 JSON 值；`OpenNamespace` 忽略、内部字段平铺
- mainCore 与 errorCore 均使用，`logger.go` 通过 `newKeyValueEncoder()` 创建

grep 检索示例：

```bash
grep 'uid=123' app.log            # 按字段精确匹配，无引号转义
grep 'requestId=abc-123' app.log  # 整条链路追踪
grep ' ERROR ' app.log            # 只看错误
grep '^DATE\[' app.log            # 按日志行前缀过滤
grep 'DATE\[2026-09-23' app.log   # 按天过滤
```

## RequestId 链路追踪（request_id.go）

context 自动注入 requestId，实现整条链路追踪：

- `WithRequestId(ctx, requestId string) context.Context`：写入 requestId（nil ctx 自动退化为 Background）
- `RequestIdFromContext(ctx) string`：读取，不存在返回空串
- 字段 key 固定 `requestId`，值统一 string 类型
- `zap.go` 新增 6 个带 ctx 方法：`DebugCtx`/`InfoCtx`/`WarnCtx`/`ErrorCtx`/`FatalCtx`/`DPanicCtx`，内部自动从 ctx 提取 requestId 追加字段；ctx 无 requestId 时行为等同原方法

用法：

```go
ctx := logger.WithRequestId(r.Context(), requestId)
log.InfoCtx(ctx, "RouteCmd", logger.String("cmd", cmd), logger.Uint64("uid", uid))
```
