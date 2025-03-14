---
title: 提示
weight: 10
---

{{< callout type="info" >}} **协议版本**：草案 {{< /callout >}}

模型上下文协议（MCP）为服务器向客户端公开提示模板提供了标准化方式。提示允许服务器提供与语言模型交互的结构化消息和指令。客户端可以发现可用的提示、检索其内容并提供参数来自定义它们。

## 用户交互模型

提示被设计为**用户控制**的，这意味着它们从服务器公开给客户端，目的是让用户能够明确选择它们使用。

通常，提示会通过用户界面中的用户发起命令触发，这允许用户自然地发现和调用可用的提示。

例如，作为斜杠命令：

![作为斜杠命令公开的提示示例](slash-command.png)

然而，实现者可以通过任何适合其需求的界面模式公开提示&mdash;协议本身不要求任何特定的用户交互模型。

## 能力

支持提示的服务器**必须**在[初始化]({{< ref "/specification/draft/basic/lifecycle#initialization" >}})期间声明 `prompts` 能力：

```json
{
  "capabilities": {
    "prompts": {
      "listChanged": true
    }
  }
}
```

`listChanged` 表示服务器是否会在可用提示列表更改时发出通知。

## 协议消息

### 列出提示

要检索可用的提示，客户端发送 `prompts/list` 请求。此操作支持[分页]({{< ref "/specification/draft/server/utilities/pagination" >}})。

**请求：**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "prompts/list",
  "params": {
    "cursor": "可选的光标值"
  }
}
```

**响应：**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "prompts": [
      {
        "name": "code_review",
        "description": "要求 LLM 分析代码质量并提出改进建议",
        "arguments": [
          {
            "name": "code",
            "description": "要审查的代码",
            "required": true
          }
        ]
      }
    ],
    "nextCursor": "下一页光标"
  }
}
```

### 获取提示

要检索特定提示，客户端发送 `prompts/get` 请求。参数可以通过[完成 API]({{< ref "/specification/draft/server/utilities/completion" >}})自动完成。

**请求：**

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "prompts/get",
  "params": {
    "name": "code_review",
    "arguments": {
      "code": "def hello():\n    print('world')"
    }
  }
}
```

**响应：**

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "description": "代码审查提示",
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "请审查这段 Python 代码：\ndef hello():\n    print('world')"
        }
      }
    ]
  }
}
```

### 列表更改通知

当可用提示列表更改时，声明了 `listChanged` 能力的服务器**应该**发送通知：

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/prompts/list_changed"
}
```

## 消息流

```mermaid
sequenceDiagram
    participant Client
    participant Server

    Note over Client,Server: 发现
    Client->>Server: prompts/list
    Server-->>Client: 提示列表

    Note over Client,Server: 使用
    Client->>Server: prompts/get
    Server-->>Client: 提示内容

    opt listChanged
      Note over Client,Server: 变更
      Server--)Client: prompts/list_changed
      Client->>Server: prompts/list
      Server-->>Client: 更新的提示
    end
```

## 数据类型

### 提示

提示定义包括：

- `name`：提示的唯一标识符
- `description`：可选的人类可读描述
- `arguments`：用于自定义的可选参数列表

### PromptMessage

提示中的消息可以包含：

- `role`："user" 或 "assistant" 以指示说话者
- `content`：以下内容类型之一：

#### 文本内容

文本内容表示纯文本消息：

```json
{
  "type": "text",
  "text": "消息的文本内容"
}
```

这是用于自然语言交互的最常见内容类型。

#### 图像内容

图像内容允许在消息中包含视觉信息：

```json
{
  "type": "image",
  "data": "base64-编码的图像数据",
  "mimeType": "image/png"
}
```

图像数据**必须**是 base64 编码的，并包含有效的 MIME 类型。这使得视觉上下文重要的多模态交互成为可能。

#### 音频内容

音频内容允许在消息中包含音频信息：

```json
{
  "type": "audio",
  "data": "base64-编码的音频数据",
  "mimeType": "audio/wav"
}
```

音频数据**必须**是 base64 编码的，并包含有效的 MIME 类型。这使得音频上下文重要的多模态交互成为可能。

#### 嵌入资源

嵌入资源允许在消息中直接引用服务器端资源：

```json
{
  "type": "resource",
  "resource": {
    "uri": "resource://example",
    "mimeType": "text/plain",
    "text": "资源内容"
  }
}
```

资源可以包含文本或二进制（blob）数据，并且**必须**包括：

- 有效的资源 URI
- 适当的 MIME 类型
- 文本内容或 base64 编码的 blob 数据

嵌入资源使提示能够无缝地将服务器管理的内容（如文档、代码示例或其他参考材料）直接合并到对话流中。

## 错误处理

服务器**应该**为常见的失败情况返回标准 JSON-RPC 错误：

- 无效的提示名称：`-32602`（无效参数）
- 缺少必需参数：`-32602`（无效参数）
- 内部错误：`-32603`（内部错误）

## 实现考虑

1. 服务器**应该**在处理前验证提示参数
2. 客户端**应该**处理大型提示列表的分页
3. 双方**应该**尊重能力协商

## 安全

实现**必须**仔细验证所有提示输入和输出，以防止注入攻击或未授权访问资源。
