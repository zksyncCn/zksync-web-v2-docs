# 合约

`zksync-web3` 没有实现任何新的 `Contract` 类，因为 `ethers.Contract` 完全开箱即用。 但是，为方便起见，库仍然重新导出此类。

由于在 zkSync 上部署智能合约与在以太坊上部署有一些不同，因此需要一个特定的`ContractFactory`方法。 它支持与 ethers.ContractFactory 相同的接口。

为了支付 ERC20 代币中的智能合约交互，应该使用 `customData` 覆盖。 您可以在 [下一章](./features.md) 中阅读有关访问 zkSync 功能的更多信息。
