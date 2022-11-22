# 系统合约

为了保持零知识电路尽可能简单，并支持简单的扩容，zkSync 的一大部分逻辑被转移到所谓的“系统合约”——一组具有特殊权限和服务于特殊用途的合约。例如，合约的部署，确保用户只为发布合约的 calldata 支付一次费用等。

系统合约的代码在经过全面测试之前不会公开。本节只会为您提供构建 zkSync 所需的知识。

## 接口

系统合约的地址和接口可以在[这里](https://github.com/matter-labs/v2-testnet-contracts/blob/8de367778f3b7ed7e47ee8233c46c7fe046a75a3/l2/system-contracts/Constants.sol)找到 。

本节将描述一些最常见的系统合约的语义。 

## ContractDeployer

[代码界面](https://github.com/matter-labs/v2-testnet-contracts/blob/8de367778f3b7ed7e47ee8233c46c7fe046a75a3/l2/system-contracts/interfaces/IContractDeployer.sol#L5)

该合约用于部署新的智能合约。它的工作是确保每个部署的合约的字节码是已知的。这个合约还定义了衍生地址。每当一个合约被部署，它就会触发 `ContractDeployed`事件。

未来，我们将添加有关如何直接与此合约进行交互的说明。

## IL1Messenger

[代码界面](https://github.com/matter-labs/v2-testnet-contracts/blob/8de367778f3b7ed7e47ee8233c46c7fe046a75a3/l2/system-contracts/interfaces/IL1Messenger.sol#L5)

该合约用于将消息从 zkSync 发送到以太坊。对于发送的每条消息，都会触发 `L1MessageSent` 事件。

## INonceHolder

[代码界面](https://github.com/matter-labs/v2-testnet-contracts/blob/8de367778f3b7ed7e47ee8233c46c7fe046a75a3/l2/system-contracts/interfaces/INonceHolder.sol#L5)

该合约存储账户随机数。账户 nonce 存储在一个地方以提高效率([tx nonce 和 deployment nonce](./contracts.md#differences-in-create-behaviour) 存储在同一个地方)并且也为了运营者的方便。

## Bootloader

为了获得更大的可扩容性并降低开销，协议的一些部分(例如账户抽象规则)被转移到名为 _bootloader_ 的临时合约中。我们称它为临时的，因为它从未正式部署过，也不能被调用，但它有一个正式的[地址](https://github.com/matter-labs/v2-testnet-contracts/blob/8de367778f3b7ed7e47ee8233c46c7fe046a75a3/l2/system-contracts/Constants.sol#L19)，当它调用其他合约时它会在 `msg.sender` 上使用。

现在，您不需要知道关于它的任何细节，但是在使用帐户抽象特性进行开发时，知道它的存在是很重要的。您总是可以假设 bootloader 不是恶意的，并且它是协议的一部分。将来，bootloader 的代码将被公开，对它的任何更改也将意味着对协议的升级。

## 某些系统合约的受保护访问权限

一些系统合约对账户的影响可能是在以太坊所没有的。例如，在以太坊上，EOA 增加 nonce 的唯一方法是发送交易。同样，发送一个交易每次只能增加 1 nonce。而在 zkSync 上，nonces 是通过 [NonceHolder](#inonceholder) 系统合约执行的。如果不小心执行，用户可以通过调用这个合约来增加他们的 nonces。这就是为什么对 nonceholder 的大多数非视图方法的调用被限制为只能用一个特殊的 `isSystem` 标志来调用，以便与重要的系统合约的交互可以由账户的开发者有意识地管理。

这同样适用于 [ContractDeployer](#contractdeployer) 系统合约。这意味着您需要明确地允许您的用户部署合约，就像[执行 DefaultAccount](https://github.com/matter-labs/v2-testnet-contracts/blob/3f4b6f906c649671022794ecb5cfc1151c278d93/l2/system-contracts/DefaultAccount.sol#L88) 一样。
