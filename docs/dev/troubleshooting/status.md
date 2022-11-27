# 功能支持

::: tip 提供反馈

随着我们陆续添加新的功能，此页面会不断更新。

如果其中任何一个阻止了您，请在我们的 [discord](https://discord.gg/px2aR7w) 上告知我们，以便我们可以相应地进行优先排序。

:::

## 使用 Solidity 库

如果使用仅包含 `private` 或 `internal` 的方法可以内联 Solidity 库，那么这个库可以不受任何限制地使用。

但是，如果库至少包含一个`private`或`internal`方法，则该库不再在 Yul 表示形式中内联。 这些地址需要显式传递给我们的编译器。 我们的安全帽插件目前不支持此功能，但以后会添加。

如需对旧版本 Solidity 和 Vyper 的支持，请查看 [这里](../developer-guides/contracts/contracts.md#solidity-vyper-support)。

## 不支持的操作码

- `SELFDESTRUCT` (它被认为是有害的，有人呼吁在 L1 上停用它)。

- `EXTCODECOPY` 如果需要可以实现，但我们暂时跳过它，因为 zkEVM 操作码无论如何都与 EVM 操作码不同)。

- `CALLCODE` (在以太坊上已弃用，有利于`DELEGATECALL`).

- `CODECOPY` - (它不返回 0，但会产生编译错误)。
  
  ## 被编译器忽略

- `PC` 总是返回 `0` ((因为 solidity 是 0.7.0, 所以在 Yul 和 Solidy 中无法访问)。

## 预编译

- zkSync 本身支持 `keccak256`, `sha256`, 以及 `ecrecover` via 预编译。

## ## 目前支持的功能

- **原生支持 ECDSA 签名**。与第一版 zkSync 和大多数 zk rollup 不同，注册用户私钥不需要特殊操作。任何帐户都可以在 L2 上使用与 L1 上使用的相同的私钥进行管理。
- **Solidity 0.8.x 和 Vyper 支持**。无需更改或重新审核代码库。
- **Web3 API**. 除了少数例外，我们的 API 与以太坊完全兼容。这使得与现有索引器、浏览器等的无缝集成成为可能。
- **安全帽插件**。这允许在 zkSync 上轻松测试和开发智能合约。
- **L1 <-> L2 智能合约通信**。

## 即将发布的功能

- **更多开发者工具**。各种 hardhat 插件与 zkSync 插件之间的可组合性、使用 Docker 的简单本地设置等对于生态系统的发展至关重要。
- **支持旧版本的 Solidity**。我们正在积极致力于支持不同版本的 Solidity，以实现现有项目的无缝集成。
- **zkPorter 扩展**。最大和最重要的功能之一。与以太坊相比，zk Rollup 账户的安全性高，费用减少 20 倍，而 zkPorter 账户的安全性远高于侧链，交易成本几乎恒定在几美分。
