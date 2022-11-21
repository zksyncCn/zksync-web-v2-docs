# 快速入门

在本快速入门指南中，您将学习如何将智能合约部署到 zkSync，并使用 zkSync 开发工具构建一个 dApp 与之互动。

以下是我们需要构建的:

- 一个部署在 zkSync 上存储问候语信息(greeting message)的智能合约。
- 一个用于获取问候信息的 dApp。
- 这个问候语信息可以被用户修改。
- 默认情况下，用户需要支付交易费用来改变以太坊的问候语信息。不过，我们也将解释如何[使用测试网 paymaster 合约](#paying-fees-using-testnet-paymaster)(注：Paymaster 即付款人，交易所需 Gas 的实际支付者)，让用户使用 ERC20 代币支付费用。
- 
::: 小提示

测试网 Paymaster 合约仅用于测试。如果您想在主网上构建一个项目，您应该阅读关于 [Paymasters 的文档](./aa.md#paymasters)。

:::

## 前提条件

- `yarn` package manager(应用程序安装包管理器)：[安装指南](https://yarnpkg.com/getting-started/install)(`npm` 示例将很快被添加)。
- 一个在 L1 上有足够 Göerli `ETH` 的钱包，用于支付 zkSync 的桥接资金以及部署智能合约。如果您想执行测试网 paymaster 合约，则需要 zkSync 上的 ERC20 测试代币。我们建议使用 [zkSync 门户网站的水龙头](https://portal.zksync.io/faucet)功能来获取测试代币。

## 初始化项目并部署智能合约

1. 初始化项目并安装依赖项。在终端上运行以下命令:

```
mkdir greeter-example
cd greeter-example
yarn init -y
yarn add -D typescript ts-node ethers zksync-web3 hardhat @matterlabs/hardhat-zksync-solc @matterlabs/hardhat-zksync-deploy
```

请注意：Typescript 语言是 zkSync 插件所必需的。

2. 创建 hardhat.config.ts 文件并在其中粘贴以下代码：

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

::: 警告

如果合约已经被编译过，您需要删除 `artifacts-zk` 与 `cache-zk` 文件夹，否则，它将不会被重新编译，除非您改变编译器版本。

:::

1. 创建 `contracts` 和 `deploy` 文件夹。前者是我们将存储所有智能合约的 `*.sol` 文件的地方，后者是我们将放置所有与部署智能合约有关的脚本的地方。

2. 创建 `contracts/Greeter.sol` 合约，并在其中粘贴以下代码：

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

5. 使用以下命令编译智能合约：

```
yarn hardhat compile
```

6. 在 `deploy/deploy.ts` 中创建以下脚本：

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

7. 将 `WALLET-PRIVATE-KEY` 替换为您用于开发的以太坊钱包的 `0x` 前缀的私钥，并使用以下命令运行脚本：

```
yarn hardhat deploy-zksync
```

在 output 中，您将会看到部署智能合约的地址。

恭喜！您已经在 zkSync 上成功部署了智能合约。现在您可以访问[zkSync 区块浏览器](https://explorer.zksync.io/)并搜索您的智能合约地址以确认它被成功部署。

[本指南](../../api/tools/block-explorer/contract-verification.md)解释了如何使用 zkSync 区块浏览器验证您的智能合约。

## 前端集成指南

### 创建项目

在本教程中，`Vue` 将被用作首选的Web框架，但无论使用哪种框架，其过程都很相似。为了将注意力集中在 `zksync-web3` SDK 的使用上，我们提供了一个已完成所有前端工作的模板。最后一步则是与 zkSync 智能合约进行交互。

1. 复制此代码：

```
git clone https://github.com/matter-labs/greeter-tutorial-starter
```

2. 启动项目:

```
cd greeter-tutorial-starter
yarn
yarn serve
```

默认情况下，该页面应该在 `http://localhost:8080` 上运行。在浏览器中打开这个 URL 就可以看到该页面。

### 连接 Metamask 并将代币桥接到 zkSync

为了与构建在 zkSync 上的 dApps 互动，请将 Metamask 钱包连接到 zkSync alpha testnet 网络，并将一些资金桥接到 L2。

- 按照[本指南](../fundamentals/testnet.md#connecting-metamask)将 Metamask 连接到 zkSync。

- 使用 [zkSync 门户](https://portal.zksync.io)将代币桥接到 zkSync。

### 项目架构

我们将在`./src/App.vue` 中编写所有代码。几乎所有的前端代码都可以直接使用，唯一剩下的任务就是填写 TODO-s 来与我们刚刚部署在 zkSync 上的合约进行交互：

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

在 `<script>` 标签的顶部，您可以看到应该填写部署的 `Greeter` 的智能合约地址和其 ABI 的路径的部分。我们将在下面的章节中填写这些字段：

```javascript
// eslint-disable-next-line
const GREETER_CONTRACT_ADDRESS = ""; // TODO: insert the Greeter contract address here
// eslint-disable-next-line
const GREETER_CONTRACT_ABI = []; // TODO: insert the path to the Greeter contract ABI here
```

### 安装 `zksync-web3`

在 greeter-tutorial-starter 根目录中运行以下命令以安装 `zksync-web3` 和 `ethers`：

```
yarn add ethers zksync-web3
```

之后，在 `App.vue` 文件的 `script` 部分导入这两个库（就在合约常量之前）。它应该是这样的：

```javascript
import {} from "zksync-web3";
import {} from "ethers";

// eslint-disable-next-line
const GREETER_CONTRACT_ADDRESS = ""; // TODO: insert the Greeter contract address here
// eslint-disable-next-line
const GREETER_CONTRACT_ABI = []; // TODO: insert the path to the Greeter contract ABI here
```

### 获取 ABI 及合约地址

打开 `./src/App.vue` 并设置 `GREETER_CONTRACT_ADDRESS` 常量等于部署问候合约(greeter contract)的地址。

要与我们刚刚部署到 zkSync 的智能合约进行交互，我们还需要它的 ABI。ABI 代表应用程序二进制接口，简而言之，它是一个文件，描述了与之交互的智能合约方法的所有可用名称和类型。

- 创建 `./src/abi.json` 文件。
- 您可以在 hardhat 项目文件夹中的 `./artifacts-zk/contracts/Greeter.sol/Greeter.json` 文件里获取合约的 ABI。您应该复制 `abi` 数组并将其粘贴到在上一步中创建的 `abi.json` 文件中。该文件应该如下所示:

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

设置 `GREETER_CONTRACT_ABI` 以引用 ABI 文件。

```js
// eslint-disable-next-line
const GREETER_CONTRACT_ADDRESS = "0x...";
// eslint-disable-next-line
const GREETER_CONTRACT_ABI = require("./abi.json");
```

### Provider 的使用

1. 转到 `./src/App.vue` 中的 `initializeProviderAndSigner` method 模块。这个 method 模块会在与 Metamask 的连接后被调用。

在这个 method 模块中，我们需要：

- 初始化 `Web3Provider` 和 `Signer` 来与 zkSync 互动。
- 初始化 `Contract` 对象来与我们刚刚部署的 `Greeter` 合约进行交互。

2. 导入必要的依赖项：

```javascript
import { Contract, Web3Provider, Provider } from "zksync-web3";
```

3. 初始化 Provider、Signer、合约实例，方法如下:

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

### 获取问候信息

1. 写入从智能合约中获取问候信息的方法：

```javascript
async getGreeting() {
    // Smart contract calls work the same way as in `ethers`
    return await this.contract.greet();
}
```

完整的 method 模块现在看起来是这样的:

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

现在可以选择用于支付费用的代币，但是余额还没有被更新。


### 获取代币余额和交易费用

获取用户账户余额的最简单方法是使用 `Signer.getBalance` method 模块。

1. 添加必要的依赖项：

```javascript
// `ethers` is only used in this tutorial for its utility functions
import { ethers } from "ethers";
```

2. Method 模块的自我调用：

```javascript
async getBalance() {
    // Getting the balance for the signer in the selected token
    const balanceInUnits = await this.signer.getBalance(this.selectedToken.l2Address);
    // To display the number of tokens in the human-readable format, we need to format them,
    // e.g. if balanceInUnits returns 500000000000000000 wei of ETH, we want to display 0.5 ETH the user
    return ethers.utils.formatUnits(balanceInUnits, this.selectedToken.decimals);
},
```

3. 费用估算：

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

::: 使用 ERC20 代币支付费用

zkSync v2 本身不支持使用 ERC20 代币支付费用，但账户抽象功能为其提供了便利。我们将在下面向您展示如何实现测试网的 Paymaster 模块。但是，当需要在主网上工作时，您应该[自己提供 Paymaster 合约](../tutorials/custom-paymaster-tutorial.md) 或使用第三方 Paymaster 合约。

:::

当打开页面并选择支付费用的代币时，将可以看到交易的余额和预期费用。

应使用 `Refresh` 按钮来重新计算费用，因为费用可能取决于我们想存储为问候语信息的长度。

也可以点击 `Change greeting` 按钮，但因为合约还没有被调用，所以不会有任何更改。

![img](../../assets/images/start-2.png)

### 更新问候语信息

1. 与智能合约交互的方式与在`以太坊`中完全是相同的，但是，如果您想使用 zkSync 的某些特定功能，您可能需要在 overrides 变量中添加一些额外的参数：

```javascript
// The example of paying fees using a paymaster will be shown in the
// section below.
const txHandle = await this.contract.setGreeting(this.newGreeting, await this.getOverrides());
```

2. 等待交易被提交：

```javascript
await txHandle.wait();
```

完整的 method 模块看起来应该是这样的：

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

你现在有了一个功能齐全的 Greeter-dApp! 然而，它并没有利用任何 zkSync 的特定功能。

### 使用 Testnet paymaster 支付费用

尽管以太币是您可以用来支付费用的唯一代币，但账户抽象功能允许您集成 [Paymasters](./aa.md#paymasters)，它可以完全为您支付费用或即时交换为您的代币。在本教程中，我们将使用在所有 zkSync 测试网上都提供的 [测试网 paymaster 合约](./aa.md#testnet-paymaster)。它允许用户使用 ERC20 代币支付费用，ETH 的兑换率为 1:1。

::: 主网集成小提示

测试网的 paymaster 合约纯粹是为了演示该功能，并不会在主网上提供。在主网上集成您的协议时，您应该遵循您将使用的 paymaster 合约文档。

:::

当用户决定使用以太币支付时，`getOverrides` method 返回对象为“空”，但当用户选择ERC20选项时，它会返回 paymaster 合约地址以及所需的全部信息。具体方法如下:

1. 通过 zkSync provider 中获取测试网 paymaster 的合约地址：

```javascript
async getOverrides() {
  if (this.selectedToken.l1Address != ETH_L1_ADDRESS) {
    const testnetPaymaster = await this.provider.getTestnetPaymasterAddress();

    // ..
  }

  return {};
}
```

注意，建议每次在任何交互之前都获取一次测试网 paymaster 的合约地址，因为它可能会发生变化。

2. 在从 `zksync-web3` SDK 导入的文件中加入 `utils`：

```javascript
import { Contract, Web3Provider, Provider, utils } from "zksync-web3";
```

2. 我们需要计算处理交易需要多少代币。由于 testnet paymaster 合约以 1:1 的比例将任何 ERC20 代币兑换成 ETH，因此数量与 ETH 数量相同：

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

3. 现在，剩下的就是按照[协议要求](./aa.md#testnet-paymaster)对 paymasterInput 进行编译，并返回所需的 overrides:

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

4. 要使用 ERC20 代币列表，请将下面这行代码:

```javascript
const allowedTokens = require("./eth.json");
```

改为:

```javascript
const allowedTokens = require("./erc20.json");
```

### 完整的应用程序

现在您应该能够更新问候语信息了。

1. 在输入框中输入新的问候语信息，然后点击 `Change greeting` 按钮：

![img](../../assets/images/start-3.png)

1. 由于提供了 `paymasterParams`，交易将符合 `EIP712` 标准。（[更多关于 EIP712 的信息](https://eips.ethereum.org/EIPS/eip-712)）：

![img](../../assets/images/start-4.png)

1. 点击“Sign”：

交易处理完成后，页面余额更新，即可以查看新的问候语信息：

![img](../../assets/images/start-5.png)

### 了解更多

- 要了解更多关于 `zksync-web3`SDK 的信息，请查看其[文档](.../.../api/js)。
- 要了解有关 zkSync hardhat 插件的更多信息，请查看其[文档](../../api/hardhat)。
