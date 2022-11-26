# 建立自定义 paymaster

让我们看看如何使用 paymaster 功能来构建一个自定义的 paymaster，这允许用户使用我们的代币支付费用。 在本教程中，我们将：

- 创建一个 paymaster，假设一个 ERC20 代币的单位足以支付任何交易费用。
- 创建 ERC20 代币合约并将一些代币发送到一个全新的钱包。
- 最后，我们将通过 paymaster 从新创建的钱包中发送 `Mint`交易。 即使交易通常需要一些 ETH 来支付 gas 费，我们的 paymaster 也会执行交易以换取 1 个 ERC20 代币。

## 先决条件

为了更好地理解这部分内容，我们建议您在深入学习本教程之前先阅读 [账户抽象设计](../developer-guides/aa.md) 。

假设您已经熟悉在 zkSync 上部署智能合约。 如果没有，请阅读[快速入门](../developer-guides/hello-world.md) 部分。 以及系统合约的 [简介](../developer-guides/contracts/system-contracts.md)。

## 安装依赖项

我们将使用 zkSync hardhat 插件来开发这个合约。 首先，我们应该为其安装所有依赖项：

```
mkdir custom-paymaster-tutorial
cd custom-paymaster-tutorial
yarn init -y
yarn add -D typescript ts-node ethers zksync-web3 hardhat @matterlabs/hardhat-zksync-solc @matterlabs/hardhat-zksync-deploy
```

由于我们正在使用 zkSync 合约，我们还需要安装包含合约及其对等依赖项的包：

```
yarn add @matterlabs/zksync-contracts @openzeppelin/contracts @openzeppelin/contracts-upgradeable
```

然后创建 `hardhat.config.ts` 配置文件、`contracts` 和 `deploy` 文件夹，就像在 [快速入门教程](../developer-guides/hello-world.md) 中一样。

## 构建

我们的协议将是一个虚拟协议，允许任何人交换某个 ERC20 代币以换取支付交易费用。

paymaster 的构建如下：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { IPaymaster, ExecutionResult } from '@matterlabs/zksync-contracts/l2/system-contracts/interfaces/IPaymaster.sol';
import { IPaymasterFlow } from '@matterlabs/zksync-contracts/l2/system-contracts/interfaces/IPaymasterFlow.sol';
import { TransactionHelper, Transaction } from '@matterlabs/zksync-contracts/l2/system-contracts/TransactionHelper.sol';

import '@matterlabs/zksync-contracts/l2/system-contracts/Constants.sol';

import "@openzeppelin/contracts/interfaces/IERC20.sol";

contract MyPaymaster is IPaymaster {
    uint256 constant PRICE_FOR_PAYING_FEES = 1;

    address public allowedToken;

    modifier onlyBootloader() {
        require(msg.sender == BOOTLOADER_FORMAL_ADDRESS, "Only bootloader can call this method");
        // Continure execution if called from the bootloader.
        _;
    }

    constructor(address _erc20) {
        allowedToken = _erc20;
    }

    function validateAndPayForPaymasterTransaction(bytes32, bytes32, Transaction calldata _transaction) external payable override onlyBootloader returns (bytes memory context) {
        // Transaction validation logic goes here
    }

    function postOp(
        bytes calldata _context,
        Transaction calldata _transaction,
        bytes32 _txHash,
        bytes32 _suggestedSignedHash,
        ExecutionResult _txResult,
        uint256 _maxRefundedErgs
    ) external payable onlyBootloader {
        // This contract does not support any refunding logic
    }

    receive() external payable {}
}
```

注意，只有 [bootloader](../developer-guides/contracts/system-contracts.md#bootloader) 应该被允许调用 `validateAndPayForPaymasterTransaction`/`postOp` 。 这就是为它们使用 `onlyBootloader` 修饰符的原因。

### 解析付款人输入

在本教程中，我们希望向用户收取一个单位的 `allowedToken`，用来换取通过合约支付的费用。

paymaster 应该接收的输入被编码在 `paymasterInput` 中。 如 [在 paymaster 文档中](../developer-guides/aa.md#built-in-paymaster-flows) 所述，有一些标准化的方法可以对用户与 paymasterInput 的交互进行编码。 为了向用户收费，我们将要求为 paymaster合同提供足够的补贴。 这就是 `approvalBased` 流程可以帮助我们解决的问题。

首先，我们需要检查 `paymasterInput` 是否被编码为 `approvalBased` 的流程：

```solidity
require(_transaction.paymasterInput.length >= 4, "The standard paymaster input must be at least 4 bytes long");

