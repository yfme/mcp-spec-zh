# 为模型上下文协议做贡献

感谢您有兴趣为模型上下文协议规范做出贡献！本文档概述了如何为此项目做贡献。

## 先决条件

开发规范需要以下软件：

- Node.js 20 或更高版本
- TypeScript
- TypeScript JSON Schema（用于生成 JSON schema）
- [Hugo](https://gohugo.io/)（可选，用于文档）
- Go（可选，用于文档）
- nvm（可选，用于管理 Node 版本）

## 入门指南

1. Fork 本仓库
2. 克隆您的 fork：

```bash
git clone https://github.com/YOUR-USERNAME/specification.git
cd specification
```

3. 安装依赖：

```bash
nvm install  # 安装正确的 Node 版本
npm install  # 安装依赖
```

## 做出更改

请注意，模式更改应在 `schema.ts` 中进行。`schema.json` 是使用 `npm run validate:schema` 从 `schema.ts` 生成的。

1. 创建新分支：

```bash
git checkout -b feature/your-feature-name
```

2. 进行更改
3. 验证您的更改：

```bash
npm run validate:schema    # 验证模式
npm run generate:json     # 生成 JSON schema
```

4. 在本地运行文档（可选）：

```bash
npm run serve:docs
```

## 提交更改

1. 将更改推送到您的 fork
2. 向主仓库提交拉取请求
3. 遵循拉取请求模板
4. 等待审核

## 行为准则

本项目遵循行为准则。请在 [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) 中查看。

## 问题咨询

如有疑问，请在仓库中创建一个 issue。

## 许可证

通过贡献，您同意您的贡献将在 MIT 许可证下授权。

## 安全

请查看我们的[安全政策](SECURITY.md)以报告安全问题。