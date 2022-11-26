## 使用 zkEVM 调试器

### 概述

合约执行的缺点是很难确定交易做了什么。交易收据有一个状态码，表示执行是否成功，但无法确定更新了哪些数据或触发了哪些外部合约。这可以通过 zkSync zkEVM 调试器来解决，它会重放您智能合约的执行并捕获有关 EVM 操作的数据，从而允许您检查每条指令。

[调试器页面](https://explorer.zksync.io/tools/debugger) 可以从顶部菜单访问。

![zkEVM!](../../../assets/images/zk-evm.png "zkEVM page")

### 调试步骤

以下是调试或跟踪交易的步骤：

1. 上传 JSON 文件：单击 `Upload JSON file` 按钮，您将看到一个带有保存提示的模式窗口，或只需将文件拖拽到上传对话框中。要了解文件类型的规范，请阅读 [EVM 跟踪规范](https://eips.ethereum.org/EIPS/eip-3155)。
2. 文件上传：出现加载屏表示正在加载跟踪文件。
3. 成功上传后，调试器状态变为活跃状态。
4. 要继续调试，请单击 `Start` 按钮。

这是调试时要注意的一些 **键盘快捷键**。

- `Cmd + K`：打开搜索栏。
- `Arrows Left / Right`：下一个或上一个指令。
- `Arrows Top / Bottom`：该合约中下一个或上一个函数。
- `Shift + Arrows`：下一个或上一个合约。
