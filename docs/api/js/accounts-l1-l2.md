# 账户：L1->L2交易

本节探讨允许 [账户](./accounts.md) 类将交易从 L1 发送到 L2 的方法。

如果您想了解 L1->L2 交互如何在 zkSync 上工作的一些背景知识，请阅读 [介绍](../../dev/developer-guides/bridging/l1-l2-interop.md) 和 [ 指南](../dev/../../dev/developer-guides/bridging/l1-l2.md)。

## 支持的账户类

以下帐户类支持将交易从L1发送到L2：

- `钱包`（如果连接到 L1 提供商）
- `L1Signer`

## 批准代币的存放

从以太坊桥接 ERC20 代币需要批准代币到 zkSync 以太坊智能合约。

```typescript
async approveERC20(
    token: Address,
    amount: BigNumberish,
    overrides?: ethers.Overrides & { bridgeAddress?: Address }
): Promise<ethers.providers.TransactionResponse>
```

### 输入和输出

| 名称                   | 说明                                                                                      |
| -------------------- | --------------------------------------------------------------------------------------- |
| token                | 以太坊上代币的地址                                                                               |
| amount               | 要批准的代币数量                                                                                |
| overrides (optional) | 以太坊交易覆盖。 可用于传递 `gasLimit`、`gasPrice` 等。您也可以提供 L1 网桥的自定义地址以供使用（默认使用 Matter Labs 团队提供的网桥） |
| returns              | `ethers.providers.TransactionResponse`对象                                                |

> Example

```typescript
import * as zksync from "zksync-web3";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://zksync2-testnet.zksync.dev/");
const ethereumProvider = ethers.getDefaultProvider("goerli");
const wallet = new zksync.Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);

const USDC_ADDRESS = "0xd35cceead182dcee0f148ebac9447da2c4d449c4";
const txHandle = await wallet.approveERC20(
  USDC_ADDRESS,
  "10000000" // 10.0 USDC
);

await txHandle.wait();
```

## 向zkSync存入代币

```typescript
async deposit(transaction: {
  token: Address;
  amount: BigNumberish;
  to?: Address;
  operatorTip?: BigNumberish;
  bridgeAddress?: Address;
  approveERC20?: boolean;
  overrides?: ethers.PayableOverrides;
  approveOverrides?: ethers.Overrides;
}): Promise<PriorityOpResponse>
```

#### 输入和输出

| 名称                                      | 说明                                                                                                             |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| transaction.token                       | 要存放的令牌的地址                                                                                                      |
| transaction.amount                      | 要存入的代币数量                                                                                                       |
| transaction.to (optional)               | 将在 L2 上接收存入代币的地址                                                                                               |
| transaction.operatorTip (optional)      | 如果随交易传递的 ETH`价值`未在覆盖中明确说明，则此字段将等于运营商将在交易基本成本之上收到的小费。 该值对`Deque`类型的队列没有意义，但它将用于确定进入`Heap`或`HeapBuffer`队列的事务的优先级 |
| transaction.bridgeAddress (optional)    | 要使用的桥接合约的地址。 默认为默认的 zkSync 桥（`L1EthBridge` 或 `L1Erc20Bridge`）                                                  |
| transaction.approveERC20 (optional)     | 令牌批准是否应该在幕后执行。 如果您桥接 ERC20 令牌并且没有事先调用 approveERC20 函数，请将此标志设置为 true                                            |
| transaction.overrides (optional)        | 以太坊交易覆盖。 可用于传递 `gasLimit`、`gasPrice` 等                                                                         |
| transaction.approveOverrides (optional) | 以太坊交易覆盖批准交易。 可用于传递 `gasLimit`、`gasPrice` 等                                                                     |
| returns                                 | `PriorityOpResponse` 对象                                                                                        |

> Example

```typescript
import * as zksync from "zksync-web3";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://zksync2-testnet.zksync.dev/");
const ethereumProvider = ethers.getDefaultProvider("goerli");
const wallet = new zksync.Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);

const USDC_ADDRESS = "0xd35cceead182dcee0f148ebac9447da2c4d449c4";
const usdcDepositHandle = await wallet.deposit({
  token: USDC_ADDRESS,
  amount: "10000000",
  approveERC20: true,
});
// Note that we wait not only for the L1 transaction to complete but also for it to be
// processed by zkSync. If we want to wait only for the transaction to be processed on L1,
// we can use `await usdcDepositHandle.waitL1Commit()`
await usdcDepositHandle.wait();

const ethDepositHandle = await wallet.deposit({
  token: zksync.utils.ETH_ADDRESS,
  amount: "10000000",
});
// Note that we wait not only for the L1 transaction to complete but also for it to be
// processed by zkSync. If we want to wait only for the transaction to be processed on L1,
// we can use `await ethDepositHandle.waitL1Commit()`
await ethDepositHandle.wait();
```

