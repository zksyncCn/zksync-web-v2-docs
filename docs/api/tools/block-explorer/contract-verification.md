# 合约验证

## 什么是合约验证？

当您在网络上部署智能合约时，您实际上是在部署 Solidity 编译器生成的字节码。

验证合约需要确定链上字节码在编译时是否与给定的源代码匹配。如果是，我们可以声明我们已经确认了字节码源代码的完整性。因此得名——验证。

## 智能合约在 zkSync 中是如何验证的？

在使用 zkSync 的验证过程中，将部署的字节码和智能合约的 Solidity 源代码进行比较。该算法编译源代码，将生成的字节码与部署的字节码进行比较。

<br>
如果双方在各方面均匹配，则合约得到验证。

代码验证需要六个参数：

- 合约地址

- 合约名称

- 源代码，包括所有导入的源

- 用于生成已部署字节码的编译器版本

- 有关编译器优化次数的信息（如果有）

- 构造函数参数
  
  如果这些信息中有任何一条是错误的，则验证过程失败。

- The constructor arguments
  
  If any of these pieces of information is wrong, the process of verification fails.

## 源代码隐私

当您的智能合约部署在 zkSync 并在区块浏览器上进行验证时，用户可以检查已验证的源代码并与之交互。

## 使用 zkSync 区块浏览器验证合约

首先点击顶部标题中的 **Tools** 选项卡，在弹出的下拉菜单中选择 **Smart Contract Verification**，之后，您将看到以下内容：

![Smart Contract Verification page!](../../../assets/images/verify-contract.png "verify contract")

### 输入合约详细信息

要验证合约，请输入以下详细信息：

- 合约地址：所提供的地址必须与合约创建时生成的 `0x` 地址相匹配。

- 合约名称：名称必须与合约中提供的名称相同。

- 优化：检查在编译合约时是否使用了优化。如果在编译期间启用了优化，请选 **Yes** 否则选 **No**。

- Solidity 编译器版本(Solc)：指定了用于编译智能合约的编译器的版本。单击下拉列表以指定使用的编译器版本。

- zkSync 编译器版本(Zksolc)：所使用的 zkSync 编译器版本，默认为 `v1.2.0`。
  
  <br>

![编译器版本](../../../assets/images/compiler-version.png "compiler version")

<br>

- 输入 Solidity Contract 代码：从你的编辑器中复制代码并将其粘贴到文本区域。
  
  > **注意：** 如果 Solidity 代码使用了库或从另一个合约继承依赖项，您可能需要扁平化（flatten）代码。 
  > 
  > 我们推荐使用 [Hardhat flatten](https://medium.com/coinmonks/flattening-smart-contracts-using-hardhat-dffe7dbc7b3f)， [Truffle flattener](https://github.com/NomicFoundation/truffle-flattener) 或 [POA Solidity flattener](https://github.com/poanetwork/solidity-flattener)。

- 构造函数参数：如果合约需要构造函数参数，则应在此处添加，但应采用 [ABI 16 进制编码形式](https://solidity.readthedocs.io/en/develop/abi-spec.html)。

<br>

### 如何获得构造函数参数？

> **注意：** 获取构造函数参数数据的最简单方法是在部署时将其打印到控制台。

例如，假如使用我们的 [教程](../hello-world.md)，您可以找到以下行：

```js
const greeterContract = await deployer.deploy(artifact, [greeting]);
```

如果您将下一行添加为

```js
console.log(greeterContract.interface.encodeDeploy([greeting]);
```

那么您将收到构造函数参数数据。

- 最后，单击 **Verify Smart Contract** 按钮。

<br>
如果一切顺利，您将看到一条成功消息。

<br>

![Smart Contract Verified!](../../../assets/images/contract-verified.png "Contract Verified")
