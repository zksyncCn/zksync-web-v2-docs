# 常见问题

这里是关于 zkSync 2.0 的一些常见问题。

## 什么是 zkSync 2.0？

zkSync 2.0 是一个零知识 (ZK) 汇总，与以太坊的通用 EVM 兼容。 zkSync 2.0 的主要好处是，开发人员可以毫不费力地将 EVM dApp 的移植到 zkSync 2.0，并在继承以太坊的安全性和去中心化的同时，实现显著降低的 gas 费用和每秒更多的交易。

## 为什么是 zkSync 2.0？

zkSync 2.0 是第 2 层技术的巨大飞跃。这是一项期待已久的改进，为以太坊开发人员提供了许多以前从未享受过的好处。

- **EVM 兼容** - zkSync 是一个 EVM 兼容的零知识汇总，支持通用的 EVM 智能合约。这意味着如果你已有 EVM 智能合约，那么将你的 dApp 移植到 zkSync 2.0 会非常容易。

- **Ethos Compatible** - 我们与去中心化和开源的精神非常一致。我们所有的代码都将努力完全开源，zkSync 将执行一个完全去中心化排序器和证明生成的路线图，以及我们将执行一个组织减法管理的路线图——也就是说，我们将去中心化我们的组织。

- **确定性** - 与一些试图扩展以太坊的方法不同，在某些情况下，它们提供的安全比 L1 弱（例如侧链、乐观汇总等），zkSync 使用零知识证明，提供 *确定性* 的安全性。

- **面向未来** - 现在采用 zkSync 2.0 的生态系统合作伙伴将享受所有未来的改进，而无需更改他们的代码，特别是来自：
  
  1.证明技术（硬件加速）。
  
  2.编译器（集成支持 LLVM 的现代编程语言）。
  
  3.zkSync 3.0（超链和超链桥）的所有创新。

- ## 什么是zkEVM？

zkEVM 是一个支持零知识证明计算的 EVM 兼容的虚拟机。与普通虚拟机不同，zkEVM 可以证明程序执行的正确性，包括操作中使用的输入和输出的有效性。

其架构由以下组成：

- zkVM，一种图灵完备的类 RISC 虚拟机，针对 ZKP 电路中的证明进行了优化。 它有几种不同的实现：
  
  - 执行器：在 CPU 上快速本地执行。
  - 见证生成器：生成 ZKP 见证的本地执行器。
  - 证明者：实际的 ZKP 电路实现。

- 基于 LLVM 的编译器：
  
  - Solidity 前端（更准确地说：Yul 前端）。
  - Vyper前端。
  - zkVM 后端。
  - 专用电路（heavily relying on PLONK’s custom gates and lookup tables）作为计算密集型操作的“预编译”，例如：
    - 非代数哈希（Keccak、SHA256、Blake2）。
    - 存储访问（默克尔路径）。
    - 椭圆曲线配对。

- 递归聚合电路（结合上述部分的证明）。

### zkEVM 与 EVM

除了操作码和 gas 计量差异之外，zkVM 严格继承了 EVM 编程模型及其不变量，包括 ABI 调用约定。 需要强调的一件重要事情是 zkVM 支持回滚和可证明的可恢复交易。 它保证了相互保护：用户不能通过可逆交易的攻击使网络停滞，逃生舱口（优先队列）保护用户将任何交易纳入区块的能力。

因此，开发人员可以完全依赖 L1 提供的抗审查能力，而无需引入任何与逃生机制相关的更改。 这意味着 zkSync 上 zkRollup 账户中的资产将具有与 L1 上完全相同的安全保证。

### EVM 改进

在保持最大兼容性的同时，zkEVM 对 EVM 进行了重大改进，从而提高了采用率并使我们的生态系统合作伙伴受益。

- **我们的编译器基于 LLVM**。 基于 LLVM 的编译器（低级虚拟机）已成为 Mac OS X、iOS、FreeBSD 和 Android 系统的默认编译器，并且是使用最广泛的编译器之一，因为它们：
  
  - 能够提高原始 EVM 字节码的效率，使用 LLVM 让我们可以利用这个成熟生态系统中可用的许多优化和工具。
  - 为我们添加对用其他编程语言编写的代码库与 LLVM 前端集成的支持铺平道路。 通过这样做，开发人员可以构建 dApp 并以当前不可能的方式使用区块链。

