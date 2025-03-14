---
title: 规范（草案）
cascade:
  type: docs
breadcrumbs: false
weight: 9
aliases:
  - /draft
---

{{< callout type="info" >}} **协议版本**：草案 {{< /callout >}}

[模型上下文协议](https://modelcontextprotocol.io)（MCP）是一个开放协议，可实现 LLM 应用程序与外部数据源和工具之间的无缝集成。无论您是在构建 AI 驱动的 IDE、增强聊天界面，还是创建自定义 AI 工作流，MCP 都提供了一种标准化方式来连接 LLM 与它们所需的上下文。

本规范定义了权威的协议要求，基于 [schema.ts](https://github.com/modelcontextprotocol/specification/blob/main/schema/draft/schema.ts) 中的 TypeScript 模式。

有关实现指南和示例，请访问 [modelcontextprotocol.io](https://modelcontextprotocol.io)。

本文档中的关键词"必须（MUST）"、"禁止（MUST NOT）"、"必需（REQUIRED）"、"应当（SHALL）"、"不应（SHALL NOT）"、"应该（SHOULD）"、"不应该（SHOULD NOT）"、"推荐（RECOMMENDED）"、"不推荐（NOT RECOMMENDED）"、"可以（MAY）"和"可选（OPTIONAL）"应按照 [BCP 14](https://datatracker.ietf.org/doc/html/bcp14) [[RFC2119](https://datatracker.ietf.org/doc/html/rfc2119)] [[RFC8174](https://datatracker.ietf.org/doc/html/rfc8174)] 中所述进行解释，且仅当这些词汇全部大写出现时，如此处所示。

## 概述

MCP 为应用程序提供了一种标准化方式来：

- 与语言模型共享上下文信息
- 向 AI 系统公开工具和功能
- 构建可组合的集成和工作流

该协议使用 [JSON-RPC](https://www.jsonrpc.org/) 2.0 消息在以下各方之间建立通信：

- **主机**：发起连接的 LLM 应用程序
- **客户端**：主机应用程序内的连接器
- **服务器**：提供上下文和功能的服务

MCP 部分受到 [语言服务器协议](https://microsoft.github.io/language-server-protocol/) 的启发，该协议标准化了如何在整个开发工具生态系统中添加对编程语言的支持。类似地，MCP 标准化了如何将额外的上下文和工具集成到 AI 应用程序的生态系统中。

## 关键细节

### 基础协议

- [JSON-RPC](https://www.jsonrpc.org/) 消息格式
- 有状态连接
- 服务器和客户端能力协商

### 功能

服务器向客户端提供以下任何功能：

- **资源**：供用户或 AI 模型使用的上下文和数据
- **提示**：用户的模板化消息和工作流
- **工具**：AI 模型可执行的函数

客户端可以向服务器提供以下功能：

- **采样**：服务器发起的智能行为和递归 LLM 交互

### 附加实用程序

- 配置
- 进度跟踪
- 取消
- 错误报告
- 日志记录

## 安全和信任与安全

模型上下文协议通过任意数据访问和代码执行路径启用强大的功能。随着这种能力的增强，所有实现者必须认真解决重要的安全和信任考虑因素。

### 关键原则

1. **用户同意和控制**

   - 用户必须明确同意并理解所有数据访问和操作
   - 用户必须保留对共享哪些数据和采取哪些操作的控制权
   - 实现者应提供清晰的 UI 用于审查和授权活动

2. **数据隐私**

   - 主机必须在向服务器公开用户数据之前获得用户的明确同意
   - 未经用户同意，主机不得将资源数据传输到其他地方
   - 用户数据应受到适当访问控制的保护

3. **工具安全**

   - 工具代表任意代码执行，必须谨慎对待
   - 主机在调用任何工具之前必须获得用户的明确同意
   - 用户应在授权使用每个工具之前了解其功能

4. **LLM 采样控制**
   - 用户必须明确批准任何 LLM 采样请求
   - 用户应控制：
     - 是否进行采样
     - 将发送的实际提示
     - 服务器可以看到的结果
   - 该协议有意限制了服务器对提示的可见性

### 实施指南

虽然 MCP 本身无法在协议级别强制执行这些安全原则，但实现者**应该**：

1. 在其应用程序中构建健壮的同意和授权流程
2. 提供清晰的安全影响文档
3. 实施适当的访问控制和数据保护
4. 在其集成中遵循安全最佳实践
5. 在其功能设计中考虑隐私影响

## 了解更多

探索每个协议组件的详细规范：

{{< cards >}} {{< card link="architecture" title="架构" icon="template" >}}
{{< card link="basic" title="基础协议" icon="code" >}}
{{< card link="server" title="服务器功能" icon="server" >}}
{{< card link="client" title="客户端功能" icon="user" >}}
{{< card link="contributing" title="贡献" icon="pencil" >}} {{< /cards >}}
