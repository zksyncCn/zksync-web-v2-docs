# 账户抽象

现在，让我们学习如何部署自定义账户并直接与 [ContractDeployer](../developer-guides/contracts/system-contracts.md#contractdeployer) 系统合约交互。
在本教程中，我们构建了一个部署 2-of-2 多重签名帐户的工厂。

## 先决条件

强烈建议在深入学习本教程之前阅读账户抽象协议的[设计](../developer-guides/aa.md)。

假设您已经熟悉在 zkSync 上部署智能合约。
如果没有，请参考[快速入门](../developer-guides/hello-world.md)。
还建议阅读系统合约的[介绍](../developer-guides/contracts/system-contracts.md)。

## 安装依赖项

我们将使用 zkSync hardhat 插件来开发这个合约。 首先，我们应该为它安装所有依赖项：

```
mkdir custom-aa-tutorial
cd custom-aa-tutorial
yarn init -y
yarn add -D typescript ts-node ethers zksync-web3 hardhat @matterlabs/hardhat-zksync-solc @matterlabs/hardhat-zksync-deploy
```

由于我们正在使用 zkSync 合约，我们还需要安装包含合约和对等依赖项的包：

```
yarn add @matterlabs/zksync-contracts @openzeppelin/contracts @openzeppelin/contracts-upgradeable
```

此外，需要创建 `hardhat.config.ts` 配置文件、`contracts` 和 `deploy` 文件夹，就像在 [快速入门教程](../developer-guides/hello-world.md) 中一样。

## 账户抽象

每个账号都需要实现[IAccount](../developer-guides/aa.md#iaccount-interface)接口。 由于我们正在与签名者建立一个帐户，因此我们还应该实施 [EIP1271](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/83277ff916ac4f58fec072b8f28a252c1245c2f1/contracts/interfaces/IERC1271.sol#L12)。

合同的框架将如下所示：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import '@matterlabs/zksync-contracts/l2/system-contracts/interfaces/IAccount.sol';
import '@matterlabs/zksync-contracts/l2/system-contracts/TransactionHelper.sol';

import "@openzeppelin/contracts/interfaces/IERC1271.sol";

contract TwoUserMultisig is IAccount, IERC1271 {

    modifier onlyBootloader() {
        require(msg.sender == BOOTLOADER_FORMAL_ADDRESS, "Only bootloader can call this method");
        // Continure execution if called from the bootloader.
        _;
    }

    function validateTransaction(bytes32, bytes32 _suggestedSignedHash, Transaction calldata _transaction) external payable override onlyBootloader {
        _validateTransaction(_suggestedSignedHash, _transaction);
    }

    function _validateTransaction(bytes32 _suggestedSignedHash, Transaction calldata _transaction) internal {

    }

    function executeTransaction(bytes32, bytes32, Transaction calldata _transaction) external payable override onlyBootloader {
        _executeTransaction(_transaction);
      }

    function _executeTransaction(Transaction calldata _transaction) internal {

    }

    function executeTransactionFromOutside(Transaction calldata _transaction) external payable {
        _validateTransaction(_transaction);
        _executeTransaction(_transaction);
    }

      bytes4 constant EIP1271_SUCCESS_RETURN_VALUE = 0x1626ba7e;

    function isValidSignature(bytes32 _hash, bytes calldata _signature) public override view returns (bytes4) {
        return EIP1271_SUCCESS_RETURN_VALUE;
    }

    function payForTransaction(bytes32, bytes32, Transaction calldata _transaction) external payable override onlyBootloader {

    }

    function prePaymaster(bytes32, bytes32, Transaction calldata _transaction) external payable override onlyBootloader {

    }

      receive() external payable {
        // If the bootloader called the `receive` function, it likely means
        // that something went wrong and the transaction should be aborted. The bootloader should
        // only interact through the `validateTransaction`/`executeTransaction` methods.
        assert(msg.sender != BOOTLOADER_FORMAL_ADDRESS);
    }
}
```

请注意，只有 [bootloader](../developer-guides/contracts/system-contracts.md#bootloader) 被允许调用 `validateTransaction`/`executeTransaction`/`payForTransaction`/`prePaymaster` 。这是使用 `onlyBootloader` 修饰符的原因。

需要 `executeTransactionFromOutside` 允许外部用户从此帐户发起交易。 实现它的最简单方法是执行与 validateTransaction + executeTransaction 相同的操作。

### 签名验证

首先，我们需要实现签名验证过程。 由于我们正在构建一个双账户多重签名，因此让我们在构造函数中传递其所有者的地址。 在本教程中，我们使用 OpenZeppelin 的“ECDSA”库进行签名验证。

添加以下导入：

```solidity
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
```

另外，将构造函数添加到合约中：

```solidity
address public owner1;
address public owner2;

constructor(address _owner1, address _owner2) {
    owner1 = _owner1;
    owner2 = _owner2;
}
```

然后我们可以通过以下方式实现 isValidSignature 方法：

```solidity
function isValidSignature(bytes32 _hash, bytes calldata _signature) public override view returns (bytes4) {
    // The signature is the concatenation of the ECDSA signatures of the owners
    // Each ECDSA signature is 65 bytes long. That means that the combined signature is 130 bytes long.
    require(_signature.length == 130, 'Signature length is incorrect');

    address recoveredAddr1 = ECDSA.recover(_hash, _signature[0:65]);
    address recoveredAddr2 = ECDSA.recover(_hash, _signature[65:130]);

    require(recoveredAddr1 == owner1);
    require(recoveredAddr2 == owner2);

    return EIP1271_SUCCESS_RETURN_VALUE;
}
```

### 交易验证

接下来让我们实现验证过程。它负责验证交易的签名并增加随机数。 请注意，此方法允许执行的操作有一些限制。 您可以 [此处](../developer-guides/aa.md#limitations-of-the-verification-step) 阅读更多相关信息。

要增加随机数，您应该使` NONCE_HOLDER_SYSTEM_CONTRACT` 系统合约的` incrementNonceIfEquals` 方法。 它获取交易随机数并检查随机数是否与提供的随机数相同。 如果不是，则交易恢复。 否则，随机数增加。

尽管上述要求只允许帐户访问其存储槽，但在`NONCE_HOLDER_SYSTEM_CONTRACT`中访问您的随机数是 [白名单](../developer-guides/aa.md#extending-the-set-of-slots- that-belong-to-a-user) 案例，因为它的行为方式与您的存储相同，所以它恰好在另一个合同中。 要调用 `NONCE_HOLDER_SYSTEM_CONTRACT`，您应该添加以下导入：

```solidity
import '@matterlabs/zksync-contracts/l2/system-contracts/Constants.sol';
```

`TransactionHelper` 库（已在上面的示例中导入）可用于获取应签署的交易的哈希值。 您也可以实现自己的签名方案并使用不同的承诺对交易进行签名，但在本例中我们使用该库提供的哈希。

使用 `TransactionHelper` 库：

```solidity
using TransactionHelper for Transaction;
```

另外，请注意，由于需要在打开 isSystem 标志的情况下调用 `NONCE_HOLDER_SYSTEM_CONTRACT` 非视图方法，因此 [systemCall](https://github.com/matter-labs/v2-testnet-contracts /blob/a3cd3c557208f2cd18e12c41840c5d3728d7f71b/l2/system-contracts/SystemContractsCaller.sol#L55) 应该使用`SystemContractsCaller` 库方法，所以这个库也需要导入：

```solidity
import '@matterlabs/zksync-contracts/l2/system-contracts/SystemContractsCaller.sol';
```

现在我们可以实现 _validateTransaction 方法：

```solidity
function _validateTransaction(bytes32 _suggestedSignedHash, Transaction calldata _transaction) internal {
    // Incrementing the nonce of the account.
    // Note, that reserved[0] by convention is currently equal to the nonce passed in the transaction
    SystemContractsCaller.systemCall(
        uint32(gasleft()),
        address(NONCE_HOLDER_SYSTEM_CONTRACT),
        0,
        abi.encodeCall(INonceHolder.incrementMinNonceIfEquals, (_transaction.reserved[0]))
    );

    bytes32 txHash;
    // While the suggested signed hash is usually provided, it is generally
    // not recommended to rely on it to be present, since in the future
    // there may be tx types with no suggested signed hash.
    if(_suggestedSignedHash == bytes32(0)) {
        txHash = _transaction.encodeHash();
    } else {
        txHash = _suggestedSignedHash;
    }

    require(isValidSignature(txHash, _transaction.signature) == EIP1271_SUCCESS_RETURN_VALUE);
}
```

### 支付交易费用

我们现在应该实现 `payForTransaction` 方法。 `TransactionHelper` 库已经为我们提供了`payToTheBootloader` 方法，该方法将`_transaction.maxFeePerErg * _transaction.ergsLimit` ETH 发送到引导加载程序。 所以实现相当简单：

```solidity
function payForTransaction(bytes32, bytes32, Transaction calldata _transaction) external payable override onlyBootloader {
        bool success = _transaction.payToTheBootloader();
        require(success, "Failed to pay the fee to the operator");
}
```

### 实施`prePaymaster`

虽然通常账户抽象协议允许与付款人交互时执行任意操作，但有一些是来自[常见模式](../developer-guides/aa.md#built-in-paymaster-flows) 的内置支持 EOA。
除非您想从您的帐户中实施或限制某些特定的付款人用例，否则最好使其与 EOA 保持一致。 `TransactionHelper` 库提供了 `processPaymasterInput`，它正是这样做的：像 EOA 一样处理 `prePaymaster` 步骤。

```solidity
function prePaymaster(bytes32, bytes32, Transaction calldata _transaction) external payable override onlyBootloader {
    _transaction.processPaymasterInput();
}
```

### 交易执行

交易执行的最基本实现非常简单：

```solidity
function _executeTransaction(Transaction calldata _transaction) internal {
    uint256 to = _transaction.to;
    // By convention, the `reserved[1]` field is msg.value
    uint256 value = _transaction.reserved[1];
    bytes memory data = _transaction.data;

    bool success;
    assembly {
        success := call(gas(), to, value, add(data, 0x20), mload(data), 0, 0)
    }

    // Needed for the transaction to be correctly processed by the server.
    require(success);
}
```

但是，请注意，调用 ContractDeployer 只能使用 isSystem 调用标志。 为了允许您的用户部署合约，您应该明确地这样做：

```solidity
function _executeTransaction(Transaction calldata _transaction) internal {
    address to = address(uint160(_transaction.to));
    uint256 value = _transaction.reserved[1];
    bytes memory data = _transaction.data;

    if(to == address(DEPLOYER_SYSTEM_CONTRACT)) {
        // We allow calling ContractDeployer with any calldata
        SystemContractsCaller.systemCall(
            uint32(gasleft()),
            to,
            uint128(_transaction.reserved[1]), // By convention, reserved[1] is `value`
            _transaction.data
        );
    } else {
        bool success;
        assembly {
            success := call(gas(), to, value, add(data, 0x20), mload(data), 0, 0)
        }
        require(success);
    }
}
```

请注意，操作员是否会认为交易成功将仅取决于对 executeTransactions 的调用是否成功。 因此，强烈建议为交易加上`require(success)`，让用户得到最好的UX。

### 账户的完整代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@matterlabs/zksync-contracts/l2/system-contracts/interfaces/IAccount.sol";
import "@matterlabs/zksync-contracts/l2/system-contracts/TransactionHelper.sol";

import "@openzeppelin/contracts/interfaces/IERC1271.sol";

// Used for signature validation
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

// Access zkSync system contracts, in this case for nonce validation vs NONCE_HOLDER_SYSTEM_CONTRACT
import "@matterlabs/zksync-contracts/l2/system-contracts/Constants.sol";
// to call non-view method of system contracts
import "@matterlabs/zksync-contracts/l2/system-contracts/SystemContractsCaller.sol";

contract TwoUserMultisig is IAccount, IERC1271 {
    // to get transaction hash
    using TransactionHelper for Transaction;

    // state variables for account owners
    address public owner1;
    address public owner2;

    bytes4 constant EIP1271_SUCCESS_RETURN_VALUE = 0x1626ba7e;

    modifier onlyBootloader() {
        require(
            msg.sender == BOOTLOADER_FORMAL_ADDRESS,
            "Only bootloader can call this method"
        );
        // Continure execution if called from the bootloader.
        _;
    }

    constructor(address _owner1, address _owner2) {
        owner1 = _owner1;
        owner2 = _owner2;
    }

    function validateTransaction(
        bytes32,
        bytes32 _suggestedSignedHash,
        Transaction calldata _transaction
    ) external payable override onlyBootloader {
        _validateTransaction(_suggestedSignedHash, _transaction);
    }

    function _validateTransaction(
        bytes32 _suggestedSignedHash,
        Transaction calldata _transaction
    ) internal {
        // Incrementing the nonce of the account.
        // Note, that reserved[0] by convention is currently equal to the nonce passed in the transaction
        SystemContractsCaller.systemCall(
            uint32(gasleft()),
            address(NONCE_HOLDER_SYSTEM_CONTRACT),
            0,
            abi.encodeCall(
                INonceHolder.incrementMinNonceIfEquals,
                (_transaction.reserved[0])
            )
        );

        bytes32 txHash;
        // While the suggested signed hash is usually provided, it is generally
        // not recommended to rely on it to be present, since in the future
        // there may be tx types with no suggested signed hash.
        if (_suggestedSignedHash == bytes32(0)) {
            txHash = _transaction.encodeHash();
        } else {
            txHash = _suggestedSignedHash;
        }

        require(
            isValidSignature(txHash, _transaction.signature) ==
                EIP1271_SUCCESS_RETURN_VALUE
        );
    }

    function executeTransaction(
        bytes32,
        bytes32,
        Transaction calldata _transaction
    ) external payable override onlyBootloader {
        _executeTransaction(_transaction);
    }

    function _executeTransaction(Transaction calldata _transaction) internal {
        address to = address(uint160(_transaction.to));
        uint256 value = _transaction.reserved[1];
        bytes memory data = _transaction.data;

        if (to == address(DEPLOYER_SYSTEM_CONTRACT)) {
            // We allow calling ContractDeployer with any calldata
            SystemContractsCaller.systemCall(
                uint32(gasleft()),
                to,
                uint128(_transaction.reserved[1]), // By convention, reserved[1] is `value`
                _transaction.data
            );
        } else {
            bool success;
            assembly {
                success := call(
                    gas(),
                    to,
                    value,
                    add(data, 0x20),
                    mload(data),
                    0,
                    0
                )
            }
            require(success);
        }
    }

    function executeTransactionFromOutside(Transaction calldata _transaction)
        external
        payable
    {
        _validateTransaction(bytes32(0), _transaction);

        _executeTransaction(_transaction);
    }

    function isValidSignature(bytes32 _hash, bytes calldata _signature)
        public
        view
        override
        returns (bytes4)
    {
        // The signature is the concatenation of the ECDSA signatures of the owners
        // Each ECDSA signature is 65 bytes long. That means that the combined signature is 130 bytes long.
        require(_signature.length == 130, "Signature length is incorrect");

        address recoveredAddr1 = ECDSA.recover(_hash, _signature[0:65]);
        address recoveredAddr2 = ECDSA.recover(_hash, _signature[65:130]);

        require(recoveredAddr1 == owner1);
        require(recoveredAddr2 == owner2);

        return EIP1271_SUCCESS_RETURN_VALUE;
    }

    function payForTransaction(
        bytes32,
        bytes32,
        Transaction calldata _transaction
    ) external payable override onlyBootloader {
        bool success = _transaction.payToTheBootloader();
        require(success, "Failed to pay the fee to the operator");
    }

    function prePaymaster(
        bytes32,
        bytes32,
        Transaction calldata _transaction
    ) external payable override onlyBootloader {
        _transaction.processPaymasterInput();
    }

    receive() external payable {
        // If the bootloader called the `receive` function, it likely means
        // that something went wrong and the transaction should be aborted. The bootloader should
        // only interact through the `validateTransaction`/`executeTransaction` methods.
        assert(msg.sender != BOOTLOADER_FORMAL_ADDRESS);
    }
}
```

## 代理

现在，让我们构建一个可以部署的账户代理。 请注意，如果我们要部署 AA，我们需要直接与`DEPLOYER_SYSTEM_CONTRACT`进行交互。 对于确定性地址，我们将使用 create2Account 方法。

代码将如下所示：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import '@matterlabs/zksync-contracts/l2/system-contracts/Constants.sol';
import '@matterlabs/zksync-contracts/l2/system-contracts/SystemContractsCaller.sol';

contract AAFactory {
    bytes32 public aaBytecodeHash;

    constructor(bytes32 _aaBytecodeHash) {
        aaBytecodeHash = _aaBytecodeHash;
    }

    function deployAccount(
        bytes32 salt,
        address owner1,
        address owner2
    ) external returns (address accountAddress) {
        bytes memory returnData = SystemContractsCaller.systemCall(
            uint32(gasleft()),
            address(DEPLOYER_SYSTEM_CONTRACT),
            0,
            abi.encodeCall(
                DEPLOYER_SYSTEM_CONTRACT.create2Account,
                (salt, aaBytecodeHash, abi.encode(owner1, owner2))
            )
        );

        (accountAddress, ) = abi.decode(returnData, (address, bytes));
    }
}
```

请注意，在 zkSync 上，部署不是通过字节码完成的，而是通过字节码哈希完成的。 字节码本身通过`factoryDeps`字段传递给操作员。 请注意，`_aaBytecodeHash` 必须专门形成：

- 首先，它用 sha256 散列。
- 然后，前两个字节被字节码的长度替换为 32 字节字。

您无需担心，因为我们的 SDK 提供了一个内置方法来执行此操作，如下所述。

## 部署代理

首先，我们需要创建一个部署脚本。 创建 在`deploy` 文件夹在创建一个文件：`deploy-factory.ts`。 将以下部署脚本放在那里：

```ts
import { utils, Wallet } from "zksync-web3";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

export default async function (hre: HardhatRuntimeEnvironment) {
  const wallet = new Wallet("<PRIVATE-KEY>");
  const deployer = new Deployer(hre, wallet);
  const factoryArtifact = await deployer.loadArtifact("AAFactory");
  const aaArtifact = await deployer.loadArtifact("TwoUserMultisig");

  // Deposit some funds to L2 in order to be able to perform L2 transactions.
  // You can remove the depositing step if the `wallet` has enough funds on zkSync
  const depositAmount = ethers.utils.parseEther("0.001");
  const depositHandle = await deployer.zkWallet.deposit({
    to: deployer.zkWallet.address,
    token: utils.ETH_ADDRESS,
    amount: depositAmount,
  });
  await depositHandle.wait();

  // Getting the bytecodeHash of the account
  const bytecodeHash = utils.hashBytecode(aaArtifact.bytecode);

  const factory = await deployer.deploy(factoryArtifact, [bytecodeHash], undefined, [
    // Since the factory requires the code of the multisig to be available,
    // we should pass it here as well.
    aaArtifact.bytecode,
  ]);

  console.log(`AA factory address: ${factory.address}`);
}
```

为了部署工厂，您应该编译合约并运行脚本：

```
yarn hardhat compile
yarn hardhat deploy-zksync --script deploy-factory.ts
```

输出如下：

```
AA factory address: 0x9db333Cb68Fb6D317E3E415269a5b9bE7c72627Ds
```

请注意，每次运行的地址都会不同。

## 使用账户

### 部署账户

现在，让我们部署一个账户并使用它启动一个新交易。 在本节中，我们假设您已经在 zkSync 上拥有一个足够资金的 EOA 账户。
在 `deploy` 文件夹中创建一个文件 `deploy-multisig.ts`，我们将在其中放置脚本。

首先，让我们部署 AA。 这将是对 `deployAccount` 函数的调用：

```ts
import { utils, Wallet, Provider, EIP712Signer, types } from "zksync-web3";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";

// Put the address of your AA factory
const AA_FACTORY_ADDRESS = "<YOUR_FACTORY_ADDRESS>";

// An example of a deploy script deploys and calls a simple contract.
export default async function (hre: HardhatRuntimeEnvironment) {
  const provider = new Provider(hre.config.zkSyncDeploy.zkSyncNetwork);
  const wallet = new Wallet("<PRIVATE-KEY>").connect(provider);
  const factoryArtifact = await hre.artifacts.readArtifact("AAFactory");

  const aaFactory = new ethers.Contract(AA_FACTORY_ADDRESS, factoryArtifact.abi, wallet);

  // The two owners of the multisig
  const owner1 = Wallet.createRandom();
  const owner2 = Wallet.createRandom();

  // For the simplicity of the tutorial, we will use zero hash as salt
  const salt = ethers.constants.HashZero;

  const tx = await aaFactory.deployAccount(salt, owner1.address, owner2.address);
  await tx.wait();

  // Getting the address of the deployed contract
  const abiCoder = new ethers.utils.AbiCoder();
  const multisigAddress = utils.create2Address(
    AA_FACTORY_ADDRESS,
    await aaFactory.aaBytecodeHash(),
    salt,
    abiCoder.encode(["address", "address"], [owner1.address, owner2.address])
  );
  console.log(`Deployed on address ${multisigAddress}`);
}
```

_注意，zkSync 与 Ethereum_ 有不同的地址规则。 您应该始终使用 `zksync-web3` SDK 的 `createAddress` 和 `create2Address` 的实用方法。

### 从此账户开始交易

在部署的账户可以进行交易之前，我们需要对其进行充值：

```ts
await(
  await wallet.sendTransaction({
    to: multisigAddress,
    // You can increase the amount of ETH sent to the multisig
    value: ethers.utils.parseEther("0.003"),
  })
).wait();
```

现在，作为示例，让我们尝试部署一个新的多重签名，但交易的发起者将是我们在上一部分中部署的帐户：

```ts
let aaTx = await aaFactory.populateTransaction.deployAccount(salt, Wallet.createRandom().address, Wallet.createRandom().address);
```

然后，我们需要填写所有交易字段：

```ts
const gasLimit = await provider.estimateGas(aaTx);
const gasPrice = await provider.getGasPrice();

aaTx = {
  ...aaTx,
  from: multisigAddress,
  gasLimit: gasLimit,
  gasPrice: gasPrice,
  chainId: (await provider.getNetwork()).chainId,
  nonce: await provider.getTransactionCount(multisigAddress),
  type: 113,
  customData: {
    // Note, that we are using the `DEFAULT_ERGS_PER_PUBDATA_LIMIT`
    ergsPerPubdata: utils.DEFAULT_ERGS_PER_PUBDATA_LIMIT,
  } as types.Eip712Meta,
  value: ethers.BigNumber.from(0),
};
```

::: 关于 gasLimit 的注意事项

目前，我们希望 `gasLimit` 涵盖验证和执行步骤。 目前，`estimateGas` 返回的 erg 数量是 `execution_ergs + 20000`，其中 `20000` 大致等于 defaultAA 收取费用和验证签名所需的开销。 如果你的 AA 有一个非常昂贵的验证步骤，你应该在 `gasLimit` 中添加一些常量。

:::

然后，我们需要对交易进行签名，并在交易的 customData 中提供 `aaParamas`：

```ts
const signedTxHash = EIP712Signer.getSignedDigest(aaTx);

const signature = ethers.utils.concat([
  // Note, that `signMessage` wouldn't work here, since we don't want
  // the signed hash to be prefixed with `\x19Ethereum Signed Message:\n`
  ethers.utils.joinSignature(owner1._signingKey().signDigest(signedTxHash)),
  ethers.utils.joinSignature(owner2._signingKey().signDigest(signedTxHash)),
]);

aaTx.customData = {
  ...aaTx.customData,
  customSignature: signature,
};
```

现在，我们准备发送交易：

```ts
console.log(`The multisig's nonce before the first tx is ${await provider.getTransactionCount(multisigAddress)}`);
const sentTx = await provider.sendTransaction(utils.serialize(aaTx));
await sentTx.wait();

// Checking that the nonce for the account has increased
console.log(`The multisig's nonce after the first tx is ${await provider.getTransactionCount(multisigAddress)}`);
```

### 完整示例

```ts
import { utils, Wallet, Provider, EIP712Signer, types } from "zksync-web3";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";

// Put the address of your AA factory
const AA_FACTORY_ADDRESS = "<YOUR_FACTORY_ADDRESS>";

export default async function (hre: HardhatRuntimeEnvironment) {
  const provider = new Provider(hre.config.zkSyncDeploy.zkSyncNetwork);
  const wallet = new Wallet("<YOUR_PRIVATE_KEY>").connect(provider);
  const factoryArtifact = await hre.artifacts.readArtifact("AAFactory");

  const aaFactory = new ethers.Contract(AA_FACTORY_ADDRESS, factoryArtifact.abi, wallet);

  // The two owners of the multisig
  const owner1 = Wallet.createRandom();
  const owner2 = Wallet.createRandom();

  // For the simplicity of the tutorial, we will use zero hash as salt
  const salt = ethers.constants.HashZero;

  const tx = await aaFactory.deployAccount(salt, owner1.address, owner2.address);
  await tx.wait();

  // Getting the address of the deployed contract
  const abiCoder = new ethers.utils.AbiCoder();
  const multisigAddress = utils.create2Address(
    AA_FACTORY_ADDRESS,
    await aaFactory.aaBytecodeHash(),
    salt,
    abiCoder.encode(["address", "address"], [owner1.address, owner2.address])
  );
  console.log(`Multisig deployed on address ${multisigAddress}`);

  await (
    await wallet.sendTransaction({
      to: multisigAddress,
      // You can increase the amount of ETH sent to the multisig
      value: ethers.utils.parseEther("0.001"),
    })
  ).wait();

  let aaTx = await aaFactory.populateTransaction.deployAccount(salt, Wallet.createRandom().address, Wallet.createRandom().address);

  const gasLimit = await provider.estimateGas(aaTx);
  const gasPrice = await provider.getGasPrice();

  aaTx = {
    ...aaTx,
    from: multisigAddress,
    gasLimit: gasLimit,
    gasPrice: gasPrice,
    chainId: (await provider.getNetwork()).chainId,
    nonce: await provider.getTransactionCount(multisigAddress),
    type: 113,
    customData: {
      ergsPerPubdata: utils.DEFAULT_ERGS_PER_PUBDATA_LIMIT,
    } as types.Eip712Meta,
    value: ethers.BigNumber.from(0),
  };
  const signedTxHash = EIP712Signer.getSignedDigest(aaTx);

  const signature = ethers.utils.concat([
    // Note, that `signMessage` wouldn't work here, since we don't want
    // the signed hash to be prefixed with `\x19Ethereum Signed Message:\n`
    ethers.utils.joinSignature(owner1._signingKey().signDigest(signedTxHash)),
    ethers.utils.joinSignature(owner2._signingKey().signDigest(signedTxHash)),
  ]);

  aaTx.customData = {
    ...aaTx.customData,
    customSignature: signature,
  };

  console.log(`The multisig's nonce before the first tx is ${await provider.getTransactionCount(multisigAddress)}`);
  const sentTx = await provider.sendTransaction(utils.serialize(aaTx));
  await sentTx.wait();

  // Checking that the nonce for the account has increased
  console.log(`The multisig's nonce after the first tx is ${await provider.getTransactionCount(multisigAddress)}`);
}
```

要运行脚本，请使用以下命令：

```
yarn hardhat deploy-zksync --script deploy-multisig.ts
```

输出应该大致如下：

```
Multisig deployed on address 0xCEBc59558938bccb43A6C94769F87bBdb770E956
The multisig's nonce before the first tx is 0
The multisig's nonce after the first tx is 1
```

::: 提示

如果您收到错误提示`Not enough balance to cover the fee。`，请尝试增加发送到多重签名钱包的 ETH 数量，以便它有足够的资金来支付交易费用。

:::

## 完成项目

您可以在 [这里](https://github.com/matter-labs/custom-aa-tutorial) 下载完整的项目。

## Learn more

- 要了解有关 zkSync 上的 L1->L2 交互的更多信息，请查看 [文档](../developer-guides/bridging/l1-l2.md)。
- 要了解有关 `zksync-web3` SDK 的更多信息，请查看 [文档。
- 要了解有关 zkSync 安全帽插件的更多信息，请查看他们的 [文档](../../api/hardhat)。