- **我们的 zkEVM 中包含帐户抽象**。 这是以太坊开发社区中期待已久的功能，它以多种方式提高开发人员的采用率和用户体验：
  
  - 对智能合约钱包（如 Argent）的本地支持，这对于引导主流用户至关重要。
  - 更好的多重签名用户体验。
  - 可以使用 [paymasters](../developer-guides/aa.html#paymasters) 以任何代币支付交易费用。
  - 协议现在可以通过智能合约为用户补贴 gas，甚至可以实现无 gas 交易。
  - 交易批次（多调用）可以一键确认（今天以太坊上的大用户体验问题）。
  - 详细了解 [zkSync 2.0 中的帐户抽象支持](../developer-guides/aa.html)。

## EVM 兼容性

关于 EVM 兼容与 EVM 等效的影响，社区中存在很多混淆。 首先，让我们定义两者的含义。

- **EVM 等效**意味着协议支持以太坊 EVM 的每个操作码，直至字节码。 因此，任何 EVM 智能合约都可以 100% 保证开箱即用。
- **EVM 兼容** 表示支持以太坊 EVM 的一定比例的操作码； 因此，一定比例的智能合约开箱即用。

zkSync 被优化为 EVM *兼容* 而不是 EVM *等效*，主要有以下三个原因：

1. 为 EVM 等效创建通用电路直到字节码，将非常昂贵且耗时。
2. 基于我们在 zkSync 1.0 中学到的知识，我们能够设计一个针对 ZK 中的性能和可证明性进行优化的系统。
3. 我们选择不支持的操作码被以太坊本身弃用，或者很少使用。 在项目需要的情况下，与 zkSync 一起工作的修改是最小的，并且不会产生对新安全审计的需求。

几乎每一个为 EVM 编写的智能合约都将得到 zkSync 2.0 的支持，并将保存所有关键的安全不变量，因此在大多数情况下不需要额外的安全重新审计。一个值得注意的例外是使用以下 EVM 操作码的合约：

- `SELFDESTRUCT` -（它被认为是有害的，有人呼吁在 L1 上停用它）。
- `EXTCODECOPY` -（如果需要可以实现，但我们暂时跳过它，因为 zkEVM 操作码无论如何都与 EVM 操作码不同）。
- `CALLCODE` -（在以太坊上已弃用，取而代之的是 `DELEGATECALL`）。
- `CODECOPY` -（它不返回 0，但会产生编译错误）。

还有一些其他区别，例如，Gas 计量会有所不同（其他 L2 也是如此）。一些 EVM 的加密预编译（特别是配对和 RSA）在第一个版本中不可用，但将在发布后很快实施，配对是优先考虑的，以允许超链和 Aztec/Dark Forest 等协议无需修改也可以部署。

### 安全审查

zkSync 2.0 的数据可用性层是以太坊。 所有基于 zkSync 2.0 构建的生态系统合作伙伴都将继承以太坊的全部安全优势。

这显然对我们来说是一个至关重要的话题，我们目前正在对 zkSync 2.0 的安全性进行一次重大审查（包括外部审计、安全竞赛以及彻底检查和扩展我们的漏洞赏金计划）。

我们很快就会在这里增加更多细节。

### 触发安全审查

虽然有一些我们不支持的很少使用的操作码，但包括我们的生态系统合作伙伴，都没有发现移植到 zkSync 生态系统发生破坏性变化的任何实例， 合作伙伴没有任何更改导致需要进行安全审计。

## 什么是账户抽象？

账户抽象允许我们使 授权*变得可编程*，从而实现更多样化的钱包和协议设计，用例包括：

- 实施智能合约钱包，改善私钥存储和恢复的用户体验（例如[社会恢复](https://vitalik.ca/general/2021/01/11/recovery.html)，多重签名）。
- 能够以 ETH 以外的代币支付Gsa。
- 帐户更改公钥和私钥的能力。
- 增加了非加密修改，用户可以要求交易有到期时间、请求确认无次序等等。
- 来自当前 ECDSA 的签名验证系统的多样性，包括后量子安全签名算法（例如 Lamport、Winternitz）。

换言之，账户抽象带来了整体用户体验的重大提升，也为开发者拓展了应用设计空间。 在 Argent 的[这篇博文](https://www.argent.xyz/blog/wtf-is-account-abstraction/) 中了解更多信息。

在 zkSync 2.0 中，账户抽象是本地实现的，这意味着账户可以发起交易，如 EOA，但也可以在其中实现任意逻辑，如智能合约。

如果您想更好地了解 zkSync 上的帐户抽象是如何工作的，您可以阅读[文档的这一部分](../developer-guides/aa.md)，或在[此处](../tutorials/custom-aa-tutorial.md).查看我们的教程。

## zkSync 2.0 vs 其他对比替代方案

### **zkSync 2.0 对比 Optimistic Rollups**

像 Arbitrum 和 Optimism 这样的Optimistic rollups利用乐观方法来保护他们的网络。 在开发时，它们代表了对其他可用选项的重要增量改进。 然而，一个广泛持有的观点（[包括 Vitalik Buterin ](https://coinculture.com/au/people/vitalik-buterin-zk-rollups-to-outperform-optimistic-rollups/)）表达了 Optimistic rollups 更趋向于一种 临时解决方案，从长远来看，唯一永久且真正可扩容的解决方案将是基于零知识证明的区块链。

Optimistic rollups存在以下关键问题：

- **Optimistic rollups 通过博弈论得到保障**。该方法假设所有交易都是有效的，然后利用事后博弈论机制向参与者支付费用，用来发现欺诈或其他无效（例如由于漏洞）交易。 博弈论从来都不是完美的，就像与稳定币和其他系统决裂的博弈论一样，我们只是认为它不能长期依赖并以真正的规模提供生态系统所需的安全性。 *另一方面，zkSync 2.0 依靠数学而不是博弈论来提供绝对确定的证据，证明每笔交易都是有效的而不是欺诈性的。*

- **Optimistic方案需要 7 天结算时间**。 结算时间正成为生态系统合作伙伴越来越重要的特征。 随着生态系统合作伙伴需求的成熟，对即时结算的需求将会增加。 使用Optimistic方案，这个解决问题不会消失。 他们需要 7 天的结算时间，因为Optimistic需要 7 天的事后博弈论结束其挑战窗口确定交易。 解决这个问题的唯一方法是引入提供一些流动性的第三方——但这种信任会增加流动性提供者的潜在安全风险。 *当 zkSync 2.0 最初在主网上启动时，它将提供数小时内的结算，但我们的目标是在数月的工作后几分钟内完成结算——随着我们将结算时间缩短到接近零——合作伙伴无需更改任何代码*。
- **Optimistic rollups 无法扩展到现在的水平**。当 optimistic 方法首次出现时，它们变得流行，因为它们扩展了以太坊——（例如，它们能够处理 10 倍的以太坊交易 ，但不会降低安全性 和权力下放）。 问题是，虽然他们现在可以将以太坊扩展 10 倍，但他们没有任何机制可以在不降低安全性和去中心化的情况下超过 10 倍。 *相比之下，zkSync 2.0 基于零知识证明，它具有Optimistic rollups所没有的重要特征——它们可以实现更大的扩容。*

### zkSync 2.0 对比其他 zkRollups

虽然所有零知识汇总都共享加密证明的底层技术，但存在许多重要差异。

**zkSync 与 Starkware。** 当您将 zkSync 与 Starkware 进行比较时，您看到的主要是两种不同的策略，其中 zkSync 是提升以太坊生态系统、技术和精神兼容的优化，而 Starkware 以理论上的性能优势为代价进行优化兼容性。

- **EVM**。 zkSync 与 EVM 兼容 VS Starknet 与 EVM 不兼容。
- **工具箱**。 zkSync 是工具箱兼容、开箱即用的，而 Starknet 要求人们学习一个名为 Cairo 的自定义语言。
- **生态系统**。 zkSync 的生态系统流畅的与以太坊生态系统的所有工具集一起工作 VS Starkware 正试图用所有新工具引导一个全新的生态系统。
- **权力下放**。 zkSync 在技术层面和组织层面都支持去中心化 VS Starknet 不支持去中心化。
- **开源。** zkSync 是完全开源的 VS Starkware 不是开源的。

## 支持的钱包

目前，我们支持任何基于以太坊的钱包。 默认情况下，zkSync 2.0 门户上提供的选项是 Metamask - 除了自动连接之外，您还可以手动将 zkSync 网络添加到您的 Metamask：

**测试网**

- Network Name: `zkSync alpha testnet`

- RPC URL: `https://zksync2-mainnet.zksync.io`

- Chain ID: `280`

- Currency Symbol: `ETH`
  
  <!-- - Block Explorer URL: `https://testnet.explorer.zksync.io/` -->

**主网**

主网的连接详细信息将很快共享。

## 我如何为测试网申请资金？

要访问测试网资金 ([Faucet](https://portal.zksync.io/faucet))，您可以发布关于我们的推文并获得代币。 确保 Twitter 消息包含您的以太坊地址，我们将向其发送资金，并且我们建议它不是您的主要以太坊账户。 

水龙头不适用于新的 Twitter 帐户和没有头像的帐户。

或者，您可以使用[我们的桥梁](https://portal.zksync.io/bridge) 将 ETH 从 Goerli 桥接到 zkSync alpha 测试网。

## 完成存款交易需要多长时间？

zkSync 2.0 上的交易不应超过 5 分钟。

## 我在哪里可以看到我提交的交易？

我们的 [Block Explorer](https://explorer.zksync.io) 将显示您可能需要的有关交易的所有信息。

## 我可以将我的资金提取回以太坊吗？

是的，这座桥是双向的。 您可以将资金提取回以太坊。 取款交易最多需要 1 小时，具体取决于 zkSync 网络的使用情况。

## Interacting - A step-by-step指南

由于测试网运行在 Goerli 网络上，您需要先获取一些 Goerli ETH。 试试下面的水龙头。

- [https://goerli-faucet.mudit.blog/](https://goerli-faucet.mudit.blog/)
- [https://faucets.chain.link/goerli](https://faucets.chain.link/goerli)
- [https://goerli-faucet.slock.it/](https://goerli-faucet.slock.it/)

**步骤1**

前往 [https://portal.zksync.io/](https://portal.zksync.io/) 并连接您的钱包。 您将被提示添加`zkSync 2.0 testnet Goerli`网络。

您也可以手动将网络添加到您的 metamask中。

- 网络名称：`zkSync alpha testnet`
- RPC URL：`https://zksync2-testnet.zksync.dev`
- Chain ID: `280`

**第二步（没有Goerli ETH的跳过）**

我们首先转到“Bridge”，然后“Deposit”将一些 $ETH 存入 zkSync 2.0。

![image](../../assets/images/faq-1.png)

**第 3 步**

接下来，我们转到“Faucet”，将一些测试网 $ETH、$LINK、$DAI、$WBTC 和 $USDC 放入我们的 zkSync 地址。

![image](../../assets/images/faq-2.png)

领取后在“余额”查看余额。

![image](../../assets/images/faq-3.png)

**第4步**

现在转到“Transfer”。 输入另一个钱包的地址并向其转入一些代币。 如果您没有 ETH，请用 DAI 支付费用。

![image](../../assets/images/faq-4.png)

**第 5 步**

最后我们去“Withdraw”把一些$DAI从zkSync中提取回Goerli。 如果您没有 ETH，请用 DAI 支付费用。

![image](../../assets/images/faq-5.png)

## 什么是测试网 ReGenesis？

有时，zkSync 工作的团队会在测试网上启动更新——升级重置，这将引入升级并将所有状态重置。 这不会发生在主网上，一般也不会在主网上线后的测试网上发生。
