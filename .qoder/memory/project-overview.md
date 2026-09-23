# 项目概述

## 基本信息

- **项目类型**：Go 库（library），非二进制程序，无 `main` 包，供其他项目 import
- **模块路径**：`github.com/go-meridian/logger`
- **核心定位**：封装 `go.uber.org/zap`，提供自定义 `LogWriter` 文件轮转 + 字段构造函数导出 + `Logger` 封装类型，消费方无需额外引入 zap 包
- **主要消费者**：`lobby` 项目（`D:/self/go/web/Lobby/`），通过 `go.mod` 的 `replace` 指令引用本地路径

## 技术栈

- Go 1.22.0
- `go.uber.org/zap` v1.27.0（核心依赖）
- `go.uber.org/multierr` v1.11.0（间接依赖）

## 目录结构

| 文件 | 职责 |
|------|------|
| `config.go` | Config 结构体（含 ErrorFile 错误日志前缀）和默认配置 |
| `writer.go` | LogWriter：文件轮转、日期/大小双维度、旧文件清理 |
| `logger.go` | Init/Get/Close，全局 zap.Logger 管理（atomic.Value + sync.Once 幂等） |
| `field.go` | zap 字段构造函数的包级导出 |
| `zap.go` | Logger 封装类型，常用日志方法 + 6 个带 ctx 的 `*Ctx` 方法 |
| `encoder.go` | 自定义 key=value 编码器，输出 DATE[时间] 级别 消息 key=value 格式 |
| `request_id.go` | RequestId 上下文注入与读取，供 Ctx 日志方法自动附加 |
