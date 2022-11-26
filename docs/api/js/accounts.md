# 账户: 概述

`zksync-web3` 导出了四个可以在 zkSync 上签署交易的类：

- `Wallet` 类是 `ethers.Wallet` 的扩展，具有额外的 zkSync 功能。
- `EIP712Signer` 类，用于签署 `EIP712` 类型的 zkSync 交易。
- `Signer` and `L1Signer` 类，应用于浏览器集成。

## `Wallet`

### 从私钥创建钱包

就像`ethers.Wallet`一样，`zksync-web3`的`Wallet`对象可以由Ethereum私钥创建。

```typescript
constructor(
    privateKey: ethers.BytesLike | utils.SigningKey,
    providerL2?: Provider,
    providerL1?: ethers.providers.Provider
)
```

#### 输入和输出

| 名称                    | 说明                          |
| --------------------- | --------------------------- |
| privateKey            | 以太坊账户的私钥                    |
| providerL2 (optional) | zkSync 节点提供者。 需要与 zkSync 交互 |
| providerL1 (optional) | 以太坊节点提供商。 需要与 L1 交互         |
| returns               | 新`Wallet` 对象                |

> 例子

```typescript
import * as zksync from "zksync-web3";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://zksync2-testnet.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("goerli");
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);
```

### 创建 Wallet 实例的其他方法

`Wallet` 类支持 `ethers.Wallet` 中用于创建钱包的所有方法，例如 从助记词创建，从加密的 JSON 创建，创建随机钱包等。所有这些方法都采用与 ethers.Wallet 相同的参数，因此您应该参考其文档以了解如何使用它们。

### 连接到 zkSync provider

要与 zkSync 网络交互，应将 Wallet 对象连接到 Provider，方法是将其传递给构造函数或使用 connect 方法。

```typescript
connect(provider: Provider): Wallet
```

#### 输入和输出

| 名称       | 说明                            |
| -------- | ----------------------------- |
| provider | 一个 zkSync 节点提供者               |
| returns  | 一个新的 `Wallet` 对象, 连接到 zkSync. |

> Example

```typescript
import { Wallet, Provider } from "zksync-web3";

const unconnectedWallet = new Wallet(PRIVATE_KEY);

const provider = new Provider("https://zksync2-testnet.zksync.dev");
const wallet = unconnectedWallet.connect(provider);
```

### 连接到 Ethereum provider

为了执行L1操作，`Wallet`对象需要连接到`ethers.providers.Provider`对象。

```typescript
connectToL1(provider: ethers.providers.Provider)
```

#### 输入和输出

| Name     | 说明                       |
| -------- | ------------------------ |
| provider | 以太坊节点提供商                 |
| returns  | 一个新的 `Wallet` 对象, 连接到以太坊 |

> Example

```typescript
import { Wallet } from "zksync-web3";
import { ethers } from "ethers";

const unconnectedWallet = new Wallet(PRIVATE_KEY);

const ethProvider = ethers.getDefaultProvider("goerli");
const wallet = unconnectedWallet.connectToL1(ethProvider);
```

可以链接 `connect` 和 `connectToL1` 的方法：

```typescript
const wallet = unconnectedWallet.connect(provider).connectToL1(ethProvider);
```

### 获取 zkSync L1智能合约

```typescript
async getMainContract(): Promise<Contract>
```

#### 输入和输出

| 名称      | 说明                         |
| ------- | -------------------------- |
| returns | `Contract` zkSync 智能合约的包装器 |

> Example

```typescript
import * as zksync from "zksync-web3";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://zksync2-testnet.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("goerli");
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);

const contract = await wallet.getMainContract();
console.log(contract.address);
```

### 获取代币余额

```typescript
async getBalance(token?: Address, blockTag: BlockTag = 'committed'): Promise<BigNumber>
```

#### 输入和输出

| 名称                  | 说明                            |
| ------------------- | ----------------------------- |
| token (optional)    | 代币的地址。 默认为以太币                 |
| blockTag (optional) | 应检查余额`committed`块，即最新处理的是默认选项 |
| returns             | `Wallet` 拥有的代币数量              |

> Example

```typescript
import * as zksync from "zksync-web3";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://zksync2-testnet.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("goerli");
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider);

// Getting balance in USDC
console.log(await wallet.getBalance(USDC_L2_ADDRESS));

// Getting balance in ETH
console.log(await wallet.getBalance());
```

### 在 L1 上获取代币余额

```typescript
async getBalanceL1(token?: Address, blockTag?: ethers.providers.BlockTag): Promise<BigNumber>
```

#### 输入和输出

| 名称                  | 说明                  |
| ------------------- | ------------------- |
| token (optional)    | 代币的地址。 默认为以太币       |
| blockTag (optional) | 应检查余额的块。 最新处理的是默认选项 |
| returns             | `Wallet` 在以太坊上的代币数量 |

