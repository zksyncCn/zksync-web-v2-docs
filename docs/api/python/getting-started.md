## 入门

在本指南中，我们将演示如何：

1. 连接到 zkSync 网络。
2. 将来自以太坊的资产存入 zkSync。
3. 检查余额。
4. 转账和提取资金（原生代币和ERC20代币）。
5. 部署智能合约。
6. 使用 create2 部署智能合约。

## 准备工作

本指南假定您熟悉 [Python](https://docs.python.org/3/) 编程语言。

# 安装

要使用 `pip install` 命令安装 SDK，请运行以下命令：

```py
pip install zksync2
```

## 运行 SDK 示例

要开始使用这个SDK，您只需要上传一个提供的配置。

```py
from web3 import Web3
from zkSync2.module.module_builder import zkSyncBuilder

URL_TO_ETH_NETWORK = "GOERLI_HTTPS_RPC"
ZKSYNC_NETWORK_URL = "https://zksync2-testnet.zksync.dev"

eth_web3 = Web3(Web3.HTTPProvider(URL_TO_ETH_NETWORK))
zkSync_web3 = zkSyncBuilder.build(ZKSYNC_NETWORK_URL)
```

## 以太坊签名

:::: warning

⚠️ 切勿将私钥提交到文件跟踪历史记录中，否则您的帐户可能会被盗用。

:::

以太坊签名者由来自“zkSync2.signer.eth_signer”的“PrivateKeyEthSigner”抽象类表示。

```py
from eth_account import Account
from eth_account.signers.local import LocalAccount
from zkSync2.signer.eth_signer import PrivateKeyEthSigner
import os

PRIVATE_KEY = os.environ.get("YOUR_PRIVATE_KEY")

account: LocalAccount = Account.from_key(PRIVATE_KEY)
signer = PrivateKeyEthSigner(account, chain_id)
```

## 存入资金

此示例显示如何将资金存入地址。

```py
from web3 import Web3
from web3.middleware import geth_poa_middleware
from eth_account import Account
from eth_account.signers.local import LocalAccount
from zkSync2.manage_contracts.gas_provider import StaticGasProvider
from zkSync2.module.module_builder import zkSyncBuilder
from zkSync2.core.types import Token
from zkSync2.provider.eth_provider import EthereumProvider


def deposit():
    #geth_poa_middleware is used to connect to geth --dev.
    eth_web3.middleware_onion.inject(geth_poa_middleware, layer=0)

    #calculate  gas fees
    gas_provider = StaticGasProvider(Web3.toWei(1, "gwei"), 555000)

    #Create the ethereum provider for interacting with ethereum node, initialize zkSync signer and deposit funds.
    eth_provider = EthereumProvider.build_ethereum_provider(zkSync=zkSync_web3,
                                                            eth=eth_web3,
                                                            account=account,
                                                            gas_provider=gas_provider)
    tx_receipt = eth_provider.deposit(Token.create_eth(),
                                    eth_web3.toWei("YOUR_AMOUNT_OF_ETH", "ether"),
                                    account.address)
    # Show the output of the transaction details.
    print(f"tx status: {tx_receipt['status']}")


if __name__ == "__main__":
    deposit()
```

## 查看余额

这个例子展示了如何在 zkSync 网络上查看你的余额。

```py
from eth_account import Account
from eth_account.signers.local import LocalAccount
from zkSync2.module.module_builder import zkSyncBuilder
from zkSync2.core.types import EthBlockParams


def get_account_balance():
    ZKSYNC_NETWORK_URL: str = 'https://zkSync2-testnet.zkSync.dev'
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    zkSync_web3 = zkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    zk_balance = zkSync_web3.zkSync.get_balance(account.address, EthBlockParams.LATEST.value)
    print(f"zkSync balance: {zk_balance}")


if __name__ == "__main__":
    get_account_balance()
```

## 转账

这个例子展示了如何在 zkSync 网络上执行从一个账户到另一个账户的转账。

```py
from eth_typing import HexStr
from web3 import Web3
from zkSync2.module.request_types import create_function_call_transaction
from zkSync2.module.module_builder import zkSyncBuilder
from zkSync2.core.types import ZkBlockParams
from eth_account import Account
from eth_account.signers.local import LocalAccount

from zkSync2.signer.eth_signer import PrivateKeyEthSigner
from zkSync2.transaction.transaction712 import Transaction712


def transfer_to_self():
    ZKSYNC_NETWORK_URL: str = 'https://zkSync2-testnet.zkSync.dev'
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    zkSync_web3 = zkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    chain_id = zkSync_web3.zkSync.chain_id
    signer = PrivateKeyEthSigner(account, chain_id)

    nonce = zkSync_web3.zkSync.get_transaction_count(account.address, ZkBlockParams.COMMITTED.value)
    tx = create_function_call_transaction(from_=account.address,
                                          to=account.address,
                                          ergs_price=0,
                                          ergs_limit=0,
                                          data=HexStr("0x"))
    estimate_gas = zkSync_web3.zkSync.eth_estimate_gas(tx)
    gas_price = zkSync_web3.zkSync.gas_price

    print(f"Fee for transaction is: {estimate_gas * gas_price}")

    tx_712 = Transaction712(chain_id=chain_id,
                            nonce=nonce,
                            gas_limit=estimate_gas,
                            to=tx["to"],
                            value=Web3.toWei(0.01, 'ether'),
                            data=tx["data"],
                            maxPriorityFeePerGas=100000000,
                            maxFeePerGas=gas_price,
                            from_=account.address,
                            meta=tx["eip712Meta"])

    singed_message = signer.sign_typed_data(tx_712.to_eip712_struct())
    msg = tx_712.encode(singed_message)
    tx_hash = zkSync_web3.zkSync.send_raw_transaction(msg)
    tx_receipt = zkSync_web3.zkSync.wait_for_transaction_receipt(tx_hash, timeout=240, poll_latency=0.5)
    print(f"tx status: {tx_receipt['status']}")


if __name__ == "__main__":
    transfer_to_self()
```

## 转账（ERC20 代币）

这个例子展示了如何将 ERC20 代币转移到你在 zkSync 网络上的账户。

```py
from zkSync2.module.request_types import create_function_call_transaction
from zkSync2.manage_contracts.erc20_contract import ERC20FunctionEncoder
from zkSync2.module.module_builder import zkSyncBuilder
from zkSync2.core.types import ZkBlockParams
from eth_account import Account
from eth_account.signers.local import LocalAccount
from zkSync2.signer.eth_signer import PrivateKeyEthSigner
from zkSync2.transaction.transaction712 import Transaction712


def transfer_erc20_token():
    ZKSYNC_NETWORK_URL: str = 'https://zkSync2-testnet.zkSync.dev'
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    zkSync_web3 = zkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    chain_id = zkSync_web3.zkSync.chain_id
    signer = PrivateKeyEthSigner(account, chain_id)

    nonce = zkSync_web3.zkSync.get_transaction_count(account.address, ZkBlockParams.COMMITTED.value)
    tokens = zkSync_web3.zkSync.zks_get_confirmed_tokens(0, 100)
    not_eth_tokens = [x for x in tokens if not x.is_eth()]
    token_address = not_eth_tokens[0].l2_address

    erc20_encoder = ERC20FunctionEncoder(zkSync_web3)
    transfer_params = [account.address, 0]
    call_data = erc20_encoder.encode_method("transfer", args=transfer_params)

    tx = create_function_call_transaction(from_=account.address,
                                          to=token_address,
                                          ergs_price=0,
                                          ergs_limit=0,
                                          data=call_data)
    estimate_gas = zkSync_web3.zkSync.eth_estimate_gas(tx)
    gas_price = zkSync_web3.zkSync.gas_price

    print(f"Fee for transaction is: {estimate_gas * gas_price}")

    tx_712 = Transaction712(chain_id=chain_id,
                            nonce=nonce,
                            gas_limit=estimate_gas,
                            to=tx["to"],
                            value=tx["value"],
                            data=tx["data"],
                            maxPriorityFeePerGas=100000000,
                            maxFeePerGas=gas_price,
                            from_=account.address,
                            meta=tx["eip712Meta"])
    singed_message = signer.sign_typed_data(tx_712.to_eip712_struct())
    msg = tx_712.encode(singed_message)
    tx_hash = zkSync_web3.zkSync.send_raw_transaction(msg)
    tx_receipt = zkSync_web3.zkSync.wait_for_transaction_receipt(tx_hash, timeout=240, poll_latency=0.5)
    print(f"tx status: {tx_receipt['status']}")


if __name__ == "__main__":
    transfer_erc20_token()
```

## 提取资金

此示例显示如何将资金提取到您的帐户中。

```py
from decimal import Decimal
from eth_typing import HexStr
from zkSync2.module.request_types import create_function_call_transaction
from zkSync2.manage_contracts.l2_bridge import L2BridgeEncoder
from zkSync2.module.module_builder import zkSyncBuilder
from zkSync2.core.types import Token, ZkBlockParams, BridgeAddresses
from eth_account import Account
from eth_account.signers.local import LocalAccount

from zkSync2.signer.eth_signer import PrivateKeyEthSigner
from zkSync2.transaction.transaction712 import Transaction712


def withdraw():
    ZKSYNC_NETWORK_URL: str = 'https://zkSync2-testnet.zkSync.dev'
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    zkSync_web3 = zkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    chain_id = zkSync_web3.zkSync.chain_id
    signer = PrivateKeyEthSigner(account, chain_id)
    ETH_TOKEN = Token.create_eth()

    nonce = zkSync_web3.zkSync.get_transaction_count(account.address, ZkBlockParams.COMMITTED.value)
    bridges: BridgeAddresses = zkSync_web3.zkSync.zks_get_bridge_contracts()

    l2_func_encoder = L2BridgeEncoder(zkSync_web3)
    call_data = l2_func_encoder.encode_function(fn_name="withdraw", args=[
        account.address,
        ETH_TOKEN.l2_address,
        ETH_TOKEN.to_int(Decimal("0.001"))
    ])

    tx = create_function_call_transaction(from_=account.address,
                                          to=bridges.l2_eth_default_bridge,
                                          ergs_limit=0,
                                          ergs_price=0,
                                          data=HexStr(call_data))
    estimate_gas = zkSync_web3.zkSync.eth_estimate_gas(tx)
    gas_price = zkSync_web3.zkSync.gas_price

    print(f"Fee for transaction is: {estimate_gas * gas_price}")

    tx_712 = Transaction712(chain_id=chain_id,
                            nonce=nonce,
                            gas_limit=estimate_gas,
                            to=tx["to"],
                            value=tx["value"],
                            data=tx["data"],
                            maxPriorityFeePerGas=100000000,
                            maxFeePerGas=gas_price,
                            from_=account.address,
                            meta=tx["eip712Meta"])

    singed_message = signer.sign_typed_data(tx_712.to_eip712_struct())
    msg = tx_712.encode(singed_message)
    tx_hash = zkSync_web3.zkSync.send_raw_transaction(msg)
    tx_receipt = zkSync_web3.zkSync.wait_for_transaction_receipt(tx_hash, timeout=240, poll_latency=0.5)
    print(f"tx status: {tx_receipt['status']}")


if __name__ == "__main__":
    withdraw()
```

## 部署智能合约

使用 zkSync，您可以使用 `create` 方法部署合约，只需将合约构建为二进制格式并将其部署到 zkSync 网络。

在接下来的步骤中，我们将指导您了解它的工作原理。

### 第一步：创建合约

这是一个简单的合约：

```solidity
pragma solidity ^0.8.0;

contract Counter {
    uint256 value;

    function increment(uint256 x) public {
        value += x;
    }

    function get() public view returns (uint256) {
        return value;
    }
}
```

> 只能通过 zkSync 编译器编译！

编译后必须有2个文件：

- contract binary representation
- contract ABI in JSON format

### 步骤 2：部署合约

要部署合约，需要合约 ABI 以标准的 `web3` 方式调用其方法。

在某些情况下，您需要在部署之前获取合约地址。

这是您将如何做的一个例子。

```py
import json
from pathlib import Path
from eth_typing import HexStr
from web3 import Web3
from web3.types import TxParams
from zkSync2.module.request_types import create_contract_transaction
from zkSync2.manage_contracts.contract_deployer import ContractDeployer
from zkSync2.manage_contracts.nonce_holder import NonceHolder
from zkSync2.module.module_builder import zkSyncBuilder
from zkSync2.core.types import ZkBlockParams, EthBlockParams
from eth_account import Account
from eth_account.signers.local import LocalAccount
from zkSync2.signer.eth_signer import PrivateKeyEthSigner
from zkSync2.transaction.transaction712 import Transaction712


def read_binary(p: Path) -> bytes:
    with p.open(mode='rb') as contact_file:
        data = contact_file.read()
        return data


def get_abi(p: Path):
    with p.open(mode='r') as json_f:
        return json.load(json_f)


class CounterContractEncoder:
    def __init__(self, web3: Web3, bin_path: Path, abi_path: Path):
        self.web3 = web3
        self.counter_contract = self.web3.eth.contract(abi=get_abi(abi_path),
                                                       bytecode=read_binary(bin_path))

    def encode_method(self, fn_name, args: list) -> HexStr:
        return self.counter_contract.encodeABI(fn_name, args)


def deploy_contract_create():
    ZKSYNC_NETWORK_URL: str = 'https://zkSync2-testnet.zkSync.dev'
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    zkSync_web3 = zkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    chain_id = zkSync_web3.zkSync.chain_id
    signer = PrivateKeyEthSigner(account, chain_id)

    counter_contract_bin = read_binary("PATH_TO_BINARY_COMPILED_CONTRACT")

    nonce = zkSync_web3.zkSync.get_transaction_count(account.address, EthBlockParams.PENDING.value)
    nonce_holder = NonceHolder(zkSync_web3, account)
    deployment_nonce = nonce_holder.get_deployment_nonce(account.address)
    deployer = ContractDeployer(zkSync_web3)
    precomputed_address = deployer.compute_l2_create_address(account.address, deployment_nonce)

    print(f"precomputed address: {precomputed_address}")

    tx = create_contract_transaction(web3=zkSync_web3,
                                     from_=account.address,
                                     ergs_limit=0,
                                     ergs_price=0,
                                     bytecode=counter_contract_bin)

    estimate_gas = zkSync_web3.zkSync.eth_estimate_gas(tx)
    gas_price = zkSync_web3.zkSync.gas_price
    print(f"Fee for transaction is: {estimate_gas * gas_price}")

    tx_712 = Transaction712(chain_id=chain_id,
                            nonce=nonce,
                            gas_limit=estimate_gas,
                            to=tx["to"],
                            value=tx["value"],
                            data=tx["data"],
                            maxPriorityFeePerGas=100000000,
                            maxFeePerGas=gas_price,
                            from_=account.address,
                            meta=tx["eip712Meta"])

    singed_message = signer.sign_typed_data(tx_712.to_eip712_struct())
    msg = tx_712.encode(singed_message)
    tx_hash = zkSync_web3.zkSync.send_raw_transaction(msg)
    tx_receipt = zkSync_web3.zkSync.wait_for_transaction_receipt(tx_hash, timeout=240, poll_latency=0.5)
    print(f"tx status: {tx_receipt['status']}")

    contract_address = tx_receipt["contractAddress"]
    print(f"contract address: {contract_address}")

    counter_contract_encoder = CounterContractEncoder(zkSync_web3, "PATH_TO_BINARY_COMPILED_CONTRACT",
                                                      "PATH_TO_CONTRACT_ABI")

    call_data = counter_contract_encoder.encode_method(fn_name="get", args=[])
    eth_tx: TxParams = {
        "from": account.address,
        "to": contract_address,
        "data": call_data
    }
    # Value is type dependent so might need to be converted to corresponded type under Python
    eth_ret = zkSync_web3.zkSync.call(eth_tx, ZkBlockParams.COMMITTED.value)
    converted_result = int.from_bytes(eth_ret, "big", signed=True)
    print(f"Call method for deployed contract, address: {contract_address}, value: {converted_result}")


if __name__ == "__main__":
    deploy_contract_create()
```

## 使用 Create2 部署智能合约

使用 [CREATE2](https://eips.ethereum.org/EIPS/eip-1014) 操作码使您能够预测将部署合约的地址，而无需这样做，从而改善用户引导。

与上面的 `create` 方法类似，这里是如何使用 zkSync 上的 `create2` 方法部署合约的实现。

```py
import os
import json
from pathlib import Path
from eth_typing import HexStr
from web3 import Web3
from web3.types import TxParams
from zkSync2.module.request_types import create2_contract_transaction
from zkSync2.manage_contracts.contract_deployer import ContractDeployer
from zkSync2.module.module_builder import zkSyncBuilder
from zkSync2.core.types import ZkBlockParams, EthBlockParams
from eth_account import Account
from eth_account.signers.local import LocalAccount
from zkSync2.signer.eth_signer import PrivateKeyEthSigner
from zkSync2.transaction.transaction712 import Transaction712

#Generate random salt(an arbitrary value provided by the sender)
def generate_random_salt() -> bytes:
    return os.urandom(32)

#Open the file
def read_binary(p: Path) -> bytes:
    with p.open(mode='rb') as contact_file:
        data = contact_file.read()
        return data

#Get the contract ABI
def get_abi(p: Path):
    with p.open(mode='r') as json_f:
        return json.load(json_f)


class CounterContractEncoder:
    def __init__(self, web3: Web3, bin_path: Path, abi_path: Path):
        self.web3 = web3
        self.counter_contract = self.web3.eth.contract(abi=get_abi(abi_path),
                                                       bytecode=read_binary(bin_path))

    def encode_method(self, fn_name, args: list) -> HexStr:
        return self.counter_contract.encodeABI(fn_name, args)

#Deploy the contract
def deploy_contract_create2():
    ZKSYNC_NETWORK_URL: str = 'https://zkSync2-testnet.zkSync.dev'
    account: LocalAccount = Account.from_key('YOUR_PRIVATE_KEY')
    zkSync_web3 = zkSyncBuilder.build(ZKSYNC_NETWORK_URL)
    chain_id = zkSync_web3.zkSync.chain_id
    signer = PrivateKeyEthSigner(account, chain_id)

    counter_contract_bin = read_binary("PATH_TO_BINARY_COMPILED_CONTRACT")

    nonce = zkSync_web3.zkSync.get_transaction_count(account.address, EthBlockParams.PENDING.value)
    deployer = ContractDeployer(zkSync_web3)
    random_salt = generate_random_salt()
    precomputed_address = deployer.compute_l2_create2_address(sender=account.address,
                                                              bytecode=counter_contract_bin,
                                                              constructor=b'',
                                                              salt=random_salt)
    print(f"precomputed address: {precomputed_address}")

    tx = create2_contract_transaction(web3=zkSync_web3,
                                      from_=account.address,
                                      ergs_price=0,
                                      ergs_limit=0,
                                      bytecode=counter_contract_bin,
                                      salt=random_salt)
    estimate_gas = zkSync_web3.zkSync.eth_estimate_gas(tx)
    gas_price = zkSync_web3.zkSync.gas_price
    print(f"Fee for transaction is: {estimate_gas * gas_price}")

    tx_712 = Transaction712(chain_id=chain_id,
                            nonce=nonce,
                            gas_limit=estimate_gas,
                            to=tx["to"],
                            value=tx["value"],
                            data=tx["data"],
                            maxPriorityFeePerGas=100000000,
                            maxFeePerGas=gas_price,
                            from_=account.address,
                            meta=tx["eip712Meta"])
    singed_message = signer.sign_typed_data(tx_712.to_eip712_struct())
    msg = tx_712.encode(singed_message)
    tx_hash = zkSync_web3.zkSync.send_raw_transaction(msg)
    tx_receipt = zkSync_web3.zkSync.wait_for_transaction_receipt(tx_hash, timeout=240, poll_latency=0.5)
    print(f"tx status: {tx_receipt['status']}")

    contract_address = tx_receipt["contractAddress"]
    print(f"contract address: {contract_address}")

    counter_contract_encoder = CounterContractEncoder(zkSync_web3, "CONTRACT_BIN_PATH", "CONTRACT_ABI_PATH")
    call_data = counter_contract_encoder.encode_method(fn_name="get", args=[])
    eth_tx: TxParams = {
        "from": account.address,
        "to": contract_address,
        "data": call_data
    }
    eth_ret = zkSync_web3.zkSync.call(eth_tx, ZkBlockParams.COMMITTED.value)
    result = int.from_bytes(eth_ret, "big", signed=True)
    print(f"Call method for deployed contract, address: {contract_address}, value: {result}")


if __name__ == "__main__":
    deploy_contract_create2()
```

::: 注意

⚠️ 这一部分文档仍在更新中，不久将会更新更详细的信息。

:::