---
title: 生命周期
type: docs
weight: 30
---

{{< callout type="info" >}} **协议版本**：草案 {{< /callout >}}

模型上下文协议（MCP）为客户端-服务器连接定义了严格的生命周期，确保正确的能力协商和状态管理。

1. **初始化**：能力协商和协议版本协议
2. **操作**：正常协议通信
3. **关闭**：连接的优雅终止

```mermaid
sequenceDiagram
    participant Client
    participant Server

    Note over Client,Server: 初始化阶段
    activate Client
    Client->>+Server: initialize 请求
    Server-->>Client: initialize 响应
    Client--)Server: initialized 通知

    Note over Client,Server: 操作阶段
    rect rgb(200, 220, 250)
        note over Client,Server: 正常协议操作
    end

    Note over Client,Server: 关闭
    Client--)-Server: 断开连接
    deactivate Server
    Note over Client,Server: 连接关闭
```

## 生命周期阶段

### 初始化

初始化阶段**必须**是客户端和服务器之间的第一次交互。在此阶段，客户端和服务器：

- 建立协议版本兼容性
- 交换和协商能力
- 共享实现详细信息

客户端**必须**通过发送包含以下内容的 `initialize` 请求启动此阶段：

- 支持的协议版本
- 客户端能力
- 客户端实现信息

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "roots": {
        "listChanged": true
      },
      "sampling": {}
    },
    "clientInfo": {
      "name": "ExampleClient",
      "version": "1.0.0"
    }
  }
}
```

服务器**必须**用自己的能力和信息响应：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "logging": {},
      "prompts": {
        "listChanged": true
      },
      "resources": {
        "subscribe": true,
        "listChanged": true
      },
      "tools": {
        "listChanged": true
      }
    },
    "serverInfo": {
      "name": "ExampleServer",
      "version": "1.0.0"
    }
  }
}
```

成功初始化后，客户端**必须**发送 `initialized` 通知，表明它已准备好开始正常操作：

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

- 在服务器响应 `initialize` 请求之前，客户端**不应**发送除 [ping]({{< ref "/specification/draft/basic/utilities/ping" >}}) 以外的请求。
- 在收到 `initialized` 通知之前，服务器**不应**发送除 [ping]({{< ref "/specification/draft/basic/utilities/ping" >}}) 和 [logging]({{< ref "/specification/draft/server/utilities/logging" >}}) 以外的请求。

#### 版本协商

在 `initialize` 请求中，客户端**必须**发送它支持的协议版本。这**应该**是客户端支持的_最新_版本。

如果服务器支持请求的协议版本，它**必须**响应相同的版本。否则，服务器**必须**响应它支持的另一个协议版本。这**应该**是服务器支持的_最新_版本。

如果客户端不支持服务器响应中的版本，它**应该**断开连接。

#### 能力协商

客户端和服务器能力确定会话期间将可用的可选协议功能。

关键能力包括：

| 类别   | 能力           | 描述                                                                                  |
| ------ | -------------- | ------------------------------------------------------------------------------------- |
| 客户端 | `roots`        | 提供文件系统[根目录]({{< ref "/specification/draft/client/roots" >}})的能力     |
| 客户端 | `sampling`     | 支持 LLM [采样]({{< ref "/specification/draft/client/sampling" >}})请求         |
| 客户端 | `experimental` | 描述对非标准实验性功能的支持                                                          |
| 服务器 | `prompts`      | 提供[提示模板]({{< ref "/specification/draft/server/prompts" >}})               |
| 服务器 | `resources`    | 提供可读取的[资源]({{< ref "/specification/draft/server/resources" >}})          |
| 服务器 | `tools`        | 公开可调用的[工具]({{< ref "/specification/draft/server/tools" >}})              |
| 服务器 | `logging`      | 发出结构化[日志消息]({{< ref "/specification/draft/server/utilities/logging" >}})|
| 服务器 | `experimental` | 描述对非标准实验性功能的支持                                                          |

能力对象可以描述子能力，如：

- `listChanged`：支持列表变更通知（适用于提示、资源和工具）
- `subscribe`：支持订阅单个项目的变更（仅适用于资源）

### 操作

在操作阶段，客户端和服务器根据协商的能力交换消息。

双方**应该**：

- 尊重协商的协议版本
- 只使用成功协商的能力

### 关闭

在关闭阶段，一方（通常是客户端）干净地终止协议连接。没有定义特定的关闭消息—相反，应使用底层传输机制来发出连接终止信号：

#### stdio

对于 stdio [传输]({{< ref "/specification/draft/basic/transports" >}})，客户端**应该**通过以下方式启动关闭：

1. 首先，关闭到子进程（服务器）的输入流
2. 等待服务器退出，或者如果服务器在合理时间内未退出，则发送 `SIGTERM`
3. 如果服务器在 `SIGTERM` 后的合理时间内未退出，则发送 `SIGKILL`

服务器**可以**通过关闭其输出流到客户端并退出来启动关闭。

#### HTTP

对于 HTTP [传输]({{< ref "/specification/draft/basic/transports" >}})，关闭通过关闭相关的 HTTP 连接来表示。

## 错误处理

实现**应该**准备处理这些错误情况：

- 协议版本不匹配
- 无法协商所需的能力
- 初始化请求超时
- 关闭超时

实现**应该**为所有请求实现适当的超时，以防止连接挂起和资源耗尽。

初始化错误示例：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "不支持的协议版本",
    "data": {
      "supported": ["2024-11-05"],
      "requested": "1.0.0"
    }
  }
}
```