bytes4 paymasterInputSelector = bytes4(_transaction.paymasterInput[0:4]);
if (paymasterInputSelector == IPaymasterFlow.approvalBased.selector) {
    (address token, uint256 minAllowance, bytes memory data) = abi.decode(_transaction.paymasterInput[4:], (address, uint256, bytes));

    require(token == allowedToken, "Invalid token");
    require(minAllowance >= 1, "Min allowance too low");

    //
    // ...
    //
} else {
    revert("Unsupported paymaster flow");
}
```

然后，我们需要检查用户是否确实提供了足够的补贴：

```solidity
address userAddress = address(uint160(_transaction.from));
address thisAddress = address(this);

uint256 providedAllowance = IERC20(token).allowance(userAddress, thisAddress);
require(providedAllowance >= PRICE_FOR_PAYING_FEES, "The user did not provide enough allowance");
```

然后，我们将资金转移给用户，以换取 1 个unit 令牌。

```solidity
// Note, that while the minimal amount of ETH needed is tx.ergsPrice * tx.ergsLimit,
// neither paymaster nor account are allowed to access this context variable.
uint256 requiredETH = _transaction.ergsLimit * _transaction.maxFeePerErg;

// Pulling all the tokens from the user
IERC20(token).transferFrom(userAddress, thisAddress, 1);
// The bootloader never returns any data, so it can safely be ignored here.
(bool success, ) = payable(BOOTLOADER_FORMAL_ADDRESS).call{value: requiredETH}("");
require(success, "Failed to transfer funds to the bootloader");
```

::: 提示   您应该首先验证是否完成所有要求

如果paymaster 节流的 [rules](../developer-guides/aa.md#paymaster-validation-rules) 第一个存储读取的值与执行时的值不同，paymaster 将不会被节流 API是属于用户的存储槽。

这就是为什么在执行任何逻辑之前验证用户是否为交易提供了所有允许的先决条件很重要。 这就是我们 _first_ 检查用户是否提供了足够的补贴的原因，然后我们才做 `transferFrom`.

:::

### Full code of the paymaster

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { IPaymaster, ExecutionResult } from '@matterlabs/zksync-contracts/l2/system-contracts/interfaces/IPaymaster.sol';
import { IPaymasterFlow } from '@matterlabs/zksync-contracts/l2/system-contracts/interfaces/IPaymasterFlow.sol';
import { TransactionHelper, Transaction } from '@matterlabs/zksync-contracts/l2/system-contracts/TransactionHelper.sol';

import '@matterlabs/zksync-contracts/l2/system-contracts/Constants.sol';

import "@openzeppelin/contracts/interfaces/IERC20.sol";

contract MyPaymaster is IPaymaster {
    uint256 constant PRICE_FOR_PAYING_FEES = 1;

    address public allowedToken;

    modifier onlyBootloader() {
        require(msg.sender == BOOTLOADER_FORMAL_ADDRESS, "Only bootloader can call this method");
        // Continure execution if called from the bootloader.
        _;
    }

    constructor(address _erc20) {
        allowedToken = _erc20;
    }

    function validateAndPayForPaymasterTransaction(bytes32 _txHash, bytes32 _suggestedSignedHash, Transaction calldata _transaction) external payable override onlyBootloader returns (bytes memory context) {
        require(_transaction.paymasterInput.length >= 4, "The standard paymaster input must be at least 4 bytes long");

        bytes4 paymasterInputSelector = bytes4(_transaction.paymasterInput[0:4]);
        if (paymasterInputSelector == IPaymasterFlow.approvalBased.selector) {
            (address token, uint256 minAllowance, bytes memory data) = abi.decode(_transaction.paymasterInput[4:], (address, uint256, bytes));

            require(token == allowedToken, "Invalid token");
            require(minAllowance >= 1, "Min allowance too low");

            address userAddress = address(uint160(_transaction.from));
            address thisAddress = address(this);

            uint256 providedAllowance = IERC20(token).allowance(userAddress, thisAddress);
            require(providedAllowance >= PRICE_FOR_PAYING_FEES, "The user did not provide enough allowance");

            // Note, that while the minimal amount of ETH needed is tx.ergsPrice * tx.ergsLimit,
            // neither paymaster nor account are allowed to access this context variable.
            uint256 requiredETH = _transaction.ergsLimit * _transaction.maxFeePerErg;

            // Pulling all the tokens from the user
            IERC20(token).transferFrom(userAddress, thisAddress, 1);
            // The bootloader never returns any data, so it can safely be ignored here.
            (bool success, ) = payable(BOOTLOADER_FORMAL_ADDRESS).call{value: requiredETH}("");
            require(success, "Failed to transfer funds to the bootloader");
        } else {
            revert("Unsupported paymaster flow");
        }
    }

    function postOp(
        bytes calldata _context,
        Transaction calldata _transaction,
        bytes32 _txHash,
        bytes32 _suggestedSignedHash,
        ExecutionResult _txResult,
        uint256 _maxRefundedErgs
    ) external payable onlyBootloader {
        // This contract does not support any refunding logic
    }

    receive() external payable {}
}
```

