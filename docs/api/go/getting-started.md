# 入门

在指南中，我们将演示如何:

1. 连接到 zkSync 网络。
2. 将资产从以太坊存入zkSync。
3. 转移和提取资金。
4. 部署一个智能合约。
5. 与智能合约进行互动。

## 准备工作

此指南假定你熟悉 [Go](https://go.dev/doc/) 编程语言。
Go的版本应该>=1.17, 并且需要 Go 模块。

## 安装

使用   `go get` 命令来安装SDK，并运行以下命令:

```go
go get github.com/zksync-sdk/zksync2-go
```

## 运行 SDK 实例

要开始使用 SDK，您只需上传一个提供的配置。

使用  `ZkSync Provider`, `EthereumProvider` 和 `Wallet`,您可以通过  zkSync网络执行所有基本操作。

::: warning

⚠️ 切勿将私钥提交给文件追踪历史记录，否则你的账户可能会被泄露。

:::

```go
package main

import (
    "github.com/ethereum/go-ethereum/rpc"
    "github.com/zksync-sdk/zksync2-go"
)

// first, init Ethereum Signer, from your mnemonic, and with the chain Id (in zkSync testnet case, 280)
ethereumSigner, err := zksync2.NewEthSignerFromMnemonic("<mnemonic words>", 280)

// or from raw PrivateKey bytes
ethereumSigner, err = zksync2.NewEthSignerFromRawPrivateKey(pkBytes, 280)

// also, init ZkSync Provider, specify ZkSync2 RPC URL (e.g. testnet)
zkSyncProvider, err := zksync2.NewDefaultProvider("https://zksync2-testnet.zksync.dev")

// then init Wallet, passing just created Ethereum Signer and ZkSync Provider   
wallet, err := zksync2.NewWallet(ethereumSigner, zkSyncProvider)

// init default RPC client to Ethereum node (Goerli network in case of ZkSync2 testnet)
ethRpc, err := rpc.Dial("https://goerli.infura.io/v3/<your_infura_node_id>")

// and use it to create Ethereum Provider by Wallet 
ethereumProvider, err := w.CreateEthereumProvider(ethRpc)
```

## 存入资金

```go
package main

import (
    "github.com/ethereum/go-ethereum/rpc"
    "github.com/zksync-sdk/zksync2-go"
)

func main() {

    tx, err := ep.Deposit(
        zksync2.CreateETH(),
        big.NewInt(1000000000000000), 
        common.HexToAddress("<target_L2_address>"), 
        nil,
    )
    if err != nil {
        panic(err)
    }
    fmt.Println("Tx hash", tx.Hash())

}
```

## 转移

```go
package main

import (
    "github.com/ethereum/go-ethereum/rpc"
    "github.com/zksync-sdk/zksync2-go"
)

func main() {

    hash, err := w.Transfer(
        common.HexToAddress("<target_L2_address>"), 
        big.NewInt(1000000000000),
        nil, 
        nil,
    )
    if err != nil {
        panic(err)
    }
    fmt.Println("Tx hash", hash)

}
```

## 撤销

```go
package main

import (
    "github.com/ethereum/go-ethereum/rpc"
    "github.com/zksync-sdk/zksync2-go"
)

func main() {

    hash, err := w.Withdraw(
        common.HexToAddress("<target_L1_address>"), 
        big.NewInt(1000000000000), 
        nil, 
        nil,
    )
    if err != nil {
        panic(err)
    }
    fmt.Println("Tx hash", hash)

}
```

## 部署智能合约

您可以通过以下方式访问智能合约部署器界面：

```go
    package main

import (
    "github.com/ethereum/go-ethereum/rpc"
    "github.com/zksync-sdk/zksync2-go"
)

func main() {

    hash, err := w.Deploy(bytecode, nil, nil, nil, nil)
    if err != nil {
        panic(err)
    }
    fmt.Println("Tx hash", hash)

    // use helper to get (compute) address of deployed SC
    address, err := zksync2.ComputeL2Create2Address(
        common.HexToAddress("<deployer_L2_address>"), 
        bytecode, 
        nil, 
        nil,
    )
    if err != nil {
        panic(err)
    }
    fmt.Println("Deployed address", address.String())
}
```

## 与智能合约交互

SDK 需要知道合约地址及其 ABI，才能完成与智能合约的交互。因此，您需要使用 ABI.Pack() [使用方法](https://github.com/ethereum/go-ethereum/accounts/abi) 加载合约的 ABI，或对数据进行调用编码以执行函数和它的参数。

调用编码数据的示例：:

```go
package main

import (
    "github.com/ethereum/go-ethereum/rpc"
    "github.com/zksync-sdk/zksync2-go"
)

func main() {

    calldata := crypto.Keccak256([]byte("get()"))[:4]
    hash, err := w.Execute(
        common.HexToAddress("<contract_address>"),
        calldata,
        nil,
    )
    if err != nil {
        panic(err)
    }
    fmt.Println("Tx hash", hash)

}
```

::: 注意

⚠️ 这一部分文档仍在更新中，不久将会更新更详细的信息。

:::