## 向zkSync添加本地令牌

新的代币在第一次存入时就会自动添加。

## 完成提款

提款分两步执行 - 在 L2 上启动并在 L1 上完成。

```typescript
async finalizeWithdrawal(withdrawalHash: BytesLike, index: number = 0): Promise<ethers.TransactionResponse>
```

#### 输入和输出

| 名称               | 说明                                    |
| ---------------- | ------------------------------------- |
| withdrawalHash   | 发起取款的L2交易哈希                           |
| index (optional) | 如果在一笔交易中有多笔取款，您可以传递一个您想要完成的取款索引（默认为0） |

## 在L2上强制执行交易

### 获取交易的基础成本

```ts
async getBaseCost(params: {
    ergsLimit: BigNumberish;
    calldataLength: number;
    gasPrice?: BigNumberish;
}): Promise<BigNumber>
```

#### 输入和输出

| 名称                         | 描述                       |
| -------------------------- | ------------------------ |
| params.ergsLimit           | `ergsLimit` 调用           |
| params.calldataLength      | 调用数据的长度（以字节为单位）          |
| params.gasPrice (optional) | 将发送执行调用请求的 L1 交易的 gas 价格 |
| returns                    | 请求合约调用的 ETH 基本成本         |

## 失败的存款

`claimFailedDeposit` 方法从启动的存款中提取资金，该存款在 L2 上完成时提示失败。
如果存款 L2 交易失败，它会发送一个 L1 交易调用 L1 桥的 `claimFailedDeposit` 方法，这会导致将 L1 代币返回给存款人，否则会出现错误。

```ts
async claimFailedDeposit(depositHash: BytesLike): Promise<ethers.ContractTransaction>
```

### 输入和输出

| 名称          | Type      | 说明             |
| ----------- | --------- | -------------- |
| depositHash | `bytes32` | 存款失败的 L2 交易哈希。 |

### 请求交易执行

```ts
async requestExecute(transaction: {
    contractAddress: Address;
    calldata: BytesLike;
    ergsLimit: BigNumberish;
    factoryDeps?: ethers.BytesLike[];
    operatorTip?: BigNumberish;
    overrides?: ethers.CallOverrides;
}): Promise<PriorityOpResponse>
```

#### 输入和输出

| Name                               | Description                                                                                                       |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| transaction.contractAddress        | 要调用的L2合约的地址                                                                                                       |
| transaction.calldata               | 调用交易的数据。 它可以像在以太坊中一样被编码                                                                                           |
| transaction.ergsLimit              | `ergsLimit`调用                                                                                                     |
| transaction.factoryDeps            | 工厂依赖的字节码数组——仅用于部署合约的交易                                                                                            |
| transaction.operatorTip (optional) | 如果随交易传递的 ETH`value`未在覆盖中明确说明，则此字段将等于运营商将在交易基本成本之上收到的小费。 该值对“Deque”类型的队列没有意义，但它将用于确定进入“Heap”或“HeapBuffer”队列的事务的优先级 |
| overrides (optional)               | 以太坊交易覆盖。 可用于传递 `gasLimit`、`gasPrice` 等                                                                            |
| returns                            | `PriorityOpResponse` 对象                                                                                           |

> Example

```typescript
import * as zksync from "zksync-web3";
import { BigNumber, ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://zksync2-testnet.zksync.dev/");
const ethereumProvider = ethers.getDefaultProvider("goerli");
const wallet = new zksync.Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);

const gasPrice = await wallet.providerL1!.getGasPrice();

// The calldata can be encoded the same way as for Ethereum.
// Here is an example on how to get the calldata from an ABI:
const abi = [
  {
    inputs: [],
    name: "increment",
    outputs: [],
    stateMutability: "nonpayable",
    type: "function",
  },
];
const contractInterface = new ethers.utils.Interface(abi);
const calldata = contractInterface.encodeFunctionData("increment", []);
const ergsLimit = BigNumber.from(1000);

const txCostPrice = await wallet.getBaseCost({
  gasPrice,
  calldataLength: ethers.utils.arrayify(calldata).length,
  ergsLimit,
});

console.log(`Executing the transaction will cost ${ethers.utils.formatEther(txCostPrice)} ETH`);

const executeTx = await wallet.requestExecute({
  calldata,
  ergsLimit,
  contractAddress: "0x19a5bfcbe15f98aa073b9f81b58466521479df8d",
  overrides: {
    gasPrice,
    value: txCostPrice,
  },
});

await executeTx.wait();
```