## 部署 ERC20 合约

为了测试我们的paymaster，我们需要一个 ERC20 代币。 为了简单表述，我们将使用稍微修改过的 OpenZeppelin 实现：

创建 `MyERC20.sol` 文件并将以下代码放入其中：

```solidity
// SPDX-License-Identifier: UNLICENSED

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyERC20 is ERC20 {
    uint8 private _decimals;

    constructor(
        string memory name_,
        string memory symbol_,
        uint8 decimals_
    ) ERC20(name_, symbol_) {
        _decimals = decimals_;
    }

    function mint(address _to, uint256 _amount) public returns (bool) {
        _mint(_to, _amount);
        return true;
    }

    function decimals() public view override returns (uint8) {
        return _decimals;
    }
}
```

## 部署 paymaster

要部署 ERC20 代币和paymaster，我们需要创建一个部署脚本。 创建 `deploy` 文件夹并在其中创建一个文件：`deploy-paymaster.ts`。 将以下部署脚本放在那里：

```ts
import { utils, Wallet } from "zksync-web3";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

export default async function (hre: HardhatRuntimeEnvironment) {
  // The wallet that will deploy the token and the paymaster
  // It is assumed that this wallet already has sufficient funds on zkSync
  // ⚠️ Never commit private keys to file tracking history, or your account could be compromised.
  const wallet = new Wallet("<PRIVATE-KEY>");
  // The wallet that will receive ERC20 tokens
  const emptyWallet = Wallet.createRandom();
  console.log(`Empty wallet's address: ${emptyWallet.address}`);
  console.log(`Empty wallet's private key: ${emptyWallet.privateKey}`);

  const deployer = new Deployer(hre, wallet);

  // Deploying the ERC20 token
  const erc20Artifact = await deployer.loadArtifact("MyERC20");
  const erc20 = await deployer.deploy(erc20Artifact, ["MyToken", "MyToken", 18]);
  console.log(`ERC20 address: ${erc20.address}`);

  // Deploying the paymaster
  const paymasterArtifact = await deployer.loadArtifact("MyPaymaster");
  const paymaster = await deployer.deploy(paymasterArtifact, [erc20.address]);
  console.log(`Paymaster address: ${paymaster.address}`);

  // Supplying paymaster with ETH
  await (
    await deployer.zkWallet.sendTransaction({
      to: paymaster.address,
      value: ethers.utils.parseEther("0.03"),
    })
  ).wait();

  // Supplying the ERC20 tokens to the empty wallet:
  await // We will give the empty wallet 3 units of the token:
  (await erc20.mint(emptyWallet.address, 3)).wait();

  console.log("Minted 3 tokens for the empty wallet");

  console.log(`Done!`);
}
```

除了部署 paymaster 之外，它还创建了一个空钱包并向其提供了一些 `MyERC20` 代币，以便它可以使用 paymaster。

要部署 ERC20 代币和付款人，您应该编译合约并运行脚本：

```
yarn hardhat compile
yarn hardhat deploy-zksync --script deploy-paymaster.ts
```

输出应该大致如下：

```
Empty wallet's address: 0xAd155D3069BB3c587E995916B320444056d8191F
Empty wallet's private key: 0x236d735297617cc68f4ec8ceb40b351ca5be9fc585d446fa95dff02354ac04fb
ERC20 address: 0x65C899B5fb8Eb9ae4da51D67E1fc417c7CB7e964
Paymaster address: 0x0a67078A35745947A37A552174aFe724D8180c25
Minted 3 tokens for the empty wallet
Done!
```

请注意，每次运行的地址和私钥都会不同。

## 使用 paymaster

在 `deploy` 文件夹中创建 `use-paymaster.ts` 脚本。 您可以在下面的代码片段中看到与paymaster 交互的示例：

```ts
import { Provider, utils, Wallet } from "zksync-web3";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";

// Put the address of the deployed paymaster here
const PAYMASTER_ADDRESS = "<PAYMASTER_ADDRESS>";

// Put the address of the ERC20 token here:
const TOKEN_ADDRESS = "<TOKEN_ADDRESS>";

