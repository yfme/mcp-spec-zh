---
title: 授权
type: docs
weight: 15
---

{{< callout type="info" >}} **协议版本**：草案 {{< /callout >}}

## 1. 介绍

### 1.1 目的和范围

模型上下文协议在传输层提供授权功能，使 MCP 客户端能够代表资源所有者向受限的 MCP 服务器发出请求。本规范定义了 HTTP+SSE 传输的授权流程。

### 1.2 协议要求

授权对 MCP 实现来说是**可选的**。当支持时：

- 使用 HTTP+SSE 传输的实现**应该**符合本规范。
- 使用 STDIO 传输的实现**不应该**遵循本规范，而应该从环境中获取凭据。
- 使用替代传输的实现**必须**遵循其协议的既定安全最佳实践。

### 1.3 标准合规性

此授权机制基于下列既定规范，但实现了它们的特定子集，以确保安全性和互操作性，同时保持简单性：

- [OAuth 2.1 IETF 草案](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12)
- OAuth 2.0 授权服务器元数据 ([RFC8414](https://datatracker.ietf.org/doc/html/rfc8414))
- OAuth 2.0 动态客户端注册协议 ([RFC7591](https://datatracker.ietf.org/doc/html/rfc7591))

## 2. 授权流程

### 2.1 概述

1. MCP 授权实现**必须**实现 OAuth 2.1，并为机密客户端和公共客户端提供适当的安全措施。

2. MCP 授权实现**应该**支持 OAuth 2.0 动态客户端注册协议 ([RFC7591](https://datatracker.ietf.org/doc/html/rfc7591))。

3. MCP 服务器**应该**而 MCP 客户端**必须**实现 OAuth 2.0 授权服务器元数据 ([RFC8414](https://datatracker.ietf.org/doc/html/rfc8414))。不支持授权服务器元数据的服务器**必须**遵循默认 URI 模式。

### 2.2 基本 OAuth 2.1 授权

当需要授权且客户端尚未证明授权时，服务器**必须**响应 _HTTP 401 未授权_。

客户端在收到 _HTTP 401 未授权_ 后启动 [OAuth 2.1 IETF 草案](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12) 授权流程。

以下演示了使用 PKCE 的公共客户端的基本 OAuth 2.1。

```mermaid
sequenceDiagram
    participant B as 用户代理（浏览器）
    participant C as 客户端
    participant M as MCP 服务器

    C->>M: MCP 请求
    M->>C: HTTP 401 未授权
    Note over C: 生成 code_verifier 和 code_challenge
    C->>B: 使用授权 URL + code_challenge 打开浏览器
    B->>M: GET /authorize
    Note over M: 用户登录并授权
    M->>B: 重定向到回调 URL 并携带授权码
    B->>C: 带授权码的回调
    C->>M: 使用授权码 + code_verifier 的令牌请求
    M->>C: 访问令牌（+ 刷新令牌）
    C->>M: 带访问令牌的 MCP 请求
    Note over C,M: 开始标准 MCP 消息交换
```

### 2.3 服务器元数据发现

对于服务器能力发现：

- MCP 客户端_必须_遵循 [RFC8414](https://datatracker.ietf.org/doc/html/rfc8414) 中定义的 OAuth 2.0 授权服务器元数据协议。
- MCP 服务器_应该_遵循 OAuth 2.0 授权服务器元数据协议。
- 不支持 OAuth 2.0 授权服务器元数据协议的 MCP 服务器_必须_支持回退 URL。

发现流程如下图所示：

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as 服务器

    C->>S: GET /.well-known/oauth-authorization-server
    alt 发现成功
        S->>C: 200 OK + 元数据文档
        Note over C: 使用元数据中的端点
    else 发现失败
        S->>C: 404 未找到
        Note over C: 回退到默认端点
    end
    Note over C: 继续授权流程
```

#### 2.3.1 服务器元数据发现头部

MCP 客户端_应该_在服务器元数据发现期间包含头部 `MCP-Protocol-Version: <protocol-version>`，以允许 MCP 服务器根据 MCP 协议版本进行响应。

例如：`MCP-Protocol-Version: 2024-11-05`

#### 2.3.2 授权基础 URL

授权基础 URL **必须**通过丢弃任何现有的 `path` 组件从 [SSE 端点]({{< ref "specification/draft/basic/transports#http-with-sse" >}}) URL 确定。例如：

如果 SSE 端点是 `https://api.example.com/v1/sse`，那么：

- 授权基础 URL 是 `https://api.example.com`
- 元数据端点**必须**位于 `https://api.example.com/.well-known/oauth-authorization-server`

这确保了授权端点始终位于提供 SSE 端点的域的根级别，无论 SSE 端点 URL 中有任何路径组件。

#### 2.3.3 没有元数据发现的服务器的回退

对于不实现 OAuth 2.0 授权服务器元数据的服务器，客户端**必须**使用以下相对于授权基础 URL（如[第 2.3.2 节]({{< ref "specification/draft/basic/authorization#232-authorization-base-url" >}})中定义）的默认端点路径：

| 端点 | 默认路径 | 描述 |
| ---- | -------- | ---- |
| 授权端点 | /authorize | 用于授权请求 |
| 令牌端点 | /token | 用于令牌交换和刷新 |
| 注册端点 | /register | 用于动态客户端注册 |

例如，对于 SSE 端点 `https://api.example.com/v1/sse`，默认端点将是：

- `https://api.example.com/authorize`
- `https://api.example.com/token`
- `https://api.example.com/register`

客户端**必须**首先尝试通过元数据文档发现端点，然后再回退到默认路径。当使用默认路径时，所有其他协议要求保持不变。

### 2.3 动态客户端注册

MCP 客户端和服务器**应该**支持 [OAuth 2.0 动态客户端注册协议](https://datatracker.ietf.org/doc/html/rfc7591)，以允许 MCP 客户端在不需要用户交互的情况下获取 OAuth 客户端 ID。这为客户端提供了一种标准化的方式来自动注册新服务器，这对 MCP 至关重要，因为：

- 客户端无法预先知道所有可能的服务器
- 手动注册会给用户带来摩擦
- 它使无缝连接到新服务器成为可能
- 服务器可以实现自己的注册策略

任何_不_支持动态客户端注册的 MCP 服务器需要提供获取客户端 ID（以及如果适用，客户端密钥）的替代方式。对于这些服务器，MCP 客户端必须：

1. 为该 MCP 服务器硬编码客户端 ID（以及如果适用，客户端密钥），或
2. 向用户呈现 UI，允许他们在自己注册 OAuth 客户端后输入这些详细信息（例如，通过服务器托管的配置界面）。

### 2.4 授权流程步骤

完整的授权流程如下进行：

```mermaid
sequenceDiagram
    participant B as 用户代理（浏览器）
    participant C as 客户端
    participant M as MCP 服务器

    C->>M: GET /.well-known/oauth-authorization-server
    alt 服务器支持发现
        M->>C: 授权服务器元数据
    else 无发现
        M->>C: 404（使用默认端点）
    end

    alt 动态客户端注册
        C->>M: POST /register
        M->>C: 客户端凭据
    end

    Note over C: 生成 PKCE 参数
    C->>B: 使用授权 URL + code_challenge 打开浏览器
    B->>M: 授权请求
    Note over M: 用户授权
    M->>B: 带授权码重定向到回调
    B->>C: 授权码回调
    C->>M: 令牌请求 + code_verifier
    M->>C: 访问令牌（+ 刷新令牌）
    C->>M: 带访问令牌的 API 请求
```

#### 2.4.1 决策流程概述

```mermaid
flowchart TD
    A[开始授权流程] --> B{检查元数据发现}
    B -->|可用| C[使用元数据端点]
    B -->|不可用| D[使用默认端点]

    C --> G{检查注册端点}
    D --> G

    G -->|可用| H[执行动态注册]
    G -->|不可用| I[需要替代注册]

    H --> J[开始 OAuth 流程]
    I --> J

    J --> K[生成 PKCE 参数]
    K --> L[请求授权]
    L --> M[用户授权]
    M --> N[用授权码交换令牌]
    N --> O[使用访问令牌]
```

### 2.5 访问令牌使用

#### 2.5.1 令牌要求

访问令牌处理**必须**符合 [OAuth 2.1 第 5 节](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12#section-5)对资源请求的要求。具体来说：

1. MCP 客户端**必须**使用授权请求头字段 [第 5.1.1 节](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12#section-5.1.1)：

```
Authorization: Bearer <access-token>
```

2. 访问令牌**禁止**包含在 URI 查询字符串中

请求示例：

```http
GET /v1/contexts HTTP/1.1
Host: mcp.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

#### 2.5.2 令牌处理

资源服务器**必须**按照 [第 5.2 节](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12#section-5.2)中所述验证访问令牌。如果验证失败，服务器**必须**按照 [第 5.3 节](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-12#section-5.3)错误处理要求进行响应。无效或过期的令牌**必须**收到 HTTP 401 响应。

### 2.6 安全考虑

**必须**实现以下安全要求：

1. 客户端**必须**按照 OAuth 2.0 最佳实践安全地存储令牌
2. 服务器**应该**强制令牌过期和轮换
3. 所有授权端点**必须**通过 HTTPS 提供服务
4. 服务器**必须**验证重定向 URI 以防止开放重定向漏洞
5. 重定向 URI **必须**是 localhost URL 或 HTTPS URL

### 2.7 错误处理

服务器**必须**为授权错误返回适当的 HTTP 状态码：

| 状态码 | 描述 | 用途 |
| ------ | ---- | ---- |
| 401 | 未授权 | 需要授权或令牌无效 |
| 403 | 禁止 | 无效的作用域或权限不足 |
| 400 | 错误请求 | 格式错误的授权请求 |

### 2.8 实现要求

1. 实现**必须**遵循 OAuth 2.1 安全最佳实践
2. 所有客户端**必须**使用 PKCE
3. **应该**实现令牌轮换以增强安全性
4. 令牌生命周期**应该**根据安全要求进行限制

### 2.9 第三方授权流程

#### 2.9.1 概述

MCP 服务器**可以**通过第三方授权服务器支持委托授权。在此流程中，MCP 服务器既作为 OAuth 客户端（对第三方授权服务器）又作为 OAuth 授权服务器（对 MCP 客户端）。

#### 2.9.2 流程描述

第三方授权流程包括以下步骤：

1. MCP 客户端与 MCP 服务器启动标准 OAuth 流程
2. MCP 服务器将用户重定向到第三方授权服务器
3. 用户向第三方服务器授权
4. 第三方服务器带授权码重定向回 MCP 服务器
5. MCP 服务器用授权码交换第三方访问令牌
6. MCP 服务器生成自己的访问令牌，绑定到第三方会话
7. MCP 服务器完成与 MCP 客户端的原始 OAuth 流程

```mermaid
sequenceDiagram
    participant B as 用户代理（浏览器）
    participant C as MCP 客户端
    participant M as MCP 服务器
    participant T as 第三方授权服务器

    C->>M: 初始 OAuth 请求
    M->>B: 重定向到第三方 /authorize
    B->>T: 授权请求
    Note over T: 用户授权
    T->>B: 重定向到 MCP 服务器回调
    B->>M: 授权码
    M->>T: 用授权码交换令牌
    T->>M: 第三方访问令牌
    Note over M: 生成绑定的 MCP 令牌
    M->>B: 重定向到 MCP 客户端回调
    B->>C: MCP 授权码
    C->>M: 用授权码交换令牌
    M->>C: MCP 访问令牌
```

#### 2.9.3 会话绑定要求

实现第三方授权的 MCP 服务器**必须**：

1. 维护第三方令牌和已发布 MCP 令牌之间的安全映射
2. 在接受 MCP 令牌之前验证第三方令牌状态
3. 实现适当的令牌生命周期管理
4. 处理第三方令牌过期和更新

#### 2.9.4 安全考虑

在实现第三方授权时，服务器**必须**：

1. 验证所有重定向 URI
2. 安全地存储第三方凭据
3. 实现适当的会话超时处理
4. 考虑令牌链接的安全影响
5. 为第三方认证失败实现适当的错误处理

## 3. 最佳实践

#### 3.1 作为公共 OAuth 2.1 客户端的本地客户端

我们强烈建议本地客户端作为公共客户端实现 OAuth 2.1：

1. 利用授权码挑战（PKCE）进行授权请求，以防止拦截攻击
2. 实现适合本地系统的安全令牌存储
3. 遵循令牌刷新最佳实践以维持会话
4. 适当处理令牌过期和更新

#### 3.2 授权元数据发现

我们强烈建议所有客户端实现元数据发现。这减少了用户手动提供端点或客户端回退到定义的默认值的需要。

#### 3.3 动态客户端注册

由于客户端不预先知道 MCP 服务器集，我们强烈建议实现动态客户端注册。这允许应用程序自动注册到 MCP 服务器，并消除了用户手动获取客户端 ID 的需要。
