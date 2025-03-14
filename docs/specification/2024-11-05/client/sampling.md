---
title: 采样
type: docs
weight: 40
---

{{< callout type="info" >}} **协议版本**：{{< param protocolRevision >}}
{{< /callout >}}

模型上下文协议（MCP）为服务器通过客户端请求 LLM 采样（"完成"或"生成"）提供了标准化方式。这种流程允许客户端保持对模型访问、选择和权限的控制，同时使服务器能够利用 AI 能力&mdash;无需服务器 API 密钥。服务器可以请求基于文本或图像的交互，并可选择在其提示中包含来自 MCP 服务器的上下文。

## 用户交互模型

MCP 中的采样允许服务器实现智能行为，通过使 LLM 调用在其他 MCP 服务器功能内_嵌套_发生。

实现可以通过适合其需求的任何界面模式公开采样&mdash;协议本身不要求任何特定的用户交互模型。

{{< callout type="warning" >}} 出于信任、安全和安全考虑，**应该**始终有一个人在循环中，能够拒绝采样请求。

应用程序**应该**：

- 提供使审核采样请求变得简单直观的 UI
- 允许用户在发送前查看和编辑提示
- 在交付前呈现生成的响应以供审核 {{< /callout >}}

## 能力

支持采样的客户端**必须**在[初始化]({{< ref "/specification/2024-11-05/basic/lifecycle#initialization" >}})期间声明 `sampling` 能力：

```json
{
  "capabilities": {
    "sampling": {}
  }
}
```

## 协议消息

### 创建消息

要请求语言模型生成，服务器发送 `sampling/createMessage` 请求：

**请求：**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "sampling/createMessage",
  "params": {
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "法国的首都是什么？"
        }
      }
    ],
    "modelPreferences": {
      "hints": [
        {
          "name": "claude-3-sonnet"
        }
      ],
      "intelligencePriority": 0.8,
      "speedPriority": 0.5
    },
    "systemPrompt": "你是一个有帮助的助手。",
    "maxTokens": 100
  }
}
```

**响应：**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "role": "assistant",
    "content": {
      "type": "text",
      "text": "法国的首都是巴黎。"
    },
    "model": "claude-3-sonnet-20240307",
    "stopReason": "endTurn"
  }
}
```

## 消息流

```mermaid
sequenceDiagram
    participant Server
    participant Client
    participant User
    participant LLM

    Note over Server,Client: 服务器启动采样
    Server->>Client: sampling/createMessage

    Note over Client,User: 人在循环中审核
    Client->>User: 呈现请求供批准
    User-->>Client: 审核并批准/修改

    Note over Client,LLM: 模型交互
    Client->>LLM: 转发已批准的请求
    LLM-->>Client: 返回生成

    Note over Client,User: 响应审核
    Client->>User: 呈现响应供批准
    User-->>Client: 审核并批准/修改

    Note over Server,Client: 完成请求
    Client-->>Server: 返回已批准的响应
```

## 数据类型

### 消息

采样消息可以包含：

#### 文本内容

```json
{
  "type": "text",
  "text": "消息内容"
}
```

#### 图像内容

```json
{
  "type": "image",
  "data": "base64-编码的图像数据",
  "mimeType": "image/jpeg"
}
```

### 模型偏好

MCP 中的模型选择需要仔细抽象，因为服务器和客户端可能使用不同的 AI 提供商，具有不同的模型产品。服务器不能简单地通过名称请求特定模型，因为客户端可能无法访问该确切模型，或者可能更喜欢使用不同提供商的等效模型。

为了解决这个问题，MCP 实现了一个偏好系统，结合了抽象能力优先级和可选模型提示：

#### 能力优先级

服务器通过三个归一化优先级值（0-1）表达其需求：

- `costPriority`：降低成本有多重要？较高的值倾向于更便宜的模型。
- `speedPriority`：低延迟有多重要？较高的值倾向于更快的模型。
- `intelligencePriority`：高级能力有多重要？较高的值倾向于更强大的模型。

#### 模型提示

虽然优先级有助于基于特性选择模型，但 `hints` 允许服务器建议特定模型或模型系列：

- 提示被视为可以灵活匹配模型名称的子字符串
- 多个提示按优先顺序评估
- 客户端**可以**将提示映射到来自不同提供商的等效模型
- 提示是建议性的&mdash;客户端做出最终模型选择

例如：

```json
{
  "hints": [
    { "name": "claude-3-sonnet" }, // 首选 Sonnet 类模型
    { "name": "claude" } // 回退到任何 Claude 模型
  ],
  "costPriority": 0.3, // 成本不太重要
  "speedPriority": 0.8, // 速度非常重要
  "intelligencePriority": 0.5 // 中等能力需求
}
```

客户端处理这些偏好，从其可用选项中选择适当的模型。例如，如果客户端无法访问 Claude 模型但有 Gemini，它可能会根据类似的能力将 sonnet 提示映射到 `gemini-1.5-pro`。

## 错误处理

客户端**应该**为常见失败情况返回错误：

错误示例：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -1,
    "message": "用户拒绝采样请求"
  }
}
```

## 安全考虑

1. 客户端**应该**实现用户批准控制
2. 双方**应该**验证消息内容
3. 客户端**应该**尊重模型偏好提示
4. 客户端**应该**实现速率限制
5. 双方**必须**适当处理敏感数据