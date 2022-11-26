# Web3 API

zkSync 完全支持标准的 [Ethereum JSON-RPC API](https://eth.wiki/json-rpc/API)，但它还具有一些额外的 L2 特定功能。

只要代码不涉及部署新的智能合约（它们只能通过 EIP712 交易进行部署，更多内容见 [下文](../../api/api.md#eip712)），您不需要更改代码库。

您可以继续使用您现在使用的 SDK。 UX 将与以太坊上的相同。

为了部署智能合约或为用户启用独特的 zkSync 功能（例如账户抽象），需要使用 EIP712 交易类型。

zkSync JSON-RPC API 的更详细描述可以在 [这里](../../api/api) 找到。
