---
title: 版本控制
type: docs
weight: 80
---

模型上下文协议使用遵循 `YYYY-MM-DD` 格式的基于字符串的版本标识符，表示最后一次向后不兼容更改的日期。

当前协议版本是 **草案**。[查看所有版本]({{< ref "/specification/draft/revisions" >}})。

{{< callout type="info" >}} 当协议更新时，只要更改保持向后兼容性，协议版本就_不会_递增。这允许在保持互操作性的同时进行增量改进。{{< /callout >}}

版本协商发生在[初始化]({{< ref "/specification/draft/basic/lifecycle#initialization" >}})期间。客户端和服务器**可以**同时支持多个协议版本，但它们**必须**就会话使用的单一版本达成一致。

如果版本协商失败，协议提供适当的错误处理，允许客户端在无法找到与服务器兼容的版本时优雅地终止连接。
