# 快速指南

在快速入门指南中，您将学习如何将智能合约部署到 zkSync 并构建 dApp，并使用 zkSync 开发工具箱进行交互。

我们需要构建的：

- 存储问候消息并部署在 zkSync 上的智能合约。
- 用于检索问候语的 dApp。
- 用户将能够更改智能合约上的问候语。
- 默认情况下，用户必须支付交易费用才能更改以太坊的问候消息。 但是，我们还将解释如何[实现 testnet paymaster](#paying-fees-using-testnet-paymaster) 以允许用户使用 ERC20 代币支付费用。

::: tip

testnet paymaster 仅用于测试。 如果您决定在主网上构建项目，您应该阅读有关 [paymasters](./aa.md#paymasters) 的文档。

:::

## 准备工作

- `yarn` 包管理器。 [这里是安装指南](https://yarnpkg.com/getting-started/install)(`npm`示例将很快添加。)
- 一个钱包，在 L1 上有足够的 Göerli `ETH` 来支付桥接资金到 zkSync 以及部署智能合约。 如果要实现测试网 paymaster，则需要 zkSync 上的 ERC20 代币。 我们推荐使用 [来自 zkSync 门户的水龙头](https://portal.zksync.io/faucet)。

## 初始化项目并部署智能合约

1. 初始化项目并安装依赖。 在终端中运行以下命令：

```
mkdir greeter-example
cd greeter-example
yarn init -y
yarn add -D typescript ts-node ethers zksync-web3 hardhat @matterlabs/hardhat-zksync-solc @matterlabs/hardhat-zksync-deploy
```

请注意，zkSync 插件需要 Typescript。

2. 创建 `hardhat.config.ts` 文件并在其中粘贴以下代码：

```typescript
require("@matterlabs/hardhat-zksync-deploy");
require("@matterlabs/hardhat-zksync-solc");

module.exports = {
  zksolc: {
    version: "1.2.0",
    compilerSource: "binary",
    settings: {
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

::: warning Tip

如果合约已经编译，你应该删除`artifacts-zk`和`cache-zk`文件夹，否则除非你更改编译器版本，否则它不会重新编译。

:::

1. 创建 `contracts` 和 `deploy` 文件夹。 前者是我们将存储所有智能合约的 `*.sol` 文件的地方，后者是我们将放置与部署合约相关的所有脚本的地方。

2. 创建 `contracts/Greeter.sol` 合约并在其中粘贴以下代码：

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

5. 使用以下命令编译合约：

```
yarn hardhat compile
```

6. 在 `deploy/deploy.ts` 中创建以下部署脚本：

```typescript
import { Wallet, Provider, utils } from "zksync-web3";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

// An example of a deploy script that will deploy and call a simple contract.
export default async function (hre: HardhatRuntimeEnvironment) {
  console.log(`Running deploy script for the Greeter contract`);

  // Initialize the wallet.
  const provider = new Provider(hre.userConfig.zkSyncDeploy?.zkSyncNetwork);
  const wallet = new Wallet("<WALLET-PRIVATE-KEY>");

  // Create deployer object and load the artifact of the contract you want to deploy.
  const deployer = new Deployer(hre, wallet);
  const artifact = await deployer.loadArtifact("Greeter");

  // Estimate contract deployment fee
  const greeting = "Hi there!";
  const deploymentFee = await deployer.estimateDeployFee(artifact, [greeting]);

  // Deposit funds to L2
  const depositHandle = await deployer.zkWallet.deposit({
    to: deployer.zkWallet.address,
    token: utils.ETH_ADDRESS,
    amount: deploymentFee.mul(2),
  });
  // Wait until the deposit is processed on zkSync
  await depositHandle.wait();

  // Deploy this contract. The returned object will be of a `Contract` type, similarly to ones in `ethers`.
  // `greeting` is an argument for contract constructor.
  const parsedFee = ethers.utils.formatEther(deploymentFee.toString());
  console.log(`The deployment is estimated to cost ${parsedFee} ETH`);

  const greeterContract = await deployer.deploy(artifact, [greeting]);

  //obtain the Constructor Arguments
  console.log("constructor args:" + greeterContract.interface.encodeDeploy([greeting]));

  // Show the contract info.
  const contractAddress = greeterContract.address;
  console.log(`${artifact.contractName} was deployed to ${contractAddress}`);
}
```

7. 将 `WALLET-PRIVATE-KEY` 替换为您用于开发的前缀为 `0x` 私钥的以太坊钱包，并使用以下命令运行脚本以运行部署脚本：

```
yarn hardhat deploy-zksync
```

在 output 中，您将会看到部署智能合约的地址。

恭喜！ 您已将智能合约部署到 zkSync！ 现在您可以访问 [zkSync 区块浏览器](https://explorer.zksync.io/) 并搜索您的合约地址以确认它已成功部署。

[本指南](../../api/tools/block-explorer/contract-verification.md)解释了如何使用 zkSync 区块浏览器验证您的智能合约。

## 前端集成

### 设置项目

在本教程中，`Vue` 将用作选择的 Web 框架，但无论使用哪种框架，过程都将非常相似。 为了专注于使用 `zksync-web3` SDK 的细节，我们提供了一个模板来完成所有前端工作。 最后一步是与 zkSync 智能合约进行交互。

1. Clone 它:

```
git clone https://github.com/matter-labs/greeter-tutorial-starter
```

2. 启动项目：

```
cd greeter-tutorial-starter
yarn
yarn serve
```

默认情况下，页面应该在 `http://localhost:8080` 运行。 在浏览器中打开此 URL 以查看该页面。

### 连接到 Metamask 并将代币桥接到 zkSync

为了与基于 zkSync 构建的 dApp 进行交互，请将 Metamask 钱包连接到 zkSync alpha 测试网网络，并将一些资金桥接到 L2。

- 按照 [本指南](../fundamentals/testnet.md#connecting-metamask) 将 Metamask 连接到 zkSync。

- 使用我们的 [portal](https://portal.zksync.io) 将资金桥接到 zkSync。

### 项目构成

我们将把所有的代码写在`./src/App.vue`中。 几乎所有的前端代码都是开箱即用的，剩下的唯一任务就是填写 TODO-s 来与我们刚刚部署在 zkSync 上的合约进行交互：

```javascript
initializeProviderAndSigner() {
  // TODO: initialize provider and signer based on `window.ethereum`
},

async getGreeting() {
  // TODO: return the current greeting
  return "";
},

async getFee() {
  // TOOD: return formatted fee
  return "";
},

async getBalance() {
  // Return formatted balance
  return "";
},
async getOverrides() {
  if (this.selectedToken.l1Address != ETH_L1_ADDRESS) {
    // TODO: Return data for the paymaster
  }

  return {};
},
async changeGreeting() {
  this.txStatus = 1;
  try {
    // TODO: Submit the transaction
    this.txStatus = 2;
    // TODO: Wait for transaction compilation
    this.txStatus = 3;
    // Update greeting
    this.greeting = await this.getGreeting();
    this.retreivingFee = true;
    this.retreivingBalance = true;
    // Update balance and fee
    this.currentBalance = await this.getBalance();
    this.currentFee = await this.getFee();
  } catch (e) {
    alert(JSON.stringify(e));
  }
  this.txStatus = 0;
  this.retreivingFee = false;
  this.retreivingBalance = false;
},
```

在 `<script>` 标记的顶部，您可能会看到应填写已部署的 `Greeter` 合约的地址及其 ABI 路径的部分。 我们将在以下部分填写这些字段。

```javascript
// eslint-disable-next-line
const GREETER_CONTRACT_ADDRESS = ""; // TODO: insert the Greeter contract address here
// eslint-disable-next-line
const GREETER_CONTRACT_ABI = []; // TODO: insert the path to the Greeter contract ABI here
```

### 安装 `zksync-web3`

在 greeter-tutorial-starter 根文件夹中运行以下命令以安装 `zksync-web3` 和 `ethers`：

```
yarn add ethers zksync-web3
```

之后，在 `App.vue` 文件的 `script` 部分中导入这两个库（就在合约常量之前）。 它应该如下所示：

```javascript
import {} from "zksync-web3";
import {} from "ethers";

// eslint-disable-next-line
const GREETER_CONTRACT_ADDRESS = ""; // TODO: insert the Greeter contract address here
// eslint-disable-next-line
const GREETER_CONTRACT_ABI = []; // TODO: insert the path to the Greeter contract ABI here
```

### 获取 ABI 和合约地址

打开 `./src/App.vue` 并设置 `GREETER_CONTRACT_ADDRESS` 常量等于部署欢迎合约的地址。

为了与我们刚刚部署到 zkSync 的智能合约进行交互，我们还需要它的 ABI。 ABI 代表应用程序二进制接口，简而言之，它是一个文件，描述了与它交互的智能合约的所有可用名称和类型。

- 创建 `./src/abi.json` 文件。
- 您可以从 `./artifacts-zk/contracts/Greeter.sol/Greeter.json` 文件的上一节中的 hardhat 项目文件夹中获取合约的 ABI。 您应该复制 `abi` 数组并将其粘贴到上一步中创建的 `abi.json` 文件中。 该文件应大致如下所示：

```json
[
  {
    "inputs": [
      {
        "internalType": "string",
        "name": "_greeting",
        "type": "string"
      }
    ],
    "stateMutability": "nonpayable",
    "type": "constructor"
  },
  {
    "inputs": [],
    "name": "greet",
    "outputs": [
      {
        "internalType": "string",
        "name": "",
        "type": "string"
      }
    ],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [
      {
        "internalType": "string",
        "name": "_greeting",
        "type": "string"
      }
    ],
    "name": "setGreeting",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  }
]
```

将 `GREETER_CONTRACT_ABI` 设置为需要 ABI 文件。

```js
// eslint-disable-next-line
const GREETER_CONTRACT_ADDRESS = "0x...";
// eslint-disable-next-line
const GREETER_CONTRACT_ABI = require("./abi.json");
```

### 与供应商合作

1.进入`./src/App.vue`中的`initializeProviderAndSigner`方法。 该方法在与 Metamask 连接成功后调用。

在这种方法中，我们应该：

- 初始化 `Web3Provider` 和 `Signer` 然后与 zkSync 交互。

- 初始化 `Contract` 对象然后与我们刚刚部署的 `Greeter` 合约进行交互。
2. 导入必要的依赖：

```javascript
import { Contract, Web3Provider, Provider } from "zksync-web3";
```

3. 像这样初始化提供者、签名者和合约实例：

```javascript
initializeProviderAndSigner() {
    this.provider = new Provider('https://zksync2-testnet.zksync.dev');
    // Note that we still need to get the Metamask signer
    this.signer = (new Web3Provider(window.ethereum)).getSigner();
    this.contract = new Contract(
        GREETER_CONTRACT_ADDRESS,
        GREETER_CONTRACT_ABI,
        this.signer
    );
},
```

### 检索问候语

1. 填写从智能合约中检索问候语的方法：

```javascript
async getGreeting() {
    // Smart contract calls work the same way as in `ethers`
    return await this.contract.greet();
}
```

完整的方法现在看起来如下：

```javascript
initializeProviderAndSigner() {
    this.provider = new Provider('https://zksync2-testnet.zksync.dev');
    // Note that we still need to get the Metamask signer
    this.signer = (new Web3Provider(window.ethereum)).getSigner();
    this.contract = new Contract(
        GREETER_CONTRACT_ADDRESS,
        GREETER_CONTRACT_ABI,
        this.signer
    );
},
async getGreeting() {
    return await this.contract.greet();
},
```

连接 Metamask 钱包后，您应该会看到以下页面：

![img](../../assets/images/start-1.png)

现在可以选择所选择的支付费用的代币。 但是，尚未更新*yet*余额。

### 检索代币余额和交易费用

检索用户余额的最简单方法是使用 `Signer.getBalance` 方法。

1.添加必要的依赖：

```javascript
// `ethers` is only used in this tutorial for its utility functions
import { ethers } from "ethers";
```

2. 实现方法：

```javascript
async getBalance() {
    // Getting the balance for the signer in the selected token
    const balanceInUnits = await this.signer.getBalance(this.selectedToken.l2Address);
    // To display the number of tokens in the human-readable format, we need to format them,
    // e.g. if balanceInUnits returns 500000000000000000 wei of ETH, we want to display 0.5 ETH the user
    return ethers.utils.formatUnits(balanceInUnits, this.selectedToken.decimals);
},
```

3. 估算费用：

```javascript
async getFee() {
    // Getting the amount of gas (ergs) needed for one transaction
    const feeInGas = await this.contract.estimateGas.setGreeting(this.newGreeting);
    // Getting the gas price per one erg. For now, it is the same for all tokens.
    const gasPriceInUnits = await this.provider.getGasPrice();

    // To display the number of tokens in the human-readable format, we need to format them,
    // e.g. if feeInGas*gasPriceInUnits returns 500000000000000000 wei of ETH, we want to display 0.5 ETH the user
    return ethers.utils.formatUnits(feeInGas.mul(gasPriceInUnits), this.selectedToken.decimals);
},
```

::: tip 以 ERC20 支付费用

zkSync v2 本身不支持以 ERC20 代币支付费用，但账户抽象功能改进了这一点。 我们将在下面向您展示如何实现 testnet paymaster，但是，在主网上工作时，您应该提供 paymaster 服务 [yourself](../tutorials/custom-paymaster-tutorial.md) 或使用第 3 方 paymaster 。

:::

打开页面并选择支付费用的代币时，将显示交易的余额和预期费用。

`Refresh` 按钮用于重新计算费用，因为费用可能取决于我们要存储为问候语的消息的长度。

也可以单击“更改问候语”按钮，但由于尚未调用合约，因此不会更改任何内容。

![img](../../assets/images/start-2.png)

### 更新问候语

1. 与智能合约交互的方式与在 `ethers` 中的交互方式完全相同，但是，如果您想使用 zkSync 特定的功能，您可能需要在覆盖中提供一些额外的参数：

```javascript
// The example of paying fees using a paymaster will be shown in the
// section below.
const txHandle = await this.contract.setGreeting(this.newGreeting, await this.getOverrides());
```

2. 等待交易被提交：

```javascript
await txHandle.wait();
```

完整的方法如下：

```javascript
async changeGreeting() {
    this.txStatus = 1;
    try {
        const txHandle = await this.contract.setGreeting(this.newGreeting, await this.getOverrides());

        this.txStatus = 2;

        // Wait until the transaction is committed
        await txHandle.wait();
        this.txStatus = 3;

        // Update greeting
        this.greeting = await this.getGreeting();

        this.retreivingFee = true;
        this.retreivingBalance = true;
        // Update balance and fee
        this.currentBalance = await this.getBalance();
        this.currentFee = await this.getFee();
    } catch (e) {
        alert(JSON.stringify(e));
    }

    this.txStatus = 0;
    this.retreivingFee = false;
    this.retreivingBalance = false;
},
```

您现在拥有一个功能齐全的 Greeter-dApp！ 但是，它没有利用任何 zkSync 特定的功能。

### 使用 testnet paymaster 支付费用

尽管 ETH 是您可以用来支付费用的唯一代币，但账户抽象功能允许您集成 [paymasters](./aa.md#paymasters)，它可以完全为您支付费用或即时交换您的代币。 在本教程中，我们将使用所有 zkSync 测试网上提供的 [testnet paymaster](./aa.md#testnet-paymaster)。 它允许用户以 1:1 的 ETH 汇率以 ERC20 代币支付费用，即 1 单位代币兑换 1 wei ETH。

::: tip 主网集成

Testnet paymaster 纯粹用于演示该功能，不会在主网上使用。 在主网上集成您的协议时，您应该遵循您将使用的 paymaster 文档。

:::

当用户决定使用 ETH 支付时，`getOverrides` method 返回对象为 `null`，但当用户选择ERC20选项时，它会返回 Paymaster 合约地址以及所需的全部信息。具体方法如下:

1. 通过 zkSync Provider 中获取测试网 Paymaster 的合约地址：

```javascript
async getOverrides() {
  if (this.selectedToken.l1Address != ETH_L1_ADDRESS) {
    const testnetPaymaster = await this.provider.getTestnetPaymasterAddress();

    // ..
  }

  return {};
}
```

请注意，建议每次在进行任何交互之前检索测试网 Paymaster 的地址，因为它可能会发生变化。

2. 将 `utils` 添加到来自 `zksync-web3` SDK 的导入中：

```javascript
import { Contract, Web3Provider, Provider, utils } from "zksync-web3";
```

2. 由于 testnet paymaster 以 1:1 的比率将任何 ERC20 代币兑换成 ETH，因此金额与 ETH 金额相同：

```javascript
async getOverrides() {
  if (this.selectedToken.l1Address != ETH_L1_ADDRESS) {
    const testnetPaymaster = await this.provider.getTestnetPaymasterAddress();

    const gasPrice = await this.provider.getGasPrice();
    const gasLimit = await this.contract.estimateGas.setGreeting(this.newGreeting);
    const fee = gasPrice.mul(gasLimit);

    // ..
  }

  return {};
}
```

3. 现在，剩下的就是按照 [协议要求](./aa.md#testnet-paymaster) 对 paymasterInput 进行编码并返回所需的覆盖：

```javascript
async getOverrides() {
  if (this.selectedToken.l1Address != ETH_L1_ADDRESS) {
    const testnetPaymaster = await this.provider.getTestnetPaymasterAddress();

    const gasPrice = await this.provider.getGasPrice();
    const gasLimit = await this.contract.estimateGas.setGreeting(this.newGreeting);
    const fee = gasPrice.mul(gasLimit);

    const paymasterParams = utils.getPaymasterParams(testnetPaymaster, {
        type: 'ApprovalBased',
        token: this.selectedToken.l2Address,
        minimalAllowance: fee,
        // empty bytes as testnet paymaster does not use innerInput
        innerInput: new Uint8Array()
    });

    return {
        maxFeePerGas: gasPrice,
        maxPriorityFeePerGas: ethers.BigNumber.from(0),
        gasLimit,
        customData: {
            ergsPerPubdata: utils.DEFAULT_ERGS_PER_PUBDATA_LIMIT,
            paymasterParams
        }
    };
  }

  return {};
}
```

4. 要使用 ERC20 代币列表，请更改以下行：

```javascript
const allowedTokens = require("./eth.json");
```

到以下：

```javascript
const allowedTokens = require("./erc20.json");
```

### 完整的应用程序

现在您应该能够更新问候消息了。

1. 在输入框中输入新的问候语，点击  `Change greeting` 按钮：

![img](../../assets/images/start-3.png)

1. 由于提供了`paymasterParams`，交易将是`EIP712`（[更多关于EIP712的信息](https://eips.ethereum.org/EIPS/eip-712)）：

![img](../../assets/images/start-4.png)

1. 点击“签名”。
   
   交易处理完毕后，页面会更新余额并可以查看新的问候语：

![img](../../assets/images/start-5.png)

### 学到更多

- 要了解有关 `zksync-web3` SDK 的更多信息，请查看 [文档](../../api/js)。
- 要了解有关 zkSync 安全帽插件的更多信息，请查看 [文档](../../api/hardhat)。
