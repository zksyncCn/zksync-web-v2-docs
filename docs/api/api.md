# Web3 API

Web3 API

zkSync 2.0 完全支持标准 [Ethereum JSON-RPC API](https://ethereum.org/en/developers/docs/apis/json-rpc/) ，并添加了一些特定于 L2 的功能。

只要代码不涉及部署新的智能合约（它们只能使用 EIP712 交易进行部署，更多关于[下面]（#eip712）），_不需要对代码库进行任何更改。_ 可以继续使用当前正在使用的 SDK。 用户将继续以 ETH 支付费用，用户体验将与以太坊相同。

然而，zkSync 有其细节，本节将对此进行描述。

## EIP712

要指定其他字段，例如自定义帐户的自定义签名或选择付款人，应使用 EIP712 交易。 这些交易具有与标准以太坊交易相同的字段，但它们也有包含额外的 L2 特定数据（`paymaster` 等）的字段。

```json
"ergsPerPubdata": "1212",
"customSignature": "0x...",
"paymasterParams": {
  "paymaster": "0x...",
  "paymasterInput": "0x..."
},
"factory_deps": ["0x..."]
```

- `ergsPerPubdata`：是一个描述用户愿意为单个字节的 pubdata 支付的最大 erg 数量的字段。
- `customSignature` 是带有自定义签名的字段，以防签名者的帐户不是 EOA。
- `paymasterParams` 是一个带有参数的字段，用于为交易配置自定义 paymaster。 paymaster 的地址和调用它的编码输入在 paymaster 参数中。
- `factory_deps` 是一个字段，应该是用于部署事务的非空 `bytes` 数组。它应该包含正在部署的合约的字节码。如果部署的合约是工厂合约，即它可以部署其他合约，则该数组还应包含它可以部署的合约的字节码。

为了让服务器识别 EIP712 交易，`transaction_type` 字段等于 `113`（不幸的是，数字 `712` 不能用作 `transaction_type`，因为类型必须是一个字节长）。

用户没有签署 RLP 编码的交易，而是签署了以下类型的 EIP712 结构：

| Field name              | Type        |
| ----------------------- | ----------- |
| txType                  | `uint256`   |
| from                    | `uint256`   |
| to                      | `uint256`   |
| ergsLimit               | `uint256`   |
| ergsPerPubdataByteLimit | `uint256`   |
| maxFeePerErg            | `uint256 `  |
| maxPriorityFeePerErg    | `uint256`   |
| paymaster               | `uint256`   |
| nonce                   | `uint256`   |
| value                   | `uint256`   |
| data                    | `bytes`     |
| factoryDeps             | `bytes32[]` |
| paymasterInput          | `bytes`     |

我们的 [SDK](./js/features.md) 可以方便地处理这些字段。

## zkSync 特定的 JSON-RPC 方法

所有 zkSync 特定的方法都位于 `zks_` 命名空间中。 API 还可能提供此处以外的方法。 这些方法是给团队内部使用的，非常不稳定。

：：： 警告

请注意，Metamask 不支持 zks_ 命名空间的方法，我们正在努力在未来支持它，或者，您可以将 `Provider` 类与测试网 RPC 一起使用，而不是依赖 Metamask 的注入提供程序。

:::

<!-- ### `zks_estimateFee`

Returns the fee for the transaction. The token in which the fee is calculated is returned based on the `fee_token` in the transaction provided.

#### Input parameters

| Parameter | Type          | Description                                                  |
| --------- | ------------- | ------------------------------------------------------------ |
| req       | `CallRequest` | The zkSync transaction for which the fee should be estimated |

#### Output format

```json
{
  "ergs_limit": 100000000,
  "ergs_price_limit": 10000,
  "fee_token": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
  "ergs_per_storage_limit": 100,
  "ergs_per_pubdata_limit": 10
}
``` -->

### `zks_getMainContract`

返回 zkSync 合约的地址。

### 输入参数

没有。

### 输出的格式

`"0xaBEA9132b05A70803a4E85094fD0e1800777fBEF"`

### `zks_L1ChainId`

返回底层 L1 的链 ID。

### 输入的参数

没有。

### 输出的格式

`12`

### `zks_getConfirmedTokens`

给定 `from` 和 `limit` 返回有关 ID 在区间 `[from..from+limit-1]` 中的已确认令牌的信息。 “确认”在这里是错误的词，因为确认令牌已经通过默认的 zkSync 桥接。

令牌按其符号的字母顺序返回，因此令牌的 id 只是它在已按符号排序的令牌数组中的位置。

### 输入参数

| Parameter | Type     | Description         |
| --------- | -------- | ------------------- |
| from      | `uint32` | 从中开始返回有关代币信息的代币 ID。 |
| limit     | `uint8`  | 从 API 返回的代币数。       |

### 输出格式

```json
[
  {
    "address": "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
    "decimals": 18,
    "name": "ETH",
    "symbol": "ETH"
  },
  {
    "address": "0xd2255612f9b045e9c81244bb874abb413ca139a3",
    "decimals": 18,
    "name": "TrueUSD",
    "symbol": "TUSD"
  },
  {
    "address": "0xd35cceead182dcee0f148ebac9447da2c4d449c4",
    "decimals": 6,
    "name": "USD Coin (goerli)",
    "symbol": "USDC"
  }
]
```
### `zks_getL2ToL1LogProof`

给定一个区块、一个发件人和一个消息，以及包含L1->L2消息的区块中可选的消息日志索引，返回通过L1Messenger系统合同发送的消息的证明。

### Input parameters

| Parameter       | Type      | Description                    |
| --------------- | --------- | ------------------------------ |
| block           | `uint32`  | 发出该消息的区块编号。                    |
| sender          | `address` | 消息的发件人（即调用L1Messenger系统合同的账户）。 |
| msg             | `uint256` | 发送信息的keccak256哈希值。             |
| l2_log_position | `uint256` | `null`                         |

The index of the log that can be obtained from the transaction receipt (it includes a list of every log produced by the transaction).

### Input parameters

| Parameter | Type      | Description                                                                               |
| --------- | --------- | ----------------------------------------------------------------------------------------- |
| tx_hash   | `bytes32` | Hash of the L2 transaction the L2 to L1 log was produced within.                                   |
| l2_to_l1_log_index| `undefined` &#124; `number` | The Index of the L2 to L1 log in the transaction. |

### 输出格式

如果没有这样的消息，则返回值为`null`。

否则，返回如下格式的对象：

```json
{
  "id": 0,
  "proof": [
    "0x66687aadf862bd776c8fc18b8e9f8e20089714856ee233b3902a591d0d5f2925",
    "0x2eeb74a6177f588d80c0c752b99556902ddf9682d0b906f5aa2adbaf8466a4e9",
    "0x1223349a40d2ee10bd1bebb5889ef8018c8bc13359ed94b387810af96c6e4268",
    "0x5b82b695a7ac2668e188b75f7d4fa79faa504117d1fdfcbe8a46915c1a8a5191"
  ],
  "root": "0x6a420705824f0a3a7e541994bc15e14e6a8991cd4e4b2d35c66f6e7647760d97"
}
```

`id` 是该块的 L2->L1 消息的 Merkle 树中叶子的位置。 `proof` 是消息的 Merkle 证明，而 `root` 是 L2->L1 消息的 Merkle 树的根。 请注意，Merkle 树使用 _sha256_ 作为树。

您无需关心内在函数，因为返回的 `id` 和 `proof` 可以立即用于与 zkSync 智能合约交互。

可以在 [此处](../dev/developer-guides/bridging/l2-l1.md) 找到通过我们的 SDK 使用此端点的一个很好的示例。


::: tip

The list of L2 to L1 logs produced by the transaction, which is included in the receipts, is a combination of logs produced by L1Messenger contract or other system contracts/bootloader. 

There is a log produced by the bootloader for every L1 originated transaction that shows if the transaction has succeeded. 

:::

### `zks_getL2ToL1MsgProof`

Given a block, a sender, a message, and an optional message log index in the block containing the L1->L2 message, it returns the proof for the message sent via the L1Messenger system contract.

### Input parameters

| Parameter | Type      | Description                                                                               |
| --------- | --------- | ----------------------------------------------------------------------------------------- |
| block     | `uint32`  | The number of the block where the message was emitted.                                    |
| sender    | `address` | The sender of the message (i.e. the account that called the L1Messenger system contract). |
| msg       | `bytes32` | The keccak256 hash of the sent message.                                          |
| l2_log_position       | `uint256` &#124; `null` | The index in the block of the event that was emitted by the [L1Messenger](../dev/developer-guides/contracts/system-contracts.md#il1messenger) when submitting this message. If it is ommitted, the proof for the first message with such content will be returned.                                          |

### Output format

The same as in [zks_getL2ToL1LogProof](#output-format-4).

::: warning

`zks_getL2ToL1MsgProof` endpoint will be deprecated because proofs for L2 to L1 messages can also be fetched from `zks_getL2ToL1LogProof`.

:::

### `zks_getBridgeContracts`

返回默认网桥的 L1/L2 地址。

### 输入参数

没有。

### 输出格式

```json
{
  "l1Erc20DefaultBridge": "0x7786255495348c08f82c09c82352019fade3bf29",
  "l1EthDefaultBridge": "0xcbebcd41ceabbc85da9bb67527f58d69ad4dfff5",
  "l2Erc20DefaultBridge": "0x92131f10c54f9b251a5deaf3c05815f7659bbe02",
  "l2EthDefaultBridge": "0x2c5d8a991f399089f728f1ae40bd0b11acd0fb62"
}
```

### `zks_getTestnetPaymaster`

返回[testnet paymaster](.../dev/developer-guides/aa.md#testnet-paymaster)的地址：在testnets上可用的paymaster，可以用ERC-20兼容代币支付费用。

### 输入参数

没有。

### 输出格式

```json
"0x7786255495348c08f82c09c82352019fade3bf29"
```

<!-- TODO: uncomment once fixed --->

<!-- ### `zks_getTokenPrice`

Given a token address, returns its price in USD. Please note that that this is the price that is used by the zkSync team and can be a bit different from the current market price. On testnets, token prices can be very different from the actual market price.

### Input parameters

| Parameter | Type   | Description              |
| --------- | ------ | ------------------------ |
| address   | `address` | The address of the token. |

### Output format

`3500.0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000` -->

<!--

#[rpc(name = "zks_getConfirmedTokens", returns = "Vec<Token>")]
fn get_confirmed_tokens(&self, from: u32, limit: u8) -> BoxFutureResult<Vec<Token>>;

#[rpc(name = "zks_isTokenLiquid", returns = "bool")]
fn is_token_liquid(&self, token_address: Address) -> BoxFutureResult<bool>;

#[rpc(name = "zks_getTokenPrice", returns = "BigDecimal")]
fn get_token_price(&self, token_address: Address) -> BoxFutureResult<BigDecimal>;

#[rpc(name = "zks_setContractDebugInfo", returns = "bool")]
fn set_contract_debug_info(
    &self,
    contract_address: Address,
    info: ContractSourceDebugInfo,
) -> BoxFutureResult<bool>;

#[rpc(name = "zks_getContractDebugInfo", returns = "ContractSourceDebugInfo")]
fn get_contract_debug_info(
    &self,
    contract_address: Address,
) -> BoxFutureResult<Option<ContractSourceDebugInfo>>;

#[rpc(name = "zks_getTransactionTrace", returns = "Option<VmDebugTrace>")]
fn get_transaction_trace(&self, hash: H256) -> BoxFutureResult<Option<VmDebugTrace>>;





Documented:
#[rpc(name = "zks_estimateFee", returns = "Fee")]
fn estimate_fee(&self, req: CallRequest) -> BoxFutureResult<Fee>;

#[rpc(name = "zks_getMainContract", returns = "Address")]
fn get_main_contract(&self) -> BoxFutureResult<Address>;

#[rpc(name = "zks_L1ChainId", returns = "U64")]
fn l1_chain_id(&self) -> Result<U64>;

#[rpc(name = "zks_getL1WithdrawalTx", returns = "Option<H256>")]
fn get_eth_withdrawal_tx(&self, withdrawal_hash: H256) -> BoxFutureResult<Option<H256>>;



Don't want to document (at least for now):

### `zks_getAccountTransactions`

### Input parameters

| Parameter | Type      | Description                                           |
| --------- | --------- | ----------------------------------------------------- |
| address   | `Address` | The address of the account                            |
| before    | `u32`     | The offset from which to start returning transactions |
| limit     | `u8`      | The maximum number of transactions to be returned     |





-->

## 发布订阅 API

zkSync 与 [Geth 的 pubsub API](https://geth.ethereum.org/docs/rpc/pubsub) 完全兼容，除了 `syncing` 订阅，它对 zkSync 网络没有意义，因为从技术上讲我们的 节点始终同步。

WebSocket URL 是 `wss://zksync2-testnet.zksync.dev/ws`。
