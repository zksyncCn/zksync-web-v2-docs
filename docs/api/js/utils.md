# 实用程序

`zksync-web3` 为 zkSync 构建器提供了一些有用的实用程序。 它们可以通过以下方式导入：

```typescript
import { utils } from "zksync-web3";
```

大多数实用程序由 zkSync 团队在内部使用。 因此，本文档将仅描述那些对您有帮助的内容。

## ETH `地址`

虽然正式的 ETH 是 zkSync 上地址为`0x00000000000000000000000000000000000800a`的代币，但我们使用“零地址”作为对用户更友好的别名：

```typescript
export const ETH_ADDRESS = "0x0000000000000000000000000000000000000000";
```

## zkSync 智能合约的 ABI

```typescript
export const ZKSYNC_MAIN_ABI = new utils.Interface(require("../abi/ZkSync.json"));
```

## IERC20 接口

在 zkSync 上与原生代币交互时很方便。

```typescript
export const IERC20 = new utils.Interface(require("../abi/IERC20.json"));
```

## 编码 paymaster 参数

为常见的 [paymaster 流程](../../dev/developer-guides/aa.md#built-in-paymaster-flows) 返回正确格式 `paymasterParams`对象的实用方法。

```typescript
export function getPaymasterParams(paymasterAddress: Address, paymasterInput: PaymasterInput): PaymasterParams;
```

`PaymasterInput` 的定义可以在 [这里](./types.md) 找到。

## 默认 pubdata 价格限制

目前，没有方法可以准确估计所需的 ergsPerPubdataLimit。 这就是为什么现在强烈建议提供 `DEFAULT_ERGS_PER_PUBDATA_LIMIT`。 提供它不会向用户收取更多费用。
稍后可以查询当前推荐的限制。

```typescript
const GAS_PER_PUBDATA_BYTE = 16;
export const DEFAULT_ERGS_PER_PUBDATA_LIMIT = GAS_PER_PUBDATA_BYTE * 10_000;
```
