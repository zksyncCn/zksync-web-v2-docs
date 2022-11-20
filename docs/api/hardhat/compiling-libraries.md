# 不可内联库的编译

Solidity库可以分为两类：

- _Inlinable_。那些只包含 `private` or `internal` 方法的。 由于它们永远无法从外部调用，因此 Solidity 编译器将它们内联，并且将这些库的代码成为使用它们代码的一部分，即：不使用外部调用来访问库的方法。
- *Non-inlinable*。 那些至少有一个 `公共` 或 `外部` 方法的。 虽然它们可能会被 Solidity 编译器内联，但在编译为 Yul 表示时，它们不会内联。 由于 Yul 是编译为 zkEVM 字节码的中间步骤，这意味着这些库不能被 zkSync 编译器内联。

**实际上，这意味着需要单独部署具有公共方法的库，并且在编译主合同时将它们的地址作为参数传递。** 使用此库的方法将替换为对其地址的调用。

## OpenZeppelin 公用程序库

请注意， 绝大多数OpenZeppelin 的公用程序库都是可内联的。这意味着不需要做任何进一步的的操作来使它们编译。

这一节只描述了不可内联的库的编译。

## 示例

假设我们有一个计算数字平方的小库:

```solidity
pragma solidity ^0.8.0;

library MiniMath {
    function square(uint256 x) public pure returns (uint256) {
         return x*x;
    }
}
```

而有一个智能合约，使用这个库

```solidity
pragma solidity ^0.8.0;

import "./MiniMath.sol";

contract Main {
    uint256 public lastNumber;

    function storeSquare(uint256 x) public {
        uint256 square = MiniMath.square(x);
        lastNumber = square;
    }
}
```

如果您尝试按照 [入门](./getting-started.md) 指南中的指南使用这两个文件创建项目，`yarn hardhat compile` 命令将失败并出现以下错误：

```
Error in plugin @matterlabs/hardhat-zksync-solc: LLVM("Library `contracts/MiniMath.sol:MiniMath` not found")
```

该错误告诉我们，应该提供 `MiniMath` 库的地址。

要解决此问题，您需要创建 _一个单独的项目_ ，其中仅存放库文件。 _仅_ 将库部署到 zkSync 后，您应该获取已部署库的地址并将其传递给编译器设置。 部署库的过程与部署智能合约的过程相同。 您可以在[入门](./getting-started.md#write-and-deploy-a-contract) 指南中了解如何在 zkSync 上部署智能合约。

假设已部署库的地址是 `0xF9702469Dfb84A9aC171E284F71615bd3D3f1EdC`。 要将此地址传递给编译器参数，请打开`Main` 合约所在项目的`harhdat.config.ts` 文件，并在`zksolc` 插件属性中添加`libraries` 部分：

```typescript
require("@matterlabs/hardhat-zksync-deploy");
require("@matterlabs/hardhat-zksync-solc");

module.exports = {
  zksolc: {
    version: "1.2.0",
    compilerSource: "docker",
    settings: {
      optimizer: {
        enabled: true,
      },
      experimental: {
        dockerImage: "matterlabs/zksolc",
        tag: "v1.2.0",
      },
      libraries: {
        "contracts/MiniMath.sol": {
          MiniMath: "0xF9702469Dfb84A9aC171E284F71615bd3D3f1EdC",
        },
      },
    },
  },
  zkSyncDeploy: {
    zkSyncNetwork: "https://zksync2-testnet.zksync.dev",
    ethNetwork: "goerli", // Can also be the RPC URL of the network (e.g. `https://goerli.infura.io/v3/<API_KEY>`)
  },
  networks: {
    hardhat: {
      zksync: true,
    },
  },
  solidity: {
    version: "0.8.16",
  },
};
```

库的地址在以下几行中传递：

```typescript
libraries: {
  'contracts/MiniMath.sol': {
    'MiniMath': '0xF9702469Dfb84A9aC171E284F71615bd3D3f1EdC'
  }
},
```

其中 `contracts/MiniMath.sol` 是库的 Solidity 文件的位置，`MiniMath` 是库的名称。

现在，运行 `yarn hardhat compile` 应该可以成功编译 `Main` 合约 。
