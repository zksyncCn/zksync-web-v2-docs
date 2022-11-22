# 入门

[Hardhat](https://hardhat.org) 是一个以太坊开发环境，专为在 Solidity 中轻松开发智能合约而设计。它最突出的特性之一是可扩展性：您可以轻松地将新插件添加到您的 hardhat 项目中。

zkSync 为 Hardhat 提供了三个插件：

- [@matterlabs/hardhat-zksync-solc](./plugins.md#matterlabs-hardhat-zksync-solc) - 用于编译用 Solidity 编写的合约。
- [@matterlabs/hardhat-zksync-vyper](./plugins.md#matterlabs-hardhat-zksync-vyper) - 用于编译用 Vyper 编写的合约。
- [@matterlabs/hardhat-zksync-deploy](./plugins.md#matterlabs-hardhat-zksync-deploy) - 用于部署智能合约。

要了解有关 Hardhat 本身的更多信息，请查看 [its official documentation](https://hardhat.org/getting-started/).

本教程展示了如何使用 Hardhat 从头开始​​设置 zkSync Solidity 项目。
如果您使用的 Vyper，请在 GitHub 查看 [Vyper plugin documentation](./plugins.md#matterlabs-hardhat-zksync-vyper) 或 [this example](https://github.com/matter-labs/hardhat-zksync/tree/main/examples/vyper-example) 。

## 先决条件

对于本教程，必须安装以下程序：

- `yarn` 包管理器。 `npm` 示例将很快添加。
- 一个在 L1 上有足够 Göerli `ETH` 的钱包，用于支付 zkSync 的桥接资金以及部署智能合约。 我们建议使用 [我们来自 zkSync 门户的水龙头](https://portal.zksync.io/faucet)。
1. 要初始化项目并安装依赖项，请在终端中运行以下命令：

```
mkdir greeter-example
cd greeter-example
yarn init -y
yarn add -D typescript ts-node ethers zksync-web3 hardhat @matterlabs/hardhat-zksync-solc @matterlabs/hardhat-zksync-deploy
```

`typescript` 和 `ts-node` 依赖项是可选的——插件可以在 vanilla JavaScript 环境中正常工作。 不过，请注意本教程 _确实_ 使用了 TypeScript。

## 配置

2. 创建 `hardhat.config.ts`文件并将以下代码粘贴到其中：

```typescript
require("@matterlabs/hardhat-zksync-deploy");
require("@matterlabs/hardhat-zksync-solc");

module.exports = {
  zksolc: {
    version: "1.2.0",
    compilerSource: "binary",
    settings: {
      optimizer: {
        enabled: true,
      },
      experimental: {
        dockerImage: "matterlabs/zksolc",
        tag: "v1.2.0",
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

::: 提示

要了解有关“hardhat.config.ts”文件中每个特定属性的更多信息，请查看 [插件文档](./plugins.md)

:::

## 编写和部署合同

3. 创建 `contracts` 和 `deploy` 文件夹。 在 contracts 文件夹中，我们将存储所有智能合约文件。 在 deploy 文件夹中，我们将放置与部署合约相关的所有脚本。

4. 创建 `contracts/Greeter.sol` 合约并粘贴以下代码：

```solidity
//SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.0;

contract Greeter {
    string private greeting;

    constructor(string memory _greeting) {
        greeting = _greeting;
    }

    function greet() public view returns (string memory) {
        return greeting;
    }

    function setGreeting(string memory _greeting) public {
        greeting = _greeting;
    }
}
```

5. 运行 `yarn hardhat compile`，它使用 `hardhat-zksync-solc`插件来编译合约。 `artifacts-zk` 和 `cache-zk` 文件夹将在根目录中创建（而不是常规 Hardhat 的 `artifacts` 和 `cache`）。

::: 提示

请注意，`artifacts-zk` 和 `cache-zk` 文件夹包含编译工件和缓存，不应添加到版本控制中，因此最好将它们包含在项目的 `.gitignore` 文件中。

:::

6. 使用以下代码在 `deploy/deploy.ts` 中创建部署脚本：

```typescript
import { utils, Wallet } from "zksync-web3";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

// An example of a deploy script that will deploy and call a simple contract.
export default async function (hre: HardhatRuntimeEnvironment) {
  console.log(`Running deploy script for the Greeter contract`);

  // Initialize the wallet.
  const wallet = new Wallet("<WALLET-PRIVATE-KEY>");

  // Create deployer object and load the artifact of the contract we want to deploy.
  const deployer = new Deployer(hre, wallet);
  const artifact = await deployer.loadArtifact("Greeter");

  // Deposit some funds to L2 in order to be able to perform L2 transactions.
  const depositAmount = ethers.utils.parseEther("0.001");
  const depositHandle = await deployer.zkWallet.deposit({
    to: deployer.zkWallet.address,
    token: utils.ETH_ADDRESS,
    amount: depositAmount,
  });
  // Wait until the deposit is processed on zkSync
  await depositHandle.wait();

  // Deploy this contract. The returned object will be of a `Contract` type, similarly to ones in `ethers`.
  // `greeting` is an argument for contract constructor.
  const greeting = "Hi there!";
  const greeterContract = await deployer.deploy(artifact, [greeting]);

  // Show the contract info.
  const contractAddress = greeterContract.address;
  console.log(`${artifact.contractName} was deployed to ${contractAddress}`);

  // Call the deployed contract.
  const greetingFromContract = await greeterContract.greet();
  if (greetingFromContract == greeting) {
    console.log(`Contract greets us with ${greeting}!`);
  } else {
    console.error(`Contract said something unexpected: ${greetingFromContract}`);
  }

  // Edit the greeting of the contract
  const newGreeting = "Hey guys";
  const setNewGreetingHandle = await greeterContract.setGreeting(newGreeting);
  await setNewGreetingHandle.wait();

  const newGreetingFromContract = await greeterContract.greet();
  if (newGreetingFromContract == newGreeting) {
    console.log(`Contract greets us with ${newGreeting}!`);
  } else {
    console.error(`Contract said something unexpected: ${newGreetingFromContract}`);
  }
}
```

7. 将 `WALLET-PRIVATE-KEY`文本替换为以太坊钱包的“0x”前缀私钥后，使用以下命令运行脚本：`yarn hardhat deploy-zksync`。 该脚本将：
- 从 Goerli 转 0.001 ETH 到 zkSync。
- 部署带有消息 `你好！`的 `Greeting` 合约。
- 从调用 `greet()` 方法的合约中检索消息。
- 使用 `setGreeting`() 方法更新合约中的问候消息。
- 再次从合约中检索消息。

**恭喜！ 您的 Hardhat 项目现在正在 zkSync 上运行 🎉**

## 了解更多

- 要了解有关 zkSync Hardhat 插件的更多信息，请查看 [插件文档](./plugins)。
- 如果您想了解更多关于如何使用 Javascript 与 zkSync 交互的信息，请查看[zksync-web3 Javascript SDK 文档](../js)。

## 未来版本

未来发布的插件主要有两点改进：:

- **与现有安全帽插件的可组合性** 。与其他安全帽插件的兼容性计划在未来进行，但尚未成为关注重点。
- **改进的跨平台支持。**
