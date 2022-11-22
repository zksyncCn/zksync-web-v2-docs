# Providers

供应商是 zkSync 节点交互的对象，以标准以太坊节点功能提供简洁、一致的接口。如果您不熟悉 `ethers` 中供应商的概念，您应该在 [此处](https://docs.ethers.io/v5/api/providers) 查看他们的文档。

zkSync 完全支持 Ethereum Web3 API，因此您可以使用 ethers.js 中的提供商对象。 然而，zkSync API 提供了一些额外的 JSON-RPC 方法，允许：

- 轻松跟踪 L1<->L2 交易。
- 确定最终交易的不同阶段。 默认情况下，我们的 RPC 返回有关服务器处理最后状态的信息，但某些用例可能只需要跟踪“已完成”的交易。

以及更多！ 通常，您可以使用 ethers 中的提供程序快速入门，但稍后切换到 zksync-web3 库中的提供程序。

`zksync-web3` 库导出两种类型的提供程序：

- `Provider ` 继承自 `ethers 的 JsonRpcProvider` 并提供对所有 zkSync JSON-RPC 端点的访问。
- `Web3Provider` 通过使其与 Web3 钱包更兼容来扩展 `Provider` 类。 这是应用于浏览器集成的钱包类型。

## `Provider`

这是最常用的提供程序类型。 它提供与 ethers.providers.JsonRpcProvider 相同的功能，但使用 zkSync 特定方法对其进行扩展。

### Creating provider

构造函数接受运算符节点的“url”和“网络”名称（可选）。

```typescript
constructor(url?: ConnectionInfo | string, network?: ethers.providers.Networkish)
```

#### 输入和输出

| 名称                 | 说明                |
| ------------------ | ----------------- |
| url (optional)     | zkSync 运营商节点的 URL |
| network (optional) | 网络的描述             |
| returns            | `Provider`对象      |

> Example

```typescript
import { Provider } from "zksync-web3";

const provider = new Provider("https://zksync2-testnet.zksync.dev");
```

### `getBalance`

返回用户对某个区块标签和原生代币的余额。
为了检查 `ETH` 中的余额，您可以省略最后一个参数或提供 `utils` 对象中提供的 `ETH_ADDRESS`。

Example:

```typescript
async getBalance(address: Address, blockTag?: BlockTag, tokenAddress?: Address): Promise<BigNumber>
```

#### 输入和输出

| 名称                      | 说明                            |
| ----------------------- | ----------------------------- |
| address                 | 用户查询余额的地址                     |
| blockTag (optional)     | 检查余额的块`committed`，即最新处理的是默认选项 |
| tokenAddress (optional) | 代币的地址。 默认为ETH                 |
| returns                 | `BigNumber` 对象                |

> Example

```typescript
import { Provider } from "zksync-web3";

const provider = new Provider("https://zksync2-testnet.zksync.dev");

// Getting  USDC balance of account 0x0614BB23D91625E60c24AAD6a2E6e2c03461ebC5 at the latest processed block
console.log(await provider.getBalance("0x0614BB23D91625E60c24AAD6a2E6e2c03461ebC5", "latest", USDC_L2_ADDRESS));

// Getting ETH balance
console.log(await provider.getBalance("0x0614BB23D91625E60c24AAD6a2E6e2c03461ebC5"));
```

### 获取 zkSync 智能合约地址

```typescript
async getMainContractAddress(): Promise<string>
```

#### 输入和输出

| 名称      | 说明             |
| ------- | -------------- |
| returns | zkSync 智能合约的地址 |

> Example

```typescript
import { Provider } from "zksync-web3";

const provider = new Provider("https://zksync2-testnet.zksync.dev");

console.log(await provider.getMainContractAddress());
```

### 获取测试网付款人地址

在 zkSync 测试网上，[testnet paymaster](../../dev/developer-guides/aa.md#paymasters) 可用。

```typescript
async getTestnetPaymasterAddress(): Promise<string|null>
```

#### 输入和输出

| 名称      | 说明                                 |
| ------- | ---------------------------------- |
| returns | testnet paymaster 的地址，如果没有则为“null” |

> Example

```typescript
import { Provider } from "zksync-web3";

const provider = new Provider("https://zksync2-testnet.zksync.dev");

console.log(await provider.getTestnetPaymasterAddress());
```

### 获取 zkSync 默认桥接合约地址

```typescript
async getDefaultBridgeAddresses(): Promise<{
    ethL1?: Address;
    ethL2?: Address;
    erc20L1?: Address;
    erc20L2?: Address;
}>
```

#### 输入和输出

| 名称      | 说明                         |
| ------- | -------------------------- |
| returns | L1 和 L2 上默认 zkSync 桥接合约的地址 |

### `getConfirmedTokens`

 `from` 和 `limit` 返回关于 ID 在区间 `[from..from+limit-1]` 中的已确认代币的信息（地址、符号、名称、小数）。  "Confirmed" 在这里用词不当，因为确认代币是通过默认 zkSync 桥接的代币。 该方法将主要由 zkSync 团队内部使用。

标记按其符号的字母顺序返回，因此基本上，标记 id 是它在按字母顺序排列的标记数组中的位置。

```typescript
async getConfirmedTokens(start: number = 0, limit: number = 255): Promise<Token[]>
```

#### 输入和输出

| 名称      | 说明                       |
| ------- | ------------------------ |
| start   | 开始返回有关代币的信息的令牌 ID。 默认为零。 |
| limit   | 从 API 返回的代币数。 默认为 255。   |
| returns | 按符号排序的“Token”对象数组        |

> Example

```typescript
import { Provider } from "zksync-web3";
const provider = new Provider("https://zksync2-testnet.zksync.dev");

console.log(await provider.getConfirmedTokens());
```

### `getTokenPrice`

::: 警告已弃用

此方法已弃用，很快就会被删除。

:::

返回代币的 USD 价格。 请注意，这是 zkSync 团队使用的价格，可能与当前市场价格略有不同。 在测试网上，代币价格可能与实际市场价格有很大差异。

```typescript
async getTokenPrice(token: Address): Promise<string | null>
```

| 名称      | 说明               |
| ------- | ---------------- |
| token   | 代币的地址            |
| returns | `string` 价格的字符串值 |

> Example

```typescript
import { Provider } from "zksync-web3";
const provider = new Provider("https://zksync2-testnet.zksync.dev");

console.log(await provider.getTokenPrice(USDC_L2_ADDRESS));
```

### 从 L1 地址获取令牌在 L2 的地址，反之亦然

代币在 L2 上的地址将与 L1 上的地址不同。
ETH 的地址在两个网络上都设置为零地址。

提供的方法仅适用于使用默认 zkSync 桥接的代币。

```typescript
// takes L1 address, returns L2 address
async l2TokenAddress(l1Token: Address): Promise<Address>
// takes L2 address, returns L1 address
async l1TokenAddress(l2Token: Address): Promise<Address>
```

| 名称      | 说明               |
| ------- | ---------------- |
| token   | 代币的地址            |
| returns | 该代币在相对 Layer 的地址 |

### `getTransactionStatus`

交易哈希，返回交易的状态。

```typescript
async getTransactionStatus(txHash: string): Promise<TransactionStatus>
```

| 名称      | 说明                                                     |
| ------- | ------------------------------------------------------ |
| token   | 代币的地址                                                  |
| returns | 交易的状态。 您可以在 [types](./types) 中找到`TransactionStatus`的描述 |

> Example

```typescript
import { Provider } from "zksync-web3";
const provider = new Provider("https://zksync2-testnet.zksync.dev");

const TX_HASH = "0x95395d90a288b29801c77afbe359774d4fc76c08879b64708c239da8a65dbcf3";
console.log(await provider.getTransactionStatus(TX_HASH));
```

### `getTransaction`

交易哈希，返回 L2 交易响应对象。

```typescript
async getTransaction(hash: string | Promise<string>): Promise<TransactionResponse>
```

| 名称      | 说明                                    |
| ------- | ------------------------------------- |
| token   | 代币的地址                                 |
| returns | `TransactionResponse` 对象，它允许轻松跟踪交易的状态 |

> Example

```typescript
import { Provider } from "zksync-web3";
const provider = new Provider("https://zksync2-testnet.zksync.dev");

const TX_HASH = "0x95395d90a288b29801c77afbe359774d4fc76c08879b64708c239da8a65dbcf3";
const txHandle = await provider.getTransaction(TX_HASH);

// Wait until the transaction is processed by the server.
await txHandle.wait();
// Wait until the transaction is finalized.
await txHandle.waitFinalize();
```

## `Web3Provider`

应用于 web3 浏览器钱包集成的类，适用于与 Metamask、WalletConnect 和其他流行的浏览器钱包轻松兼容。

### Creating `Web3Provider`

与 `Provider` 类的构造函数的主要区别在于它接受 `ExternalProvider` 而不是节点 URL。

```typescript
constructor(provider: ExternalProvider, network?: ethers.providers.Networkish)
```

#### 输入和输出

| 名称                 | 说明                                                                          |
| ------------------ | --------------------------------------------------------------------------- |
| provider           | `ethers.providers.ExternalProvider` 类实例。 例如，对于 Metamask，它是“window.ethereum” |
| network (optional) | 网络的描述                                                                       |
| returns            | `Provider` 对象                                                               |

> Example

```typescript
import { Web3Provider } from "zksync-web3";

const provider = new Web3Provider(window.ethereum);
```

### 获取 zkSync 签名者

返回可用于签署 zkSync 交易的 `Signer` 对象。 有关 `Signer` 类的更多详细信息，请参见下一个 [部分](./accounts.md#signer)。

#### 输入和输出

| 名称      | 说明                       |
| ------- | ------------------------ |
| returns | `Signer` class object类对象 |

> Example

```typescript
import { Web3Provider } from "zksync-web3";

const provider = new Web3Provider(window.ethereum);
const signer = provider.getSigner();
```
