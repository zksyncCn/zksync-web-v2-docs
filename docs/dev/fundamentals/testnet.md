# zkSync 测试网

欢迎来到 zkSync 2.0 测试网！我们的团队很高兴看到您可以在 zkSync 上构建，同时很高兴您提供任何反馈！

::: 请注意这是 Alpha 版本的测试网！

注意，该系统仍然在开发中，这意味着：

- **将来可能会发生重大变化。**
- **一些更新可能需要重置**，即删除所有的余额和智能合约，并重新启动区块链。我们将确保事先沟通所有的重置事项！ 请务必关注我们的 [Discord](https://discord.gg/px2aR7w)。

:::

zkSync 2.0 的直观体验：

- 访问 [zkSync 2.0 门户](https://portal.zksync.io)。
- 通过水龙头获取或从以太坊 Görli 测试网存入一些测试代币。
- 体验转账功能。

该门户网站是用户和开发人员等进入 zkSync 2.0 生态系统的中央入口点。它包含指向所有相关资源的链接，例如[区块浏览器](https://explorer.zksync.io)或特色 dApp 目录。

## 我需要有 zkSync 1.x 的经验吗？

zkSync 1.x 的一些经验将有助于理解一些核心概念，例如最终确定性如何是如何工作的。从其他方面来看，zkSync 2.0 和 zkSync 1.x 有很大的不同，所以在 zkSync 2.0 上构建不需要使用后者的经验。

## 如何开始构建？

现有的所有用于以太坊的 SDK 都可以直接使用，您的用户将获得与以太坊相同的体验。如果您想启用高级的 zkSync 功能，比如账户抽象，请使用 zkSync SDK。

唯一需要使用 zkSync SDK 的地方是在合约部署期间。这可以通过我们的 hardhat 插件轻松完成。

## zkSync 快速入门

请看我们的分步[快速入门指南](../developer-guides/hello-world.md)，在那里您将学到：

- 如何安装 zkSync hardhat 插件并使用它部署智能合约。
- 如何使用 `zksync-web3` 库为您的 dApp 构建前端。

## 连接 Metamask 钱包

如需将 Metamask 连接到 zkSync，请在钱包中添加 zkSync alpha testnet 网络。

1. 打开 Metamask 钱包并单击顶部中心的网络选项：

![img](../../assets/images/connect-1.png)

2. 点击 "手动添加网络"。

3. 填写关于 zkSync alpha testnet 网络的详细信息，并单击“保存”：

- 网络名称：`zkSync alpha testnet`
- 新的 RPC 地址：`https://zksync2-testnet.zksync.dev`
- 链 ID：`280`
- 货币符号：`ETH`
<!-- - Block Explorer URL: `https://explorer.zksync.io` -->