// Wallet private key
// ⚠️ Never commit private keys to file tracking history, or your account could be compromised.
const EMPTY_WALLET_PRIVATE_KEY = "<EMPTY_WALLET_PRIVATE_KEY>";

function getToken(hre: HardhatRuntimeEnvironment, wallet: Wallet) {
  const artifact = hre.artifacts.readArtifactSync("MyERC20");
  return new ethers.Contract(TOKEN_ADDRESS, artifact.abi, wallet);
}

export default async function (hre: HardhatRuntimeEnvironment) {
  const provider = new Provider(hre.config.zkSyncDeploy.zkSyncNetwork);
  const emptyWallet = new Wallet(EMPTY_WALLET_PRIVATE_KEY, provider);

  // Obviously this step is not required, but it is here purely to demonstrate that indeed the wallet has no ether.
  const ethBalance = await emptyWallet.getBalance();
  if (!ethBalance.eq(0)) {
    throw new Error("The wallet is not empty");
  }

  console.log(`Balance of the user before mint: ${await emptyWallet.getBalance(TOKEN_ADDRESS)}`);

  const erc20 = getToken(hre, emptyWallet);

  const gasPrice = await provider.getGasPrice();

  // Estimate gas fee for mint transaction
  const gasLimit = await erc20.estimateGas.mint(emptyWallet.address, 100, {
    customData: {
      ergsPerPubdata: utils.DEFAULT_ERGS_PER_PUBDATA_LIMIT,
      paymasterParams: {
        paymaster: PAYMASTER_ADDRESS,
        // empty input as our paymaster doesn't require additional data
        paymasterInput: "0x",
      },
    },
  });

  const fee = gasPrice.mul(gasLimit.toString());

  // Encoding the "ApprovalBased" paymaster flow's input
  const paymasterParams = utils.getPaymasterParams(PAYMASTER_ADDRESS, {
    type: "ApprovalBased",
    token: TOKEN_ADDRESS,
    // set minimalAllowance as we defined in the paymaster contract
    minimalAllowance: ethers.BigNumber.from(1),
    // empty bytes as testnet paymaster does not use innerInput
    innerInput: new Uint8Array(),
  });

  await (
    await erc20.mint(emptyWallet.address, 100, {
      // Provide gas params manually
      maxFeePerGas: gasPrice,
      maxPriorityFeePerGas: gasPrice,
      gasLimit,

      // paymaster info
      customData: {
        paymasterParams,
        ergsPerPubdata: utils.DEFAULT_ERGS_PER_PUBDATA_LIMIT,
      },
    })
  ).wait();

  console.log(`Balance of the user after mint: ${await emptyWallet.getBalance(TOKEN_ADDRESS)}`);
}
```

使用上一步中提供的输出填写参数 `PAYMASTER_ADDRESS`、`TOKEN_ADDRESS` 和 `EMPTY_WALLET_PRIVATE_KEY` 后，使用以下命令运行此脚本：

```
yarn hardhat deploy-zksync --script use-paymaster.ts
```

输出应该大致如下：

```
Balance of the user before mint: 3
Balance of the user after mint: 102
```

运行部署脚本后，钱包有 3 个代币，在将交易发送给`mint`及另外 100 个代币后，余额为 102，因为 1 个代币用于向 paymaster 支付交易费用。

## 常见错误

如果 `use-paymaster.ts` 脚本失败并出现错误 `Failed to submit transaction: Failed to validate the transaction。 原因：验证恢复：Paymaster 验证错误：无法将资金转移到 bootloader`，请尝试向 paymaster 发送额外的 ETH，以便它有足够的资金来支付交易。 您可以使用 [zkSync 门户](https://portal.zksync.io/)。

如果在铸造新 ERC20 代币时 `use-paymaster.ts` 脚本失败并出现错误 `Error: transaction failed`，并且交易在 [zkSync explorer](https://explorer.zksync. io/)，请通过 [我们的 Discord](https://discord.com/invite/px2aR7w) 或 [联系页面](https://zksync.io/contact.html) 与我们联系。 作为一种解决方法，请尝试在交易中包含特定的 `gasLimit` 值。

## 完成项目

您可以在 [这里](https://github.com/matter-labs/custom-paymaster-tutorial) 下载完整的项目。

## 学到更多

- 要了解有关 zkSync 上 L1->L2 交互的更多信息，请查看 [文档](../developer-guides/bridging/l1-l2.md)。
- 要了解有关 `zksync-web3` SDK 的更多信息，请查看其 [文档](../../api/js)。
- 要了解有关 zkSync 安全帽插件的更多信息，请查看他们的 [文档](../../api/hardhat)。
