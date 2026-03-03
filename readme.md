# Transparent Context SDK

**定位**: 分布式系统中的透明上下文传播库，解决跨服务调用时的元数据透传问题。

## 🚀 核心痛点
在微服务链路中，透传上下文通常面临以下问题：
1. **侵入性强**：需要在每个服务中手动解析和传递 Header。
2. **适配繁琐**：需为不同框架（Gin, gRPC）编写重复的适配代码。
3. **心智负担**：开发者需时刻关注上下文的传递，容易遗漏。

本 SDK 提供**开箱即用**的中间件，自动完成上下文的**提取、注入与传播**，让透传变得“透明”。

## ✨ 功能特性
*   **全链路透传 (`Req-All-*` / `Resp-All-*`)**：数据在整个调用链中自动传递（如 TraceID）。
*   **单跳透传 (`Req-Once-*` / `Resp-Once-*`)**：数据仅在相邻服务间传递（如鉴权 Token、SpanID、CalleeUsn、CalleeFunc）。
*   **多协议支持**：内置 Gin (HTTP) 和 gRPC 中间件。
*   **零侵入 API**：基于 Go标准 `context.Context` 操作。

## 🏗 架构原理
![alt text](image.png)

透传流程分为三个阶段：
1.  **入站 (Inbound)**: 中间件自动从 Header/Metadata 提取数据写入 `ctx`。
2.  **站内 (In-Process)**: 业务逻辑通过 `ctx` 读写透传数据。
3.  **出站 (Outbound)**: 客户端拦截器自动将 `ctx` 中的数据注入下游请求。

## ⚡️ 快速开始

### 1. 安装
```bash
go get github.com/shuaibizhang/transparent-context
```

### 2. 接入指南

#### 服务端 (Server)
只需一行代码注册中间件，即可自动提取透传数据。

**Gin (HTTP)**
```go
import "github.com/shuaibizhang/transparent-context/middleware/httpmiddleware"

r := gin.Default()
r.Use(httpmiddleware.TransparentContextMiddleware()) // 启用中间件
```

**gRPC**
```go
import "github.com/shuaibizhang/transparent-context/middleware/grpcmiddleware"

s := grpc.NewServer(
    grpc.UnaryInterceptor(grpcmiddleware.TransparentContextUnaryServerInterceptor()), // 启用拦截器
)
```

#### 客户端 (Client)
自动将上下文注入到请求头中。

**HTTP Client**
```go
import "github.com/shuaibizhang/transparent-context/middleware/httpmiddleware"

// 发起请求前注入 Header
httpmiddleware.InjectToHttpClientHeader(ctx, req.Header)
client.Do(req)
```

**gRPC Client**
```go
import "github.com/shuaibizhang/transparent-context/middleware/grpcmiddleware"

conn, err := grpc.NewClient("target",
    grpc.WithUnaryInterceptor(grpcmiddleware.TransparentContextUnaryClientInterceptor()), // 启用拦截器
)
```

### 3. 业务使用
在业务代码中，直接通过 `context` 读写数据，无需关心底层协议。

```go
import transparentcontext "github.com/shuaibizhang/transparent-context/context"

func Handler(ctx context.Context) {
    tc := transparentcontext.GetTransparentContext(ctx)
    if tc == nil {
        return
    }
    
    // 1. 读取上游传递的数据
    traceID := tc.GetReqAllByKey("TraceID") 
    from := tc.GetReqOnceByKey("From")

    // 2. 设置传递给下游的数据
    tc.SetReqAllByKey("UserID", "1001")
    
    // 3. 设置返回给上游的数据
    tc.SetRespAllByKey("ProcessTime", "100ms")
}
```

## 🧪 验证 Demo
本项目提供了一个完整的 E2E Demo (`Service A -> Service B -> Service C`) 验证透传能力。

**运行测试**：
```bash
cd demo && go test -v ./e2e_test.go
```

**验证点**：
- ✅ **全链路透传**：Service A 设置的 `Req-All-TraceID` 能被 B 和 C 收到。
- ✅ **单跳透传**：Service A 设置的 `Req-Once-From` 仅由 B 收到，不会传给 C。
- ✅ **响应透传**：Service C 设置的 `Resp-All-TraceID` 能逆向回传给 A。
