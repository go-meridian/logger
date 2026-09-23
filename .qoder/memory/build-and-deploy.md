# 构建与部署

## 依赖

```go
// go.mod
module github.com/go-meridian/logger

go 1.22.0

require go.uber.org/zap v1.27.0

require go.uber.org/multierr v1.11.0 // indirect
```

## 验证命令

```bash
go build ./...
go vet ./...
```

## 本地引用方式（Lobby）

Lobby 通过 `go.mod` 的 `replace` 指令引用本地路径：

```
replace github.com/go-meridian/logger => D:/self/go/web/logger
```

## 初始化示例

```go
import "github.com/go-meridian/logger"

_, err := logger.Init(logCfg)
if err != nil {
    panic("logger.Init error: " + err.Error())
}
defer logger.Close()

log := logger.L()
```

## 兼容性注意

- `mq` 库（`github.com/go-meridian/mq`）通过 `logger.Get()` 获取 `*zap.Logger`，修改 `Get()` 签名需同步更新该库
- 修改 Config 结构体需考虑向后兼容，Lobby 不一定使用所有字段
