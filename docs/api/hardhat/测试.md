# 本地测试

有时出于网络延迟或费用原因，需要在本地环境中测试合约。

zkSync 团队为此提供了一个 dockerized 本地设置。

## 先决条件

您需要在计算机上安装 `Docker` 和 `docker-compose`。 在此处找到 [安装指南](https://docs.docker.com/get-docker/)

本指南假定您熟悉 `zkSync Hardhat` 插件。 如果您是使用 `Hardhat` 在 zkSync 上进行新开发，请查看[入门部分](./getting-started.md)。

## 安装测试环境

使用以下命令下载 dockerized 项目：

```
git clone https://github.com/matter-labs/local-setup.git
```

## 启动本地节点

要在本地运行 zkSync，请运行 `start.sh` 脚本：

```
cd local-setup
./start.sh
```

此命令将启动三个 docker 容器：

- Postgres（用作 zkSync 的数据库）。
- 本地 Geth 节点（用作 zkSync 的 L1）。
- zkSync 节点本身。

默认情况下，HTTP JSON-RPC API 将在端口 `3050`上运行，而 WS API 将在端口`3051`上运行。

： 警告

请注意，第一个`start.sh`脚本调用不间断很重要。 如果您在引导过程意外停止后遇到任何问题，您应该[重置](#resetting-the-zksync-state) 本地 zkSync 状态并重试。

:::

## 重置 zkSync 状态

要重置 zkSync 状态， 请运行`./clear.sh` 脚本:

```
./clear.sh
```

请注意，您在运行此命令时可能会收到"权限被拒绝 " 的错误提示。 在这种情况下，您应该以 root 权限运行它：

```
sudo ./clear.sh
```

## 富有的钱包

本地 zkSync 设置带有一些 “富有“的钱包，在 L1 和 L2 上都有大量的 ETH。

 [此处](https://github.com/matter-labs/local-setup/blob/main/rich-wallets.json) 可以找到具有相应私钥的这些帐户地址的完整列表

::: 警告  富有的钱包只有ETH，**如果你需要用 ERC20代币 进行测试，你应该自己部署**。

如果你希望本地节点再次附带预部署的代币，请在我们的[discord](https://discord.gg/px2aR7w)上告诉我们，这样我们就可以相应地安排优先级。

:::

## 使用自定义数据库或以太坊节点

要使用自定义的 Postgres 数据库或第1层节点，你应该改变 docker-compose 文件中的环境参数：

```yml
environment:
  - DATABASE_URL=postgres://postgres@postgres/zksync_local
  - ETH_CLIENT_WEB3_URL=http://geth:8545
```

- `DATABASE_URL` 是指向Postgres数据库的URL。
- `ETH_CLIENT_WEB3_URL` 是 L1 节点的 HTTP JSON-RPC 接口的 URL。

## 使用 `mocha` + `chai`进行测试

由于在 `hardhat.config.ts` 中提供了 zkSync 节点 URL，因此使用不同 URL 进行部署和本地测试的最佳方式是使用环境变量。 标准方法是在调用测试之前设置 `NODE_ENV=test` 环境变量。

### 项目设置

1. 按照 [入门指南](./getting-started.md) 作为参考，创建一个新的Hardhat项目。

2. 要安装测试库，请运行以下命令：

```
yarn add -D mocha chai @types/mocha @types/chai
```

3. 将一下内容添加到您的根目录下 `package.json`中：

```json
"scripts": {
    "test": "NODE_ENV=test hardhat test"
}
```

这将在 `Hardhat` 环境中启动运行测试，并将`NODE_ENV`环境变量设置为`test`

### 配置

4. 修改 `hardhat.config.ts` ，以使用本地节点进行测试:

```typescript
require("@matterlabs/hardhat-zksync-deploy");
require("@matterlabs/hardhat-zksync-solc");

// changes endpoint depending on environment variable
const zkSyncDeploy =
  process.env.NODE_ENV == "test"
    ? {
        zkSyncNetwork: "http://localhost:3050",
        ethNetwork: "http://localhost:8545",
      }
    : {
        zkSyncNetwork: "https://zksync2-testnet.zksync.dev",
        ethNetwork: "goerli",
      };

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
  // load endpoints
  zkSyncDeploy,
  solidity: {
    version: "0.8.11",
  },
  networks: {
    hardhat: {
      zksync: true,
    },
  },
};
```

创建一个 `test` 文件夹，测试将保存在其中。

### 编写测试文件

5. 现在你可以编写你的第一个测试了！ 使用以下代码创建一个 `test/main.test.ts` 文件：

```ts
import { expect } from "chai";
import { Wallet, Provider, Contract } from "zksync-web3";
import * as hre from "hardhat";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

const RICH_WALLET_PK = "0x7726827caac94a7f9e1b160f7ea819f172f7b6f9d2a97f992c38edeab82d4110";

async function deployGreeter(deployer: Deployer): Promise<Contract> {
  const artifact = await deployer.loadArtifact("Greeter");
  return await deployer.deploy(artifact, ["Hi"]);
}

describe("Greeter", function () {
  it("Should return the new greeting once it's changed", async function () {
    const provider = Provider.getDefaultProvider();

    const wallet = new Wallet(RICH_WALLET_PK, provider);
    const deployer = new Deployer(hre, wallet);

    const greeter = await deployGreeter(deployer);

    expect(await greeter.greet()).to.eq("Hi");

    const setGreetingTx = await greeter.setGreeting("Hola, mundo!");
    // wait until the transaction is mined
    await setGreetingTx.wait();

    expect(await greeter.greet()).to.equal("Hola, mundo!");
  });
});
```

此脚本部署在 [入门指南](./getting-started.md#write-and-deploy-a-contract) 中创建的 `Greeter` 合约，并测试它在调用 `greet`() 时返回正确的消息方法，并且可以使用 `setGreeting`()方法更新消息。

您现在可以使用以下命令运行测试文件：

```
yarn test
```

**恭喜！ 您已经使用 zkSync 在本地运行了你的第一个测试 🎉**

## 完整的示例

完整的示例和测试可以在[这里](https://github.com/matter-labs/tutorial-examples/tree/main/local-setup-testing)找到 。
