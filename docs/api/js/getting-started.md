# zkSync 功能

虽然 zkSync 主要与 Web3 兼容，但与以太坊相比有一些差异。 其中主要有：

- 帐户抽象支持（帐户可能具有近乎任意的验证逻辑，并且还启用了 paymaster 支持）。
- 部署交易需要在单独的字段中传递合约的字节码。
- 收费系统有些不同。

这些要求我们使用新的自定义字段扩展标准以太坊交易。 此类扩展交易称为 EIP712 交易，因为 [EIP712](https://eips.ethereum.org/EIPS/eip-712) 用于对它们进行签名。 您可以在 [此处](../api.md#eip712) 查看 EIP712 交易的内部结构。

本文档将专注于如何将这些参数传递给 SDK。

## 覆盖

`ethers` 有一个覆盖的概念。 对于任何链上交易，`ethers` 会在后台找到最佳的`gasPrice`、`gasLimit`、`nonce`和其他重要字段。 但有时，您可能需要明确提供这些值（例如，您想要设置较小的 `gasPrice`，或使用未来的 `nonce` 签署交易）。

在这种情况下，您可以提供一个`Overrides` 对象作为最后一个参数。 在那里，您可以提供诸如 `gasPrice`、`gasLimit`、`nonce` 等字段。

为了使 SDK 尽可能灵活，库使用覆盖来提供 zkSync 特定的字段。 要提供 zkSync 特定的字段，您需要传递以下覆盖：

```typescript
{
    customData: {
        ergsPerPubdata?: BigNumberish;
        factoryDeps?: BytesLike[];
        customSignature?: BytesLike;
        paymasterParams?: {
            paymaster: Address;
            paymasterInput: BytesLike;
        };
    }
}
```

例子：

重写以部署带有字节码`0xcde...12`的合约，并强制运营商不会对第 1 层上的每个已发布字节收取超过`100`ergs 的费用：

```typescript
{
    customData: {
        ergsPerPubdata: "100",
        factoryDeps: ["0xcde...12"],
    }
}
```

为帐户使用自定义签名“0x123456”，同时使用地址为`0x8e1DC7E4Bb15927E76a854a92Bf8053761501fdC`的paymaster和paymaster，输入`0x8c5a3445`：

```typescript
{
    customData: {
        customSignature: "0x123456",
        paymasterParams: {
            paymaster: "0x8e1DC7E4Bb15927E76a854a92Bf8053761501fdC",
            paymasterInput: "0x8c5a3445"
        }
    }
}
```

## 编码 paymaster 参数

虽然 paymaster 功能本身不会对 paymasterInput 的值施加任何限制，但 Matter Labs 团队认可某些类型的 [paymaster flows](../../dev/developer-guides/aa.md#built- in-paymaster-flows) 可由 EOA 处理。

zkSync SDK 提供了一个实用方法，可用于获取正确格式的 `paymasterParams` 对象：[getPaymasterParams](./utils.md#encoding-paymaster-params)。

## 实际操作

如果你想调用名为 greeter 的 ethers `Contract` 对象的 `setGreeting` ，你可以使用下面的方法，同时使用 [testnet paymaster](../../dev/developer-guides /aa.md#testnet-paymaster):

```javascript
// The `setGreeting` method has a single parameter -- new greeting
// We will set its value as `a new greeting`.
const greeting = "a new greeting";
const tx = await greeter.populateTransaction.setGreeting(greeting);
const gasPrice = await sender.provider.getGasPrice();
const gasLimit = await greeter.estimateGas.setGreeting(greeting);
const fee = gasPrice.mul(gasLimit);

const paymasterParams = utils.getPaymasterParams(testnetPaymaster, {
    type: 'ApprovalBased',
    token,
    minimalAllowance: fee,
    innerInput: new Uint8Array()
});
const sentTx = await sender.sendTransaction({
    ...tx,
    maxFeePerGas: gasPrice,
    maxPriorityFeePerGas: BigNumber.from(0),
    gasLimit,
    customData: {
        ergsPerPubdata: utils.DEFAULT_ERGS_PER_PUBDATA_LIMIT,
        paymasterParams
    }
});
```

您还可以在成熟的迷你 dApp 上查看我们的[教程](../../dev/developer-guides/hello-world.md)，用户可以选择代币来支付费用。
