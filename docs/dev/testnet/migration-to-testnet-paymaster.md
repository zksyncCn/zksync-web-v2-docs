# 迁移到测试网 paymaster

## 准备工作

在进一步进入本节之前，请确保您已阅读[paymasters](../developer-guides/aa.md#paymasters)。

虽然之前的 zkSync 2.0 测试网迭代原生支持以不同的代币支付费用，但它导致了与以太坊生态系统的几个兼容性问题。随着 [paymasters](../developer-guides/aa.md#paymasters) 的出现，此功能已变得多余，因为现在任何人都可以部署他们的 paymaster 智能合约，将 ERC-20 代币即时交换为 ETH。您可以在 [此处](../tutorials/custom-paymaster-tutorial.md) 阅读有关部署自定义 paymaster 的教程。

为了支持生态系统，zkSync 不打算在主网上部署任何 paymaster。但是，考虑到更好的 DevEx，我们在测试网上部署了一个。 testnet paymaster 能够以 1:1 的汇率使用兼容 ERC-20 的代币支付费用。您可以阅读文档 [此处](../developer-guides/aa.md#testnet-paymaster)。在本节中，我们将展示一个关于从使用 ERC20 代币支付费用的旧方式迁移到新方式的简短示例。

本文档是关于 testnet paymaster *only*。在主网上部署您的项目时，您将需要部署您的 paymaster 或找到第 3 方的付款人并阅读其文档。

## 过去的版本

在之前的测试网版本中，您在交易的覆盖中提供了 `feeToken `，因此以 USDC 支付费用的智能合约调用大致如下所示：

```js
const tx = await contract.callMethod({
    customData: {
        feeToken: USDC_ADDRESS
    }
}) 
```

## 使用测试网paymaster

使用 testnet paymaster 包括三个步骤：

1. 获取测试网 paymaster 的地址。

```js
const testnetPaymaster = await provider.getTestnetPaymasterAddress();
```

注意：不建议缓存paymaster的地址，因为它可能会在没有警告的情况下更改。

2. 编码要在交易中使用paymaster参数， 为此，您可以使用`utils.getPaymasterParams `方法：

```js
import { utils } from 'zksync-web3'

const paymasterParams = utils.getPaymasterParams(testnetPaymaster, {
    type: 'ApprovalBased',
    token: USDC_ADDRESS,
    // Note, that the allowance for the testnet paymaster must be
    // at least maxFeePerErg * gasLimit, where maxFeePerErg and gasLimit
    // are parameters used in the transaction.
    minimalAllowance: maxFeePerErg.mul(gasLimit),
    innerInput: new Uint8Array()
});
```

3. 使用提供的 paymaster 参数发送交易：

```js
const tx = await contract.callMethod({
    customData: {
        paymasterParams
    }
});
```
