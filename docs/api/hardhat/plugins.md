# 插件

## `hardhat-zksync-solc`

这个插件用于在将Solidity智能合约部署到 zkSync 2.0 之前提供一个方便的接口来编译它们。

### Npm

[@matterlabs/hardhat-zksync-solc](https://www.npmjs.com/package/@matterlabs/hardhat-zksync-solc)

使用以下命令将此插件的最新版本添加到您的项目中：

```
# Yarn
yarn add -D @matterlabs/hardhat-zksync-solc

# Npm
npm i -D @matterlabs/hardhat-zksync-solc
```

### Exports

这个插件通常不会直接在代码中使用。

### 配置

此插件在项目的 `hardhat.config.ts`文件中配置。 这是一个例子：

```typescript
zksolc: {
  version: "1.2.0",
  compilerSource: "binary",  // binary or docker
  settings: {
    compilerPath: "zksolc",  // ignored for compilerSource: "docker"
    experimental: {
      dockerImage: "matterlabs/zksolc", // required for compilerSource: "docker"
      tag: "latest"   // required for compilerSource: "docker"
    },
    libraries{} // optional. References to non-inlinable libraries
  }
}
networks: {
  hardhat: {
    zksync: true  // enables zksync in hardhat local network
  }
}
```

- `version` 是带有 `zksolc` 编译器版本的字段。目前没有使用。
- `compilerSource` 表示编译器源，可以是 `docker` 或 `binary`（推荐）。如果尚未安装编译器二进制文件，插件将自动下载它。如果使用 `docker`，您需要在后台运行 Docker desktop，并在实验部分提供 `dockerImage` 和 `tag`。
- `compilerPath` 是一个包含 `zksolc` 二进制文件路径的字段。默认情况下，使用 `$PATH` 中的二进制文件。如果 `compilerSource` 是 `docker`，这个字段会被忽略。
- `dockerImage` 和 `tag` 组成编译器 docker 镜像的名称。如果  `compilerSource`  是 `binary`，这些字段将被忽略。
- `libraries` 如果您的合约使用不可内联的库作为依赖项，则必须在此处定义它们。详细了解[在此处编译库](./compiling-libraries.md)
- `zksync` 网络选项指示是否在特定网络上启用了 zksolc。默认情况下为“假”。对于您可以仅为特定网络启用“zksync”的多链项目很有用。

### 命令

`hardhat compile` -- 编译 `contracts` 目录中的所有智能合约，并创建包含所有编译工件的 `artifacts-zk` 文件夹，包括合约的工厂依赖项，可用于合约部署。

要了解工厂依赖项是什么，请在 [Web3 API](../api.md) 文档中阅读更多相关信息。

## `hardhat-zksync-vyper`

该插件用于在将 Vyper 智能合约部署到 zkSync 2.0 之前为编译 Vyper  智能合约提供一个方便的接口。

### Npm

[@matterlabs/hardhat-zksync-vyper](https://www.npmjs.com/package/@matterlabs/hardhat-zksync-vyper)

此插件与 [@nomiclabs/hardhat-vyper](https://www.npmjs.com/package/@nomiclabs/hardhat-vyper) 结合使用。
要使用它，您必须在 hardhat.config.ts 文件中安装并导入这两个插件：

```javascript
import "@nomiclabs/hardhat-vyper";
import "@matterlabs/hardhat-zksync-vyper";
```

使用以下命令将此插件的最新版本添加到您的项目中：

```
# Yarn
yarn add -D @matterlabs/hardhat-zksync-vyper

# Npm
npm i -D @matterlabs/hardhat-zksync-vyper
```

### Exports

这个插件通常不会直接在代码中使用。

### 配置

```typescript
zkvyper: {
  version: "0.1.0",
  compilerSource: "binary",  // binary or docker
  settings: {
    compilerPath: "zkvyper",  // ignored for compilerSource: "docker"
    experimental: {
      dockerImage: "matterlabs/zkvyper",  // required for compilerSource: "docker"
      tag: "latest"  // required for compilerSource: "docker"
    },
    libraries{} // optional. References to non-inlinable libraries

  }
}
networks: {
  hardhat: {
    zksync: true  // enables zksync in hardhat local network
  }
}
```

- `version` 是带有 `zkvyper` 编译器版本的字段。目前没有使用。
- `compilerSource` 表示编译器源，可以是 `docker` 或 `binary`（推荐）。如果尚未安装编译器二进制文件，插件将自动下载它。如果使用 `docker`，您需要在后台运行 Docker desktop，并在实验部分提供 `dockerImage` 和 `tag`。
- `compilerPath` 是一个包含 `zkvyper` 二进制文件路径的字段。默认情况下，使用 `$PATH` 中的二进制文件。如果 `compilerSource` 是 `docker`，这个字段会被忽略。
- `dockerImage` 和 `tag` 组成编译器 docker 镜像的名称。如果 compilerSource 是 binary，这些字段将被忽略。
- `libraries` 如果您的合约使用不可内联的库作为依赖项，则必须在此处定义它们。详细了解[在此处编译库](./compiling-libraries.md)
- `zksync` 网络选项指示是否在特定网络上启用了 zkvyper。默认情况下为“假”。对于您可以仅为特定网络启用“zksync”的多链项目很有用。

### Commands

`hardhat compile`——编译 `contracts` 目录中的所有智能合约，并创建 `artifacts-zk` 文件夹，其中包含所有编译工件，包括合约的工厂依赖项，可用于合约部署。

要了解工厂依赖项是什么，请在 [Web3 API](../api.md) 文档中阅读更多相关信息。

## `hardhat-zksync-deploy`

该插件提供实用程序，用于在 zkSync 上部署智能合约，并使用 `@matterlabs/hardhat-zksync-solc` 或 `@matterlabs/hardhat-zksync-vyper` 插件构建的工件。

::: 注意

Contracts must be compiled using the official `@matterlabs/hardhat-zksync-solc` or `@matterlabs/hardhat-zksync-vyper` plugins. Contracts compiled with other compilers will fail to deploy to zkSync using this plugin.

:::

### Npm

[@matterlabs/hardhat-zksync-deploy](https://www.npmjs.com/package/@matterlabs/hardhat-zksync-deploy)

Add the latest version of this plugin to your project with the following command:

```
# Yarn

yarn add -D @matterlabs/hardhat-zksync-deploy

# Npm

npm i -D @matterlabs/hardhat-zksync-deploy
```

### Exports

#### `Deployer`

The main export of this plugin is the `Deployer` class. It is used to wrap a `zksync-web3` Wallet instance and provides a convenient interface to deploy smart contracts. Its main methods are:

```typescript
class Deployer {

  /**
   * @param hre Hardhat runtime environment. This object is provided to scripts by hardhat itself.
   * @param zkWallet The wallet which will be used to deploy the contracts.
   */
  constructor(hre: HardhatRuntimeEnvironment, zkWallet: zk.Wallet)

  /**
   * Created a `Deployer` object on ethers.Wallet object.
   *
   * @param hre Hardhat runtime environment. This object is provided to scripts by hardhat itself.
   * @param ethWallet The wallet which will be used to deploy the contracts.
   */
  static fromEthWallet(hre: HardhatRuntimeEnvironment, ethWallet: ethers.Wallet)

  /**
   * Loads an artifact and verifies that it was compiled by `zksolc\.
   *
   * @param contractNameOrFullyQualifiedName The name of the contract.
   *   It can be a bare contract name (e.g. "Token") if it's
   *   unique in your project, or a fully qualified contract name
   *   (e.g. "contract/token.sol:Token") otherwise.
   *
   * @throws Throws an error if a non-unique contract name is used,
   *   indicating which fully qualified names can be used instead.
   *
   * @throws Throws an error if an artifact was not compiled by `zksolc`.
   */
  public async loadArtifact(
    contractNameOrFullyQualifiedName: string
  ): Promise<ZkSyncArtifact>

  /**
   * Estimates the price of calling a deploy transaction in a certain fee token.
   *
   * @param artifact The previously loaded artifact object.
   * @param constructorArguments List of arguments to be passed to the contract constructor.
   *
   * @returns Calculated fee in ETH wei.
   */
  public async estimateDeployFee(
    artifact: ZkSyncArtifact,
    constructorArguments: any[]
  ): Promise<ethers.BigNumber>

  /**
    * Sends a deploy transaction to the zkSync network.
    * For now it uses defaults values for the transaction parameters:
    *
    * @param artifact The previously loaded artifact object.
    * @param constructorArguments List of arguments to be passed to the contract constructor.
    * @param overrides Optional object with additional deploy transaction parameters.
    * @param additionalFactoryDeps Additional contract bytecodes to be added to the factory dependencies list.
    * The fee amount is requested automatically from the zkSync server.
    *
    * @returns A contract object.
    */
  public async deploy(
    artifact: ZkSyncArtifact,
    constructorArguments: any[],
    overrides?: Overrides,
    additionalFactoryDeps?: ethers.BytesLike[],
  ): Promise<zk.Contract>

  /**
   * Extracts factory dependencies from the artifact.
   *
   * @param artifact Artifact to extract dependencies from
   *
   * @returns Factory dependencies in the format expected by SDK.
   */
  async extractFactoryDeps(artifact: ZkSyncArtifact): Promise<string[]>
```

To see an example script of how to use a `Deployer` to deploy a contract, check out [deployment section of the quickstart](./getting-started.md#write-and-deploy-a-contract).

### Configuration

Add the following properies in the `hardhat.config.ts` file:

```json
zkSyncDeploy: {
  zkSyncNetwork: "https://zksync2-testnet.zksync.dev",
  ethNetwork: "goerli"  // Can also be the RPC URL of the network
}
```

- `zkSyncNetwork` is a field with the URL of the zkSync node.
- `ethNetwork` is a field with the URL of the Ethereum node. You can also provide network name (e.g. `goerli`) as the value of this field. In this case, the default `ethers` provider for the network will be used.

### Commands

`hardhat deploy-zksync` -- runs through all the scripts in the `deploy` folder. To run a specific script, add the `--script` argument, e.g. `hardhat deploy-zksync --script 001_deploy.ts` will run script `./deploy/001_deploy.ts`.

::: tip

Note that deployment scripts must be placed in the `deploy` folder!

:::
