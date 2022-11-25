# 跨链治理

本教程作为如何实现 L1 到 L2 合约交互的示例。本教程中实现了以下功能：

- 在 zkSync 上部署了一个“计数器”（counter）智能合约，它存储了一个可以通过调用 `increment` 方法递增的数字。
- “治理”（governance）智能合约部署在 L1，它有权在 zkSync 上增加计数器。

## 准备工作

在本教程中，假设您已经熟悉在 zkSync 上部署智能合约。如果没有，请参考[快速入门教程](../developer-guides/hello-world.md)的第一部分。

还假设您已经有一些使用以太坊的经验。

## 项目结构

由于我们将在 L1 和 L2 上部署合约，因此我们将把这个项目放在两个不同的文件夹中：

- `/L1-governance`：用于 L1 合约和脚本。
- `/L2-counter`：用于 L2 合约和脚本。

所以创建这些文件夹。

::: 提示

请注意，`governance` 项目是默认的 Hardhat 项目，因为它将用于在 L1 部署合约，而  `counter` 项目包括所有 zkSync 依赖项和具体配置，因为它会在 L2 部署合约。

:::

## L1 治理

要初始化 `/L1-governance` 文件夹内的项目，请运行 `npx hardhat init` 并选择"Create a Typescript project"选项。

要使用 Solidity 与 zkSync 桥接合约进行交互，您需要使用 zkSync 合约接口。有两种选择：

