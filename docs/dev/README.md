# zkSync 开发者手册

本文档旨在帮助您在 zkSync 上进行开发。 
本文档中对 zkSync 的概念、网络堆栈以及针对一些复杂应用程序和用例的开发进行了说明。

鉴于此文档是开源的，您可以随意提出新主题、添加新内容，并提供您认为有用的示例。如果您不知道从何入手，[请按照这些说明](./troubleshooting/docs-contribution/docs.md)进行操作。

## 基础内容

如果这是您第一次使用 zkSync，我们建议您仔细阅读本文档。

- [rollups 介绍](./fundamentals/rollups.md) - rollups 的简要概述。
- [zkSync 概述](./fundamentals/zkSync.md) - zkSync 技术的简要概述。
- [zkSync 测试网](./fundamentals/testnet.md) - zkSync 测试网的简介。

### 开发者指南

- [快速入门](./developer-guides/hello-world.md) - 学习如何使用 zkSync 开发工具构建一个完整的 dApp。
- [合约部署](./developer-guides/contracts/contracts.md) - 关于如何在 zkSync 上部署智能合约的指南。
  - [智能合约验证](../api/tools/block-explorer/contract-verification.md) - 关于如何使用 zkSync 区块浏览器验证智能合约的指南。
- [系统合约](./developer-guides/contracts/system-contracts.md) - zkSync 系统合约的简介。
- [交易](./developer-guides/transactions/transactions.md) -关于 zkSync 如何处理交易的说明。
  - [区块](./developer-guides/transactions/blocks.md) - 了解区块如何在 zkSync 上工作的说明。
  - [费用机制](./developer-guides/transactions/fee-model.md) - zkSync 费用结构的简介。
- [账户抽象](./developer-guides/aa.md) - 了解账户抽象。
- [桥接资产](./developer-guides/bridging/bridging-asset.md) - 关于代币桥接的简介。
  - [L1 / L2 互操作性](./developer-guides/bridging/l1-l2-interop.md) - L1 和 L2 之间数据通信的简要介绍。
    - [L1 向 L2 发送](./developer-guides/bridging/l1-l2.md) - 了解如何从以太坊发送数据到 zkSync。
    - [L2 向 L1 发送](./developer-guides/bridging/l2-l1.md) - 了解如何从 zkSync 发送数据到以太坊。
- [重要链接](./troubleshooting/important-links.md) - 通过这里快速找到重要的链接。
- [状态信息](./troubleshooting/status.md) - 获取我们正在进行的工作的最新信息。
- [常见问题解答](./troubleshooting/known-issues.md) - 获取您可能发现的常见问题的答案。

### 开发者工具

- [zkSync 2.0 门户网站](https://portal.zksync.io) - 通过这里获取钱包、跨链桥以及水龙头。
- [区块浏览器](../api/tools/block-explorer/) - 在 zkSync 区块浏览器上搜索关于区块、交易、地址等实时和历史信息。

### 示例和教程

- [跨链治理](./tutorials/cross-chain-tutorial.md) - 了解如何使用 L1 与 L2 间的合约交互。
- [账户抽象](./tutorials/custom-aa-tutorial.md) - 了解如何部署您的自定义账户并与 zkSync 系统合约进行交互。
- [构建自定义的支付系统](./tutorials/custom-paymaster-tutorial.md) - 了解如何构建一个自定义的支付系统，让用户在使用您发行的代币进行支付。