> Example

```typescript
import * as zksync from "zksync-web3";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://zksync2-testnet.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("goerli");
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider);

// Getting balance in USDC
console.log(await wallet.getBalanceL1(USDC_ADDRESS));

// Getting balance in ETH
console.log(await wallet.getBalanceL1());
```

### 获得随机数

`Wallet` 还提供了 `getNonce` 方法，它是 [getTransactionCount] 的别名](https://docs.ethers.io/v5/api/signer/#Signer-getTransactionCount).

```typescript
async getNonce(blockTag?: BlockTag): Promise<number>
```

#### 输入与输出

| 名称                  | 说明                                  |
| ------------------- | ----------------------------------- |
| blockTag (optional) | 随机数应该放在的块上。 `committed`，即最新处理的是默认选项 |
| returns             | `Wallet` 拥有的代币数量                    |

> Example

```typescript
import * as zksync from "zksync-web3";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://zksync2-testnet.zksync.dev");
// Note that we don't need ethereum provider to get the nonce
const wallet = new Wallet(PRIVATE_KEY, zkSyncProvider);

console.log(await wallet.getNonce());
```

### 在 zkSync 内部传输令牌

为了方便起见，`钱包 `类有 `转移 `方法，它可以在同一接口内转移 `ETH `或任何 `ERC20 `令牌。

```typescript
async transfer(tx: {
    to: Address;
    amount: BigNumberish;
    token?: Address;
    overrides?: ethers.CallOverrides;
}): Promise<TransactionResponse>
```

#### 输出与输入

| 名称                   | 说明                           |
| -------------------- | ---------------------------- |
| tx.to                | 收件人的地址                       |
| tx.amount            | 要转移的代币数量                     |
| token (optional)     | 代币的地址。 默认为`ETH`              |
| overrides (optional) | 交易覆盖，例如 `nonce`、`gasLimit` 等 |
| returns              | 一个 `TransactionResponse` 对象  |

> Example

```typescript
import * as zksync from "zksync-web3";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://zksync2-testnet.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("goerli");
const wallet = new zksync.Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);

const recipient = zksync.Wallet.createRandom();

// We transfer 0.01 ETH to the recipient and pay the fee in USDC
const transferHandle = wallet.transfer({
  to: recipient.address,
  amount: ethers.utils.parseEther("0.01"),
});
```

### 发起向 L1 的提款

```typescript
async withdraw(transaction: {
    token: Address;
    amount: BigNumberish;
    to?: Address;
    bridgeAddress?: Address;
    overrides?: ethers.CallOverrides;
}): Promise<TransactionResponse>
```

| 名称                       | 说明                           |
| ------------------------ | ---------------------------- |
| tx.to                    | L1 上收件人的地址                   |
| tx.amount                | 要转移的代币数量                     |
| token (optional)         | 代币的地址，默认为 `ETH`              |
| bridgeAddress (optional) | 要使用的桥接合约的地址                  |
| overrides (optional)     | 交易覆盖，例如 `nonce`、`gasLimit` 等 |
| returns                  | 一个 `TransactionResponse` 对象  |

### 检索底层 L1 钱包

您可以使用 `ethWallet() `方法获得具有相同私钥的 `ethers.Wallet` 对象。

#### 输入和输出

| 名称      | 说明                        |
| ------- | ------------------------- |
| returns | 具有相同私钥的`ethers.Wallet`对象。 |

> Example

```typescript
import * as zksync from "zksync-web3";
import { ethers } from "ethers";

const PRIVATE_KEY = "0xc8acb475bb76a4b8ee36ea4d0e516a755a17fad2e84427d5559b37b544d9ba5a";

const zkSyncProvider = new zksync.Provider("https://zksync2-testnet.zksync.dev");
const ethereumProvider = ethers.getDefaultProvider("goerli");
const wallet = new zksync.Wallet(PRIVATE_KEY, zkSyncProvider, ethereumProvider);

const ethWallet = wallet.ethWallet();
```

## `EIP712Signer`

这个类的方法大多在内部使用。 使用此类的示例即将推出！

## `Signer`

此类将在浏览器环境中使用。 构造它的最简单方法是使用 `Web3Provider` 的 `getSigner` 方法。 此结构扩展了“ethers.providers.JsonRpcSigner”，因此支持所有可用的方法。

```typescript
import { Web3Provider } from "zksync-web3";

const provider = new Web3Provider(window.ethereum);
const signer = provider.getSigner();
```

### 获得代币余额

```typescript
async getBalance(token?: Address, blockTag: BlockTag = 'committed'): Promise<BigNumber>
```

#### 输入和输出

| 名称                  | 说明                               |
| ------------------- | -------------------------------- |
| token (optional)    | 代币的地址。 默认为ETH                    |
| blockTag (optional) | 应检查余额的块。 `committed`，即最新处理的是默认选项 |
| returns             | `Signer` 拥有的代币数量。                |

> Example

```typescript
import { Web3Provider } from "zksync-web3";
import { ethers } from "ethers";

const provider = new Web3Provider(window.ethereum);
const signer = provider.getSigner();

// Getting balance in USDC
console.log(await signer.getBalance(USDC_L2_ADDRESS));

// Getting balance in ETH
console.log(await signer.getBalance());
```

### 获得随机数

`Wallet` 类还提供了 `getNonce` 方法，它是 [getTransactionCount](https://docs.ethers.io/v5/api/signer/#Signer-getTransactionCount) 的别名。

```typescript
async getNonce(blockTag?: BlockTag): Promise<number>
```

#### 输入和输出

| 名称                  | 说明                                   |
| ------------------- | ------------------------------------ |
| blockTag (optional) | 随机数应该放在的块上。 `committed`，即最新处理的是默认选项。 |
| returns             | The the `Wallet` has.                |

> Example

```typescript
import { Web3Provider } from "zksync-web3";

const provider = new Web3Provider(window.ethereum);
const signer = provider.getSigner();

console.log(await signer.getNonce());
```

### 在 zkSync 中传输代币

请注意，目前与以太坊不同，zkSync 不支持原生转账，即所有交易的 `value` 字段都等于 `0`。 所有代币转移都是通过 ERC20 的 `转移` 函数调用完成的。

但是为了方便起见，`Wallet` 类有`transfer` 方法，可以转移任何`ERC20` 代币。

```typescript
async transfer(tx: {
    to: Address;
    amount: BigNumberish;
    token?: Address;
    overrides?: ethers.CallOverrides;
}): Promise<ethers.ContractTransaction>
```

#### 输入和输出

| 名称                   | 说明                              |
| -------------------- | ------------------------------- |
| tx.to                | 收件人的地址                          |
| tx.amount            | 要转移的代币数量                        |
| token (optional)     | 代币的地址 ，默认为`ETH`                 |
| overrides (optional) | 交易覆盖，例如 `nonce`、`gasLimit` 等    |
| returns              | `ethers.ContractTransaction` 对象 |

> 例子

```typescript
import { Wallet, Web3Provider } from "zksync-web3";
import { ethers } from "ethers";

const provider = new Web3Provider(window.ethereum);
const signer = provider.getSigner();

const recipient = Wallet.createRandom();

// We transfer 0.01 ETH to the recipient and pay the fee in USDC
const transferHandle = signer.transfer({
  to: recipient.address,
  amount: ethers.utils.parseEther("0.01"),
});
```

## `L1Signer`

此类将在浏览器环境中用于在第 1 层执行与 zkSync 相关的操作。此类扩展了 ethers.providers.JsonRpcSigner，因此支持所有可用的方法。

构建它的最简单方法是从`Web3Provider`对象。

```typescript
import { Web3Provider, Provider, L1Signer } from "zksync-web3";

const provider = new ethers.Web3Provider(window.ethereum);
const zksyncProvider = new Provider("https://zksync2-testnet.zksync.dev");
const signer = L1Signer.from(provider.getSigner(), zksyncProvider);
```

### 获取 zkSync L1 智能合约

```typescript
async getMainContract(): Promise<Contract>
```

#### 输入和输出

| 名称      | 描述                                               |
| ------- | ------------------------------------------------ |
| returns | `Contract` wrapper of the zkSync smart contract. |

> 例子

```typescript
import { Web3Provider, Provider, L1Signer } from "zksync-web3";
import { ethers } from "ethers";

const provider = new ethers.Web3Provider(window.ethereum);
const zksyncProvider = new Provider("https://zksync2-testnet.zksync.dev");
const signer = L1Signer.from(provider.getSigner(), zksyncProvider);

const mainContract = await signer.getMainContract();
console.log(mainContract.address);
```

### 在 L1 上获取代币余额

```typescript
async getBalanceL1(token?: Address, blockTag?: ethers.providers.BlockTag): Promise<BigNumber>
```

#### 输入和输出

| 名称                  | 说明                     |
| ------------------- | ---------------------- |
| token (optional)    | 代币的地址。 默认为 ETH。        |
| blockTag (optional) | 应检查的块。一个 最新处理的默认选项     |
| returns             | `L1Signer` 在以太坊上的代币数量。 |

> 例子

```typescript
import { Web3Provider, Provider, L1Signer } from "zksync-web3";
import { ethers } from "ethers";

const provider = new ethers.Web3Provider(window.ethereum);
const zksyncProvider = new Provider("https://zksync2-testnet.zksync.dev");
const signer = L1Signer.from(provider.getSigner(), zksyncProvider);

// Getting balance in USDC
console.log(await signer.getBalanceL1(USDC_ADDRESS));

// Getting balance in ETH
console.log(await signer.getBalanceL1());
```
