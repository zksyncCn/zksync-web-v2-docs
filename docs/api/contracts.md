## L1合约接口

要从 L1 与 zkSync 交互，您需要规范桥的接口。有两种主要方法可以将其导入您的代码库：

- 通过从 `@matterlabs/zksync-contracts`npm 包中导入它。 （首选）
- 从 [repo](https://github.com/matter-labs/v2-testnet-contracts) 下载合约。

可以在 [此处](../dev/developer-guides/bridging/l1-l2.md) 找到有关与 zkSync 规范桥交互的指南以及 Solidity 和 `zksync-web3` SDK 中的示例。

此页面主要用作您可能需要的接口和类型以及如何导入它们的快速参考。

## `@matterlabs/zksync-contracts` 参考

- `@matterlabs/zksync-contracts/contracts/interfaces/IZkSync.sol` 是 zkSync L1 合约接口 `IZkSync` 所在的文件。特别感兴趣的是 `IBridge`功能。它的实现可以在 [这里](https://github.com/matter-labs/v2-testnet-contracts/blob/main/l1/contracts/zksync/interfaces/IZkSync.sol) 找到。

- `@matterlabs/zksync-contracts/libraries/Operations.sol` 是存储桥中所有用户类型的 `Operations` 库文件。它的实现可以在 [这里](https://github.com/matter-labs/v2-testnet-contracts/blob/main/libraries/Operations.sol) 找到。

- 存储库中的代码可能包含一些配置常量。这些是从开发环境中获取的占位符值。您应该仅将库用于它提供的接口和类型。