1. 从 `@matterlabs/zksync-contracts` 包中导入。（首选）
2. 从 [contracts repo](https://github.com/matter-labs/v2-testnet-contracts) 下载。

我们将使用第一种方式，通过运行以下命令来安装 `@matterlabs/zksync-contracts` 包（只要确保您在 `/L1-governance` 文件夹中）：

```
yarn add -D @matterlabs/zksync-contracts
```

我们部署在 L1 的治理合约代码如下：

```sol
//SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.0;

import "@matterlabs/zksync-contracts/l1/contracts/zksync/interfaces/IZkSync.sol";

contract Governance {
    address public governor;

    constructor() {
        governor = msg.sender;
    }

    function callZkSync(
        address zkSyncAddress,
        address contractAddr,
        bytes memory data,
        uint64 ergsLimit
    ) external payable {
        require(msg.sender == governor, "Only governor is allowed");

        IZkSync zksync = IZkSync(zkSyncAddress);
        zksync.requestL2Transaction{value: msg.value}(contractAddr, 0, data, ergsLimit, new bytes[](0));
    }
}
```

这是一个非常简单的治理合约。它将合约的创建者设置为单一治理者，并具有向 zkSync 智能合约发送调用的功能。

### 部署 L1 治理合约

尽管本教程并不关注在 L1 上部署合约的过程，但我们将为您提供有关如何继续的快速概述。

1. 您需要一个 Göerli 测试网的 RPC 节点端口来提交部署交易。您可以[在这里找到多个节点提供商](https://github.com/arddluma/awesome-list-rpc-nodes-providers)。

2. 创建文件 `/L1-governance/goerli.json` 并填写以下值：

```json
{
  "nodeUrl": "", // your Goerli Ethereum node  URL.
  "deployerPrivateKey": "" //private key of the wallet that will deploy the governance smart contract. It needs to have some ETH on Göerli.
}
```

3. 添加 Göerli 网络部分到 `hardhat.config.ts` 文件中：

```ts
import { HardhatUserConfig, task } from "hardhat/config";
import "@nomiclabs/hardhat-etherscan";
import "@nomiclabs/hardhat-waffle";
import "@typechain/hardhat";

// import file with Goerli params
const goerli = require('./goerli.json');

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.4",
  networks: {
    // Göerli network
    goerli: {
      url: goerli.nodeUrl,
      accounts: [goerli.deployerPrivateKey]
    },
  }
}
```

4. 使用以下代码创建部署脚本 `/L1-governance/scripts/deploy.ts`：

```ts
// We require the Hardhat Runtime Environment explicitly here. This is optional
// but useful for running the script in a standalone fashion through `node <script>`.
//
// When running the script with `npx hardhat run <script>` you'll find the Hardhat
// Runtime Environment's members available in the global scope.
import { ethers } from "hardhat";

async function main() {
  // We get the contract to deploy
  const Governance = await ethers.getContractFactory("Governance");

  const contract = await Governance.deploy();
  await contract.deployed();

  console.log(`Governance contract was successfully deployed at ${contract.address}`);
}

// We recommend this pattern to be able to use async/await everywhere
// and properly handle errors.
main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

5. 编译合约并运行部署脚本：

```
# compile contract
yarn hardhat compile

# deploy contract
yarn hardhat run --network goerli ./scripts/deploy.ts
```

最后一条命令将输出已部署的治理智能合约地址。

## L2 计数器

现在我们已经有了 L1 治理合约地址，让我们继续在 L2 上部署计数器合约。

1. 要初始化 `/L2-counter` 文件夹中的项目，请运行以下命令：

```
yarn init -y
# install all dependencies
yarn add -D typescript ts-node ethers zksync-web3 hardhat @matterlabs/hardhat-zksync-solc @matterlabs/hardhat-zksync-deploy
```

2. 创建 `hardhat.config.ts` 文件并粘贴以下代码：

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

如果您的默认网络不是 `hardhat`，请确保`zksync: true`也在其配置中。

3. 创建 `contracts` 和 `deploy` 文件夹。前者是所有 `*.sol` 合约文件存储的地方，后者是所有与部署合约相关脚本放置的地方。

4. 创建 `contracts/Counter.sol` 合约文件。该合约将拥有部署在 L1 的治理合约地址和一个计数器。增加计数器的功能只能由治理合约调用。代码如下：

```sol
//SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.0;

contract Counter {
    uint256 public value = 0;
    address public governance;

    constructor(address newGovernance) {
        governance = newGovernance;
    }

    function increment() public {
        require(msg.sender == governance, "Only governance is allowed");

        value += 1;
    }
}
```

5. 使用以下命令编译合约：

```
yarn hardhat compile
```

6. 在 `deploy/deploy.ts` 中创建部署脚本：

```typescript
import { utils, Wallet } from "zksync-web3";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

// Insert the address of the governance contract
const GOVERNANCE_ADDRESS = "<GOVERNANCE-ADDRESS>";

// An example of a deploy script that will deploy and call a simple contract.
export default async function (hre: HardhatRuntimeEnvironment) {
  console.log(`Running deploy script for the Counter contract`);

  // Initialize the wallet.
  const wallet = new Wallet("<WALLET-PRIVATE-KEY>");

  // Create deployer object and load the artifact of the contract you want to deploy.
  const deployer = new Deployer(hre, wallet);
  const artifact = await deployer.loadArtifact("Counter");

  // Deposit some funds to L2 to be able to perform deposits.
  const deploymentFee = await deployer.estimateDeployFee(artifact, [
    GOVERNANCE_ADDRESS,
  ]);
  const depositHandle = await deployer.zkWallet.deposit({
    to: deployer.zkWallet.address,
    token: utils.ETH_ADDRESS,
    amount: deploymentFee.mul(2),
  });
  // Wait until the deposit is processed on zkSync
  await depositHandle.wait();

  // Deploy this contract. The returned object will be of a `Contract` type, similar to the ones in `ethers`.
  // The address of the governance is an argument for contract constructor.
  const counterContract = await deployer.deploy(artifact, [GOVERNANCE_ADDRESS]);

  // Show the contract info.
  const contractAddress = counterContract.address;
  console.log(`${artifact.contractName} was deployed to ${contractAddress}`);
}
```

7. 将 `<WALLET-PRIVATE-KEY>` 和 `<GOVERNANCE-ADDRESS>` 分别替换为 Göerli 上带有一些 ETH 余额的以太坊钱包 `0x` 前缀私钥，以及 L1 治理合约的地址后，使用以下命令运行脚本 ：

```
yarn hardhat deploy-zksync
```

在输出中，您应该看到部署合约的地址。

::: 提示

您可以在[快速入门教程](../developer-guides/hello-world.md)或 zkSync [hardhat 插件](../../api/hardhat/getting-started.md)文档中找到有关部署合约的更多具体细节。

:::

## 读取计数器值

部署两个合约后，我们可以创建一个小脚本来检索计数器的值。为了简单起见，我们将在 `/L2-counter` 文件夹中创建此脚本。为了保持教程的通用性，hardhat 特有功能将不会使用。

### 获取计数器合约的 ABI

以下是获取计数器合约 ABI 的方法：

1. 从位于 `/L2-counter/artifacts-zk/contracts/Counter.sol/Counter.json` 的编译文件中复制 `abi` 数组。

2. 在 `/L2-counter` 项目文件夹中创建 `scripts` 文件夹。

3. 创建一个新文件 `/L2-counter/scripts/counter.json` 并粘贴计数器合约的 ABI。

4. 创建 `/L2-counter/scripts/display-value.ts` 文件并在其中粘贴以下代码：

```ts
import { Contract, Provider, Wallet } from "zksync-web3";

// The address of the counter smart contract
const COUNTER_ADDRESS = "<COUNTER-ADDRESS>";
// The ABI of the counter smart contract
const COUNTER_ABI = require("./counter.json");

async function main() {
  // Initializing the zkSync provider
  const l2Provider = new Provider("https://zksync2-testnet.zksync.dev");

  const counterContract = new Contract(COUNTER_ADDRESS, COUNTER_ABI, l2Provider);

  console.log(`The counter value is ${(await counterContract.value()).toString()}`);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

该代码相对简单，基本等同于它与 `ethers` 的工作方式。它只会从 L2 合约中检索计数器值。

将 `<COUNTER-ADDRESS>` 替换为部署的计数器合约地址后，运行此脚本

```
yarn ts-node ./scripts/display-value.ts
```

输出应该是：

```
The counter value is 0
```

## 从 L1 调用 L2 合约

现在，让我们从 L1 调用 `increment` 方法。

1. 获取编译后的治理合约 ABI 数组，其位于 `/L1-governance/artifacts/contracts/Governance.sol/Governance.json`，将其保存为 `/L2-counter/scripts/governance.json` 的新文件（确保在 /L2-counter 文件夹中创建它！）。
2. 创建 `L2-counter/scripts/increment-counter.ts` 文件并为脚本粘贴以下模板：

```ts
// Imports and constants will be put here

async function main() {
  // The logic will be put here
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

4. 为了与治理智能合约交互，我们需要初始化一个以太坊提供者（provider）和相应的 `ethers` `Contract` 对象，所以我们需要有它被部署到的地址：

```ts
// Imports
import { BigNumber, Contract, ethers, Wallet } from "ethers";

const GOVERNANCE_ABI = require("./governance.json");
const GOVERNANCE_ADDRESS = "<GOVERNANCE-ADDRESS>";
```

```ts
async function main() {
  // Ethereum L1 provider
  const l1Provider = ethers.providers.getDefaultProvider("goerli");

  // Governor wallet, the same one as the one that deployed the
  // governance contract
  const wallet = new ethers.Wallet("<WALLET-PRIVATE-KEY>", l1Provider);

  const govcontract = new Contract(GOVERNANCE_ADDRESS, GOVERNANCE_ABI, wallet);
}
```

将 `<GOVERNANCE-ADDRESS>` 和 `<WALLET-PRIVATE-KEY>` 分别替换为 L1 治理智能合约的地址，以及部署治理合约的钱包私钥。

5. 要与 zkSync 桥交互，我们需要它的 L1 地址。虽然在主网上您可能希望将 zkSync 智能合约的地址设置为环境变量或常量，但值得注意的是您可以动态获取智能合约地址。如果您在测试网上工作，我们建议您使用这种方法，因为可能会发生重新生成并且合约地址可能会发生变化。

```ts
// Imports
import { Provider, utils } from "zksync-web3";
```

```ts
async function main() {
  // ... Previous steps

  // Initializing the L2 privider
  const l2Provider = new Provider("https://zksync2-testnet.zksync.dev");
  // Getting the current address of the zkSync L1 bridge
  const zkSyncAddress = await l2Provider.getMainContractAddress();
  // Getting the `Contract` object of the zkSync bridge
  const zkSyncContract = new Contract(zkSyncAddress, utils.ZKSYNC_MAIN_ABI, wallet);
}
```

6. 从 L1 执行交易需要调用者向 L2 运营者支付一些费用。

首先，此费用取决于 calldata 的长度和 `ergsLimit`。如果您不熟悉这个概念，那么它与以太坊上的 `gasLimit` 几乎一样。您可以在此处阅读有关 [zkSync 费用机制](../developer-guides/transactions/fee-model.md)的更多信息。

其次，费用取决于交易调用期间使用的 gas 价格。 因此，为了获得可预测的调用费用，应该获取 gas 价格并使用获得的值。

```ts
// Imports
const COUNTER_ABI = require("./counter.json");
```

```ts
async function main() {
  // ... Previous steps

  // Encoding L1 transaction is the same way it is done on Ethereum.
  const counterInterface = new ethers.utils.Interface(COUNTER_ABI);
  const data = counterInterface.encodeFunctionData("increment", []);

  // The price of L1 transaction requests depend on the gas price used in the call,
  // so we should explicitly fetch the gas price before the call.
  const gasPrice = await l1Provider.getGasPrice();

  // Here we define the constant for ergs limit.
  // There is currently no way to get the exact ergsLimit required for an L1->L2 tx.
  // You can read more on that in the tip below
  const ergsLimit = BigNumber.from(100000);

  // Getting the cost of the execution in Wei.
  const baseCost = await zkSyncContract.l2TransactionBaseCost(gasPrice, ergsLimit, ethers.utils.hexlify(data).length);
}
```

::: 提示 费用机制和费用估算当前仍在开发中

您可能已经注意到 L1->L2 交易中缺少 `ergs_per_pubdata` 和 `ergs_per_storage` 字段。这些对于协议的安全性肯定很重要，并且很快就会添加。请注意，这将是合约接口的重大更改。

此外，目前还没有简单的方法来评估执行 L1->L2 交易所需的 `ergs` 准确数量。在撰写本文时，即使提供的 `ergsLimit` 为 `0`，也可能会处理交易。这将在未来发生变化。

:::

7. 现在可以调用治理合约，这将使调用重定向到 zkSync：

```ts
// Imports
const COUNTER_ADDRESS = "<COUNTER-ADDRESS>";
```

```ts
async function main() {
  // ... Previous steps

  // Calling the L1 governance contract.
  const tx = await govcontract.callZkSync(zkSyncAddress, COUNTER_ADDRESS, data, ergsLimit, {
    // Passing the necessary ETH `value` to cover the fee for the operation
    value: baseCost,
    gasPrice,
  });

  // Waiting until the L1 transaction is complete.
  await tx.wait();
}
```

确保将 `<COUNTER-ADDRESS>` 替换为 L2 计数器合约的地址。

8. 您可以跟踪相应 L2 交易的状态。`zksync-web3` 的 `Provider` 有一个方法，给定调用 zkSync 桥的交易的 L1 `ethers.TransactionResponse` 对象，返回 L2 交易的对应 `TransactionResponse` 对象，让其可以方便地等待在 L2 上被处理的交易。

```ts
async function main() {
  // ... Previous steps

  // Getting the TransactionResponse object for the L2 transaction corresponding to the
  // execution call
  const l2Response = await l2Provider.getL2TransactionFromPriorityOp(tx);

  // The receipt of the L2 transaction corresponding to the call to the counter contract
  const l2Receipt = await l2Response.wait();
  console.log(l2Receipt);
}
```

### 完整代码

以下是获取 zkSync 合约地址，对交易数据进行编码，计算费用，将交易发送到 L1 并在 L2 中跟踪对应交易的完整代码：

```ts
import { BigNumber, Contract, ethers, Wallet } from "ethers";
import { Provider, utils } from "zksync-web3";

const GOVERNANCE_ABI = require("./governance.json");
const GOVERNANCE_ADDRESS = "<GOVERNANCE-ADDRESS>";
const COUNTER_ABI = require("./counter.json");
const COUNTER_ADDRESS = "<COUNTER-ADDRESS>";

async function main() {
  // Ethereum L1 provider
  const l1Provider = ethers.providers.getDefaultProvider("goerli");

  // Governor wallet
  const wallet = new Wallet("<WALLET-PRIVATE-KEY>", l1Provider);

  const govcontract = new Contract(GOVERNANCE_ADDRESS, GOVERNANCE_ABI, wallet);

  // Getting the current address of the zkSync L1 bridge
  const l2Provider = new Provider("https://zksync2-testnet.zksync.dev");
  const zkSyncAddress = await l2Provider.getMainContractAddress();
  // Getting the `Contract` object of the zkSync bridge
  const zkSyncContract = new Contract(zkSyncAddress, utils.ZKSYNC_MAIN_ABI, wallet);

  // Encoding the tx data the same way it is done on Ethereum.
  const counterInterface = new ethers.utils.Interface(COUNTER_ABI);
  const data = counterInterface.encodeFunctionData("increment", []);

  // The price of the L1 transaction requests depends on the gas price used in the call
  const gasPrice = await l1Provider.getGasPrice();

  // Here we define the constant for ergs limit.
  const ergsLimit = BigNumber.from(100000);
  // Getting the cost of the execution.
  const baseCost = await zkSyncContract.l2TransactionBaseCost(gasPrice, ergsLimit, ethers.utils.hexlify(data).length);

  // Calling the L1 governance contract.
  const tx = await govcontract.callZkSync(zkSyncAddress, COUNTER_ADDRESS, data, ergsLimit, {
    // Passing the necessary ETH `value` to cover the fee for the operation
    value: baseCost,
    gasPrice,
  });

  // Waiting until the L1 tx is complete.
  await tx.wait();

  // Getting the TransactionResponse object for the L2 transaction corresponding to the
  // execution call
  const l2Response = await l2Provider.getL2TransactionFromPriorityOp(tx);

  // The receipt of the L2 transaction corresponding to the call to the Increment contract
  const l2Receipt = await l2Response.wait();
  console.log(l2Receipt);
}

// We recommend this pattern to be able to use async/await everywhere
// and properly handle errors.
main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

您可以使用以下命令运行脚本：

```
yarn ts-node ./scripts/increment-counter.ts
```

在输出中，您应该在 L2 中看到完整的交易收据。您可以获取 `transactionHash` 并在 [zkSync 浏览器](https://explorer.zksync.io/)中对其进行跟踪。

9. 之后，您可以通过再次运行 `display-value` 脚本来验证交易确实成功了：

```
yarn ts-node ./scripts/display-value.ts
```

L2 合约中的计数器应该在交易后增加，所以输出应该是：

```
The counter value is 1
```

## 完整项目

您可以在[这里](https://github.com/matter-labs/cross-chain-tutorial)下载完整项目。

## 学到更多

- 要了解有关 zkSync L1->L2 交互的更多信息，请查看此[文档](../developer-guides/bridging/l1-l2.md)。
- 要了解有关 `zksync-web3` SDK的更多信息，请查看此[文档](../../api/js)。
- 要了解有关 zkSync hardhat 插件的更多信息，请查看此[文档](../../api/hardhat)。
