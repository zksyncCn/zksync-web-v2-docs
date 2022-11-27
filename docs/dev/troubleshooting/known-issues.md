# 已知的问题

zkSync 2.0 目前处于 alpha 阶段，因此您习惯的某些功能可能无法正常工作。 请记住，该系统仍在持续开发中。

## 为什么 Metamask 原生合约交互不起作用？

目前无法通过 Metamask 与 EIP-1559 交易和 zkSync 智能合约进行交互。 zkSync 不支持 EIP1559 交易。

**解决方案。** 在交易覆盖中明确指定 `{ type: 0 }` ，来使用以太坊旧的交易。

## 为什么我的钱包没有资金，我的合约消失了？

我们预计会不断更新我们的测试网，因此我们需要不时进行重新生成。 这将导致整个状态重置，所有已部署的合约将不复存在。

**我们将在新的事件发生之前告知用户！**

## 为什么`wait()` 会卡在 L1->L2 交易中？

如果 `wait()` 花费的时间比预期的要长得多，则很可能交易已失败。

## 为什么会出现 `unexpected end of JSON input` 编译错误？

这是编译大型智能合约代码库时通常会遇到的错误。

如果遇到此类错误，请执行以下操作：

- 更新 `@matterlabs/hardhat-zksync-solc` 库，然后尝试重新编译智能合约。
- 如果重新编译后出现“ `Library not found` 错误，则应按照 [此处](../../../api/hardhat/compiling-libraries.md) 中的说明进行操作。
- 如果同样的错误仍然存在，请将问题报告给我们的团队。 我们会尽力帮助您。

## 为什么我不能将 CREATE/CREATE2 操作码与原始字节码一起使用？

zkSync 不支持将 CREATE/CREATE2 与原始字节码一起使用。 我们强烈建议使用 `new` 运算符来避免任何问题。

## 为什么 Hardhat 的 `console.log` 不起作用？

zkSync 不支持 Nomic 基金会的 `console.log` 合约。 由于不同的地址规则，即使在部署时，`console.log` 库也可能与以太坊上的地址不同。

<!---

## Metamask native transfers not working

It is not currently possible to transfer ERC-20 tokens inside the Metamask interface.

**Solution.** For now, transfers inside zkSync you should be done via the [zkSync Wallet](https://portal.zksync.io) dApp.


## Transfers with the _entire_ token balance fail

If you try to transfer the entire balance of a token, which is also the token you pay the fee with, the transaction fails. The reason is that we don’t deduct the fee before setting the amount to be transferred.

**Solution.** Keep aside a small amount to cover the fee.

## Errors before sending a transaction

Similar to above, in cases where the fee should be deducted from the token amount, you may get an error if estimate_gas returns an error.

**Solution.** As above, make sure to keep aside a small amount to cover the fee.

## My contract does not compile, due to an error with “cyclic dependencies”

Unfortunately, some contracts have trouble compiling with our hardhat plugin. This is due to the contracts importing external dependencies. This happens to a small number of projects. We are currently working on resolving this issue.

## My transaction is not shown on the block explorer

Currently, the block explorer does not index the latest produced block. As long as a new block is not produced after the block that contains your transaction, it won't appear
on the block explorer.

**Solution.** You can make a simple transfer (or any other transaction) to make the system produce a new block. The previous block would then appear, including your transaction.
Note that if you know the tx id, you can use our wallet to see its status.
--->
