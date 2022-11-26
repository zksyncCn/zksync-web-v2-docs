# 账户抽象支持

## 介绍

在以太坊上有两种类型的账户：[外部账户 (EOA)](https://ethereum.org/en/developers/docs/accounts/#externally-owned-accounts-and-key-pairs) 和 [合约账户](https://ethereum.org/en/developers/docs/accounts/#contract-accounts)。
前一种是唯一可以发起交易的，
而后者是唯一可以实现任意逻辑的。对于某些用例，例如智能合约钱包或隐私协议，这种差异会产生很多变化。
因此，此类应用程序需要 L1 中继器，例如一个 EOA，帮助促进智能合约钱包的交易。

zkSync 2.0 中的账户可以像 EOA 一样发起交易，但也可以在其中实现任意逻辑，例如智能合约。此功能称为“账户抽象”，它旨在解决上述问题。

::: 警告

功能不稳定

这是 zkSync 2.0 上账户抽象（AA）的测试版本。我们很高兴听到您的反馈！请注意：**应该预料到对 AA 所需的 API/接口的重大更改。**

zkSync 2.0 是最早采用 AA 的 EVM 兼容链之一，因此该测试网也用于查看来自 EVM 链的“经典”项目如何与帐户抽象功能共存。

:::

### 先决条件

为了更好地理解此页面，我们建议您先花一些时间阅读关于 [accounts指南](https://ethereum.org/en/developers/docs/accounts/).

## 设计

zkSync 上的账户抽象协议与 [EIP4337](https://eips.ethereum.org/EIPS/eip-4337) 非常相似，但为了效率和更好的用户体验，我们的协议仍然不同。

### 保持随机数唯一

::: 警告 预计会有变化

当前模型有一些重要的缺点：它不允许自定义钱包同时发送多个交易，同时保持确定性的顺序。 这是为了 EOAs 的 nonce 按照预计顺序增长，而对于自定义账户，交易顺序无法确定。

在未来，我们计划切换到一个模型，让账户可以选择他们是否想要顺序随机数排序（与 EOA 相同）或者他们想要任意排序。

:::

每个区块链的重要不变量之一是每个交易都有一个唯一的哈希值。 使用任意帐户抽象来持有此属性并非易事，
尽管帐户通常可以接受多个相同的交易。 即使这些交易在技术上按照区块链的规则是有效的，但违反索引器和其他工具很难处理哈希唯一性。

需要有一个协议级别的解决方案，对用户来说既便宜又在恶意运营商的情况下稳健。 确保交易哈希不重复的最简单方法之一是让一对（发送者、随机数）始终唯一。

使用以下协议：

- 在每笔交易开始之前，系统会查询[NonceHolder](./contracts/system-contracts.md#inonceholder) 来检查提供的nonce是否已经被使用。
- 如果尚未使用 nonce，则运行交易验证。 在此期间，预计提供的 nonce 将被标记为“已使用”。
- 验证后，系统检查此 nonce 现在是否标记为已使用。

用户将被允许使用任何 256 位数字作为 nonce，并且他们可以将任何非零值放在系统合约中的相应键下。 协议已经得到了支持，但不在服务器端。

一旦服务器端的支持发布，将提供有关与“NonceHolder”系统合约的各种交互的更多文档以及教程。 目前，建议仅使用 `incrementNonceIfEquals` 方法，该方法实际上强制随机数的顺序排序。

### 

### 账户接口

建议每个账户实现 [IAccount](https://github.com/matter-labs/v2-testnet-contracts/blob/main/l2/system-contracts/interfaces/IAccount.sol) 接口。它包含以下五种方法：

- `validateTransaction` 是强制性的，系统将使用它来确定 AA 逻辑是否同意继续进行交易。 如果交易不被接受（例如签名错误），该方法应该恢复。 如果该方法调用成功，则认为实现的账户逻辑接受交易，系统将继续交易流程。
- `payForTransaction` 是可选的，如果交易没有付款人，系统将调用它，即帐户愿意为交易付款。 应使用此方法通过账户支付费用。 请注意，如果您的帐户永远不会支付任何费用并且将始终依赖 [paymaster](#paymasters) 功能，则您不必使用此方法。 此方法必须至少向 [bootloader](./contracts/system-contracts.md#bootloader) 地址发送 `tx.gasprice * tx.ergsLimit` ETH。
- `prePaymaster`是可选的，如果交易有一个paymaster，系统会调用它，即有一个不同的地址为用户支付交易费用。 此方法应用于准备与付款人的交互。 一个值得注意的 [示例](#approval-based-paymaster-flow) 可能会有所帮助，即为付款人批准 ERC-20 代币。
- `executeTransactionFromOutside`，从技术上讲，不是强制性的，但 _强烈鼓励_ ，因为需要某种方式，在优先模式的情况下（例如，如果运营者没有响应），能够从您的帐户开始交易 ''outside''（基本上这是标准以太坊方法的回退，EOA 从你的智能合约开始交易）。
- `executeTransaction` 是强制性的，将在向用户收取费用后由系统调用。 这个函数应该执行事务。

###支付接口

就像在 EIP4337 中一样，我们的账户抽象协议支持`paymaster` ：可以补偿其他账户交易执行的账户。 你可以在[这里](#paymasters)阅读更多关于它们的信息。

每个付款人都应该实现 [IPaymaster](https://github.com/matter-labs/v2-testnet-contracts/blob/main/l2/system-contracts/interfaces/IPaymaster.sol) 接口。它包含以下两种方法：

- `validateAndPayForPaymasterTransaction` 是强制性的，系统将使用它来确定付款人是否批准为此交易付款。如果付款人愿意为交易付款，则此方法必须、至少向运营商发送 `tx.gasprice * tx.ergsLimit`。它应该返回将成为 `postOp` 方法的调用`context`参数之一。
- `postOp` 是可选的，将在交易执行后调用。请注意，与 EIP4337 不同，_不保证会调用此方法_。特别是，如果交易因 `out of gas` 错误而失败，则不会调用此方法。它有四个参数：`validateAndPayForPaymasterTransaction` 方法返回的内容、交易本身、交易执行是否成功，以及付款可能退还的最大 erg 数量。将支持添加到 zkSync 后，会提供更多有关退款的文档。

### 具有特殊含义的 `Transaction` 结构的保留字段

请注意，上述每种方法都接受 [Transaction](https://github.com/matter-labs/v2-testnet-contracts/blob/0e1c95969a2f92974370326e4430f03e417b25e7/l2/system-contracts/TransactionHelper.sol#L15) 结构。
虽然它的一些字段是不言自明的，但也有 6 个“保留”字段，每个字段的含义由交易类型定义。 我们决定不给这些字段命名，因为在未来的某些交易类型中可能不需要它们。 目前，约定是：

- `reserved[0]` 是随机数。
- `reserved[1]` 是应该与交易一起传递的 `msg.value`。

### 交易流程

每笔交易都经过以下流程：

#### 验证

在验证步骤期间，帐户应决定是否接受交易，如果接受，则支付费用。如果任何部分验证失败，则不会向账户收取费用，并且此类交易不能被包含在区块中。

**步骤 1.** 系统检查交易的 nonce 是否之前没有被使用过。您可以在 [此处](#keeping-noces-unique) 阅读有关保留随机数唯一性的更多信息。

**步骤 2.** 系统调用账户的`validateTransaction`方法。如果它没有恢复，请继续下一步。

**步骤 3.** 系统检查交易的 nonce 是否已标记为已使用。

**步骤 4 (no paymaster).** 系统调用账户的`payForTransaction`方法。如果它没有恢复，请继续下一步。

**步骤5 (paymaster).** 系统调用发送方的`prePaymaster`方法。如果此调用未恢复，则调用付款人 `validateAndPayForPaymasterTransaction` 。如果它也没有恢复，请继续下一步。

**步骤 6.** 系统验证引导加载程序已至少收到 `tx.ergsPrice * tx.ergsLimit` ETH 给引导加载程序。如果是，则考虑验证完成，我们可以进行下一步。

#### 执行步骤

执行步骤被认为负责实际执行交易并将任何未使用的 erg 的退款发送回用户。 如果这一步有任何还原，交易仍然被认为是有效的，并将被包含在块中。

**Step 7.** 系统调用账户的`executeTransaction`方法。

**Step 8。（仅在交易有付款人的情况下）** 付款人的 `postOp` 方法被调用。 此步骤通常用于向发件人退款
未使用的 ergs，以防付款人被用来促进以 ERC-20 代币支付费用。

### Fees

费用

在 EIP4337 中，您可以看到三种类型的气体限制：`verificationGas`、`executionGas`、`preVerificationGas`，它们描述了区块中包含交易的不同步骤的气体限制。
zkSync 2 只有一个字段`ergsLimit`，涵盖了所有三个字段的费用。 提交交易时，确保 `ergsLimit` 足以覆盖验证，支付费用（上面提到的 ERC20 转账），以及实际执行本身。

默认情况下，调用 `estimateGas` 会添加一个常量来覆盖 EOA 账户的收费和签名验证。

## 使用 `SystemContractsCaller` 库

为了安全起见，`NonceHolder` 和 `ContractDeployer` 系统合约都只能使用特殊的 `isSystem` 标志调用。您可以在 [此处](./contracts/system-contracts.md#protected-access-to-some-of-the-system-contracts) 阅读更多相关信息。要使用此标志进行调用，请使用 [SystemContractsCaller](https://github.com/matter-labs/v2-testnet-contracts/blob/sb-system-contracts-for-new-update) 的 `systemCall` 方法应该使用[SystemContractsCaller](https://github.com/matter-labs/v2-testnet-contracts/blob/sb-system-contracts-for-new-update/l2/system-contracts/SystemContractsCaller.sol) 库。

在开发自定义账户时使用这个库上是必须的，因为这是调用`NonceHolder`系统合约的非视图方法的唯一方法。此外，如果你想允许用户部署他们自己的合约，你将不得不使用这个库。您可以使用[实现](https://github.com/matter-labs/v2-testnet-contracts/blob/sb-system-contracts-for-new-update/l2/system-contracts/DefaultAccount.sol) EOA 帐户作为参考。

## 扩容 EIP4337

为了向运营商提供 DoS 保护，EIP4337 对帐户的验证步骤施加了一些[限制](https://eips.ethereum.org/EIPS/eip-4337#simulation)。
他们中的大多数，尤其是关于禁止操作码的，仍然具有相关性。 但为了更好的用户体验，已经取消了一些限制。

### 扩容允许的操作码

- 允许调用已经部署的合约。 与以太坊不同，我们无法编辑已部署的代码或通过 selfdestruct 删除合约，因此我们可以确定合约执行期间的代码是相同的。

### 扩展属于用户的槽集

在原始EIP中，AA的`validateTransaction`步骤允许账户只读取自己的存储槽。 然而，有些插槽在语义上属于该用户，但实际上位于另一个合约的地址上。 一个值得注意的例子是 `ERC20`余额。

此限制通过确保不同帐户用于验证的插槽 _不重叠_ 来提供 DDoS 安全性，因此它们没有必要 _实际上_ 属于帐户的存储。

为了能够在验证步骤中读取用户的 ERC20 余额或补贴，在验证步骤中，地址为`A`的帐户将允许使用以下类型的插槽：

1. 属于地址`A`的插槽。
2. 在任何其他地址上插入`A`。
3. 任何其他地址上类型为`keccak256(A || X)`的插槽。 （覆盖 `mapping(address => value)`，它通常用于 ERC20 代币中的余额）。
4. 任何其他地址上类型为`keccak256(X || OWN)`的插槽，其中`OWN`是前一个（第三）类型的某个插槽（以覆盖“映射（地址⇒映射（地址⇒uint256））`通常是 用于 ERC20 代币中的“补贴`）。

### 将来可以允许什么？

将来，我们甚至可能允许有时间限制的交易，例如 允许检查 `block.timestamp <= value` 如果它返回 `false` 等。这将需要部署一个单独的此类可信方法库，但它会大大增加帐户的功能。

## 建立自定义帐户

如上所述，每个帐户都应实现 [IAccount](#iaccount-interface) 接口。

AA 接口实现的一个例子是[实现](https://github.com/matter-labs/v2-testnet-contracts/blob/6a93ff85d33dfff0008624eb9777d5a07a26c55d/l2/system-contracts/DefaultAA.sol#L16) EOA 帐户。
请注意，此帐户与以太坊上的标准 EOA 帐户一样，无论何时被外部地址调用都会成功返回空值，而这可能不是您希望帐户的行为。

### EIP1271

如果您正在构建智能钱包，我们也 _强烈建议_ 您实施 [EIP1271](https://eips.ethereum.org/EIPS/eip-1271) 签名验证方案。
这是 zkSync 团队认可的标准。 它用于本节下面描述的签名验证库。

### 部署过程

部署账户逻辑的过程与部署智能合约的过程非常相似。
为了保护不想被视为账户的智能合约，应该使用部署系统合约的不同方法来做到这一点。
您应该使用部署系统合约的 `createAccount`/`create2Account` 方法，而不是使用 `create`/`create2`。

以下是如何使用 `zksync-web3` SDK 部署帐户逻辑的示例：

```ts
import { ContractFactory } from "zksync-web3";

const contractFactory = new ContractFactory(abi, bytecode, initiator, "createAccount");
const aa = await contractFactory.deploy(...args);
await aa.deployed();
```

### 验证步骤的局限性

::: 警告 尚未实施

目前尚未完全执行验证规则。 即使您的自定义帐户现在可以正常使用，如果不遵守以下规则，它将来也可能会停止工作。

:::

为了保护系统免受 DoS 威胁，验证步骤必须具有以下限制：

- 账户逻辑只能接触属于该帐户的插槽。 请注意，[定义](#extending-the-set-of-slots-that-belong-to-a-user) 远远超出了用户地址处的插槽。
- 账户逻辑不能使用上下文变量（例如 `block.number`）
- 还要求您的账户将 nonce 增加 1。仅需要此限制以保持交易哈希抗冲突性。 将来，将取消此要求以允许更通用的用例（例如隐私协议）。违反上述规则的交易将不会被 API 接受，尽管这些要求不能在电路/VM 级别强制执行并且不适用于 L1->L2 交易。

为了让您更快地试用该功能，我们决定在完全实施对账户验证步骤的限制检查之前公开发布账户抽象。
目前，尽管违反了上述要求，您的交易仍可以通过 API，但很快就会改变。

### Nonce持有者合约

出于优化目的，[tx nonce 和部署 nonce](./contracts/contracts.md#differences-in-create-behaviour) 都放在 [NonceHolder](./contracts/system-contracts. md#inonceholder) 系统合约。
为了增加您帐户的随机数，强烈建议调用 [incrementNonceIfEquals](https://github.com/matter-labs/v2-testnet-contracts/blob/0e1c95969a2f92974370326e4430f03e417b25e7/l2/system-contracts/interfaces /INonceHolder.sol#L10) 函数并传递交易中提供的随机数的值。

这是白名单调用之一，允许账户逻辑调用外部智能合约。

### 从账户发送交易

目前，仅支持 EIP712 交易。 要从特定账户提交交易，您应该提供交易的“from”字段作为发件人的地址，并在“customData”的“customSignature”字段中提供账户签名。

```ts
import { utils } from "zksync-web3";

// here the `tx` is a `TransactionRequest` object from `zksync-web3` SDK.
// and the zksyncProvider is the `Provider` object from `zksync-web3` SDK connected to zkSync network.
tx.from = aaAddress;
tx.customData = {
  ...tx.customData,
  customSignature: aaSignature,
};
const serializedTx = utils.serialize({ ...tx });

const sentTx = await zksyncProvider.sendTransaction(serializedTx);
```

## 付款人

想象一下能够为您的协议用户支付费用！Paymasters 是可以代替其他账户交易的账户。 的另一个重要用例    
paymasters 是为了方便使用 ERC20 代币支付费用。 虽然 ETH 是 zkSync 中的正式费用代币，但 paymasters 可以提供将 ERC20 代币即时兑换成 ETH 的能力。

如果用户想与 paymaster 交互，他们应该在他们的 EIP712 交易中提供非零的`paymaster` 地址。 在付款人的`paymasterInput`字段中提供付款人的输入数据。

### 付款人验证规则

::: 警告 尚未实施

目前尚未完全执行验证规则。 即使您的出纳员现在正在工作，如果不遵守以下规则，将来也可能会停止工作。

:::

由于应允许多个用户访问同一个付款人，恶意付款人 _可以_ 对我们的系统进行 DoS 攻击。 为了解决这个问题，将使用类似于 [EIP4337 信誉评分](https://eips.ethereum.org/EIPS/eip-4337#reputation-scoring-and-throttlingbanning-for-paymasters) 的系统。

与原始 EIP 不同，付款人可以接触任何存储槽。 此外，如果满足以下任一条件，付款人将不会受到限制：

- 自 API 节点验证通过以来，已经过去了超过 `X` 分钟（`X` 的确切值将在稍后定义）。
- 被读取槽的顺序与 API 节点上运行期间的顺序相同，第一个值发生变化的槽是用户的槽之一。 这是保护付款人免受恶意用户侵害所必需的（例如，用户可能已经删除了 ERC20 代币的配额）。

### 内置付款人流程

虽然一些付款人可以在没有用户交互的情况下进行简单的操作（例如，始终为用户支付费用的协议），但有些付款人需要交易发送者的积极参与。 一个值得注意的例子是将用户的 ERC20 代币交换为 ETH 的 paymaster，因为它要求用户为 paymaster 设置必要的津贴。

账户抽象协议本身是通用的，允许账户和付款人实现任意交互。 然而，默认账户 (EOA) 的代码是不变的，但我们仍然希望它们能够参与自定义账户和付款人的生态系统。 这就是为什么我们对交易的` paymasterInput` 字段进行了标准化，以涵盖 paymaster 功能的最常见用例。

您的账户可以自由实施或不实施对这些流程的支持。 但是，强烈建议保持 EOA 和自定义帐户的界面相同。

#### 一般付款流程

如果付款人操作不需要用户事先采取行动，则应使用它。

`paymasterInput` 字段必须编码为对具有以下接口的函数的调用：

```solidity
function general(bytes calldata data);
```

EOA 账户什么都不做，出纳员可以以任何方式解释这个`数据`。

#### 基于批准的付款流程

如果要求用户为付款人操作的代币设置一定的补贴，则应该使用它。 `paymasterInput` 字段必须编码为对具有以下签名的函数的调用：

```solidity
function approvalBased(
    address _token,
    uint256 _minAllowance,
    bytes calldata _innerInput
)
```

EOA 将确保支付主管的`token`限额至少设置为`minAllowance`。 `_innerInput` 参数是一个额外的有效负载，可以发送给付款人以实现任何逻辑（例如，可以由付款人验证的额外签名或密钥）。

如果你正在开发付款人，你 _不应该_ 相信交易发送方会诚实（例如，通过 `approvalBased` 流程提供所需的补贴）。 这些流程主要用作对 EOA 的说明，付款主管应始终仔细检查这些要求。

#### 使用 `zksync-web3` SDK 处理 paymaster 流程

`zksync-web3` SDK 提供了 [方法](../../api/js/utils.md#encoding-paymaster-param) 用于为所有内置的 paymaster 流编码格式正确的 paymaster 参数。

### 测试网支付主管

为了确保用户在 testnet 上体验 paymasters，并继续支持使用 ERC20 代币支付费用，Matter Labs 团队提供了 testnet paymaster，它能够以与 ETH 1:1 的汇率（即这个单位的一个单位）以 ERC20 代币支付费用。 token 等于 1 wei 的 ETH）。

paymaster 仅支持 [approval based](#approval-based-paymaster-flow) paymaster 流程，并要求 `token` 参数等于被交换的令牌，`minAllowance` 至少等于 `tx.maxFeePerErg * tx .ergsLimit`。 此外，testnet paymaster 不使用 `_innerInput` 参数，因此不应提供任何内容（空 `bytes`）。

在 [快速入门](./hello-world.md#paying-fees-using-testnet-paymaster) 教程中可以看到如何使用 testnet paymaster 的示例。

## `aa-signature-checker`

您的项目可以开始为原生 AA 支持做准备。我们强烈建议您这样做，因为它将允许您加入数十万用户（例如，已经使用第一版 zkSync 的 Argent 用户）。
我们预计未来会有更多用户转向智能钱包。

要建立的各种类型的帐户之间最显着的区别之一是不同的签名方案。我们希望账户支持 [EIP-1271](https://eips.ethereum.org/EIPS/eip-1271) 标准。我们的团队创建了一个用于验证帐户签名的实用程序库。目前，它仅支持 ECDSA 签名，但我们很快也会添加对 EIP-1271 的支持。

`aa-signature-checker` 库提供了一种验证不同帐户实现签名的方法。目前仅支持验证 ECDSA 签名。很快我们也会添加对 EIP-1271 的支持。

_我们强烈建议您在需要检查帐户签名是否正确时使用此库。_

### 将库添加到您的项目：

```
yarn add @matterlabs/signature-checker
```

### 使用库的例子

```solidity
pragma solidity ^0.8.0;

import { SignatureChecker } from "@matterlabs/signature-checker/contracts/SignatureChecker.sol";

contract TestSignatureChecker {
    using SignatureChecker for address;

    function isValidSignature(
        address _address,
        bytes32 _hash,
        bytes memory _signature
    ) public pure returns (bool) {
        return _address.checkSignature(_hash, _signature);
    }
}
```

## 在我们的 SDK 中验证 AA 签名

**不推荐**使用 `ethers.js` 库来验证用户签名。

我们的 SDK 提供了两种方法及其 utils 来验证帐户的签名：

```ts
export async function isMessageSignatureCorrect(address: string, message: ethers.Bytes | string, signature: SignatureLike): Promise<boolean>;

export async function isTypedDataSignatureCorrect(
  address: string,
  domain: TypedDataDomain,
  types: Record<string, Array<TypedDataField>>,
  value: Record<string, any>,
  signature: SignatureLike
): Promise<boolean>;
```

目前这些方法仅支持验证ECDSA签名，但很快它们将支持EIP1271签名验证。

这两种方法都根据消息签名是否正确返回“true”或“false”。
