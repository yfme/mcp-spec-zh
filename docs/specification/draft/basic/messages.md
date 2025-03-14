---
title: 消息
type: docs
weight: 20
---

{{< callout type="info" >}} **协议版本**：草案 {{< /callout >}}

MCP 中的所有消息**必须**遵循 [JSON-RPC 2.0](https://www.jsonrpc.org/specification) 规范。该协议定义了三种类型的消息：

## 请求

请求从客户端发送到服务器，或从服务器发送到客户端。

```typescript
{
  jsonrpc: "2.0";
  id: string | number;
  method: string;
  params?: {
    [key: string]: unknown;
  };
}
```

- 请求**必须**包含字符串或整数 ID。
- 与基础 JSON-RPC 不同，ID**不得**为 `null`。
- 请求 ID 在同一会话中**不得**被请求者先前使用过。

## 响应

响应作为对请求的回复发送。

```typescript
{
  jsonrpc: "2.0";
  id: string | number;
  result?: {
    [key: string]: unknown;
  }
  error?: {
    code: number;
    message: string;
    data?: unknown;
  }
}
```

- 响应**必须**包含与其对应的请求相同的 ID。
- **必须**设置 `result` 或 `error`。响应**不得**同时设置两者。
- 错误代码**必须**是整数。

## 通知

通知从客户端发送到服务器，或从服务器发送到客户端。接收者**不得**发送响应。

```typescript
{
  jsonrpc: "2.0";
  method: string;
  params?: {
    [key: string]: unknown;
  };
}
```

- 通知**不得**包含 ID。
