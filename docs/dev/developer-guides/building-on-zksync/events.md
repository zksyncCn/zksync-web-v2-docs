# 处理事件

## 概述

鉴于智能合约本身无法读取事件，事件是一种向区块链外的听众发布信息的机制。

区块链在设计上是公开的，因此向公众提供所有信息，任何行为都可以通过仔细查看交易来发现。事件是让外部系统轻松获取特定信息的捷径；他们让 dApps 跟踪并响应智能合约发生的事情。它们也可以被搜索，因为它们是可索引的。因此，您应该在任何时候在您的智能合约中发生一些区块链之外的系统应该知道的事件时发出事件，以便外部系统可以侦听此类事件。
事件包含在包含原始交易的同一块的交易日志中。

在 zkSync 中，事件的行为方式与以太坊中的相同。

## 事件过滤

过滤用于查询索引数据并在不需要在链上访问数据时提供成本较低的数据存储。

这些可以与 [Provider Events API](https://docs.ethers.io/v5/api/providers/provider/#Provider--event-methods) 和 [Contract Events API](https ://docs.ethers.io/v5/api/contract/contract/#Contract--events）。

```solidity
    abi = [
  "event Transfer(address indexed src, address indexed dst, uint val)"
];

contract = new Contract(tokenAddress, abi, provider);

// List all token transfers *to* myAddress:
contract.filters.Transfer(null, myAddress)
//{
//    "jsonrpc":"2.0",
//   "id":1,
//    "method":"eth_getLogs",
//    "params":[{
//       "fromBlock": "0x8e8e6",
//        "toBlock": "0x8e8e6",
//        "topics": [["0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"]]
//    }]
//}
```

zkSync 每个响应有 10K 的日志限制。
