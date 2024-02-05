# 2024-02-05-Olympus RBS2.0 Audit Review

# Pre

Olympus 介绍：[https://docs.olympusdao.finance/main/overview/](https://docs.olympusdao.finance/main/overview/)

函数说明：[Kernel | Olympus Docs (olympusdao.finance)](https://docs.olympusdao.finance/main/contracts/contract-docs/Kernel)

代码地址：https://github.com/sherlock-audit/2024-01-olympus-on-chain-governance

审计详情：[https://audits.sherlock.xyz/contests/128](https://audits.sherlock.xyz/contests/128)

# What is Olympus

Olympus协议是一种支持OHM（Olympus DAO 平台的主要协议代币，以太坊网络上由treasury支持）的DeFi系统。Olympus利用Protocol Owned Liquidity（POL）、Range Bound Stability（RBS）和Cooler Loans的机制，创建一个稳健、灵活、抗审查和智能的货币。

- Treasury
    
    Treasury是Olympus的关键组成部分，代表协议所拥有和控制的所有资产。
    
- POL
    
    在传统的加密货币交易对中，流动性通常是由流动性提供者（LPs）贡献的，他们将资产锁定在流动性池内以提供交易对的深度，而获得交易费用作为回报。然而，这种模式将流动性视为临时的，因为LPs随时可以撤回他们的资产。
    
    与此相反，POL方法将流动性视为永久性的，该流动性由协议本身所拥有。这意味着协议本身可以收集交易费用，使用这些资金进行自我增长，或者根据其治理机制来再分配这些资金。这也消除了流动性相关的一些风险，比如临时的流动性损失，以及流动性提供者可能撤回资产，导致交易深度下降的风险。
    
    OlympusDAO 是典型的项目之一，它通过"资金池"概念，协议自行持有并控制流动性，而非依赖外部的流动性提供者。
    
- RBS
    
    "Range Bound Stability"是一种金融术语，用来描述一个特定资产的价格在一段时间内在一个特定的价格区间内波动。这种状况下，资产的价格并没有明显的上升或下降的趋势，而是在两个明确定义的水平（一个是阻力水平，另一个是支撑水平）之间波动。
    
    在加密货币市场中，这种策略也被广泛使用。交易员会设定具体的买入和卖出点，当价格达到这些点时进行交易，从而在这个范围内实现利润。
    
- Cooler Loans
    
    Cooler Loans是一种去中心化的贷款机制，允许OHM代币持有人使用其gOHM（governance OHM）代币作为抵押品来借入DAI。这种贷款机制是无许可的、不可变的，由奥林巴斯智能合约管理。有了Cooler Loans，用户将可以使用其gOHM代币作为抵押品可靠地获得流动性。
    
- Cross-Chain Bridge
    
    跨链桥是区块链技术中的一个重要概念，它允许不同的区块链网络之间进行安全、可靠地信息交换和资产转移。这种技术解决了原本在区块链孤立网络中资产不能相互流通的问题，提高了区块链的互操作性。
    

# Contract Overview

Olympus合约包含几个重要组成部分：the core registry (Kernel), treasury, minter, governor, RBS system。

所有合约依赖和授权都会通过Kernel.sol中的操作来管理，包括：

- 安装模块
- 更新模块
- 激活策略
- 关闭策略
- 改变 Executor
    - Executor 是一个可以调用主要内核函数的地址
- 移动内核

模块是面向内部的智能合约，跨协议存储共享状态，并用作协议内策略的依赖项。在Olympus v3 中，有以下几个模块：

- `MINT`：Mint模块，铸造或销毁ERC20
- `TRSRY`：Treasury模块，用于在协议内存放和提取资产，还管理分配给policies的代币债务
- `PRICE`：存储历史价格预言机数据，用于RBS
- `RANGE`：存储RBS的范围信息
- `INSTR`：指令模块，用于存储批处理的内核指令
- `ROLES`：角色模块用于允许策略定义角色、为这些角色分配地址，以及指定的管理这些角色

模块合约定义了协议副作用发生的位置，而策略合约定义了它们发生的方式。

Olympus v3 中包含以下Policies：

- RBS policies：
    - `Operator.sol`：RBS的主要policy，投入市场订单，启动债券市场，并按照RBS规范与treasury进行交换
    - `BondCallback.sol`：被用做债券市场的回调。允许债券市场铸造OHM用来支付
    - `Heart.sol`：合约允许管理员访问来调用RBS管理员功能
    - `PriceConfig.sol`：指定的用户用来调整Price模块的参数
- Cooler Loans policies：
    - `Clearinghouse.sol`：Clearinghouse是一种贷款人所有的合约，管理贷款工作流程，包括履行请求、延长到期日、违约索赔和重新平衡Olympus Treasury 的往来资金
- Governance policies：
    - `Parthenon.sol`：专为违约创建的管理合约，以Proof-of-Stake为模型
    - `VohmVault.sol`：将投票权分配给投票柜的政策
- General protocol and management policies:
    - `TreasuryCustodian.sol`：允许指定角色在异常情况下修改状态的实用程序策略。用于授予和取消授权以及管理债务
    - `Distributor.sol`：用于处理OHM持有者的增益发行的合约
    - `RolesAdmin.sol`：管理ROLES模块
    - `Emergency.sol`：在特殊情况下关闭和重启核心系统的紧急合约

![Untitled](Olympus%20RBS2%200%2072ad54c1c87a426a8b2c739533abe86b/Untitled.png)

# Security Issues Review

## 1. 🔴 BunniSupply.sol

### 1.1 getProtocolOwnedLiquidityReserves：返回不正确的储备数量

`getProtocolOwnedLiquidityReserves`函数的用途是获取协议的流动性储备总额，但函数会计算所有的BunniToken，若BunniToken可以被外部用户任意修改，则会导致总额不符合原意。

![Untitled](Olympus%20RBS2%200%2072ad54c1c87a426a8b2c739533abe86b/Untitled%201.png)

函数中获取的是所有Bunni Token，而不是协议自己的。且在BunniHub.sol中有`deposit`函数：

![Untitled](Olympus%20RBS2%200%2072ad54c1c87a426a8b2c739533abe86b/Untitled%202.png)

此函数允许任意用户给一个代币添加流动性。因此返回的储备并是可以被其他用户修改的。

### **1.2 getProtocolOwnedLiquidityOhm**：其他用户的存储导致ProtocolOwnedLiquidityOhm计算错误

与1相似。

`getProtocolOwnedLiquidityOhm`函数的用途是获取以OHM为标的的协议拥有的流动性储备。其在计算过程中会计算所有的BunniToken。因此如果BunniToken可以被外部用户任意修改，会导致流动性储备失真

## 2. 🔴 OlympusPrice.v2.sol::storePrice

移动平均价格应该只由预言机提供的价格来计算，但此处不仅是通过预言机提供的价格，还有移动平均价格。

移动平均价格被用于递归计算移动平均价格，会使移动平均价格变得更加平滑，因为它是在一个特定时间窗口内的所有价格的平均值。这种方式有助于削弱短期波动和随机噪声的影响，而强调长期趋势。当我们递归使用移动平均来计算新的移动平均价格时，平滑效果被加强。因为我们实际上是在已经平滑过的数据上再进行一次平滑，这使结果数据变得更加稳定，减少了短期波动，但也可能过度抑制了实际的价格波动。

### 2.1 移动平均价格被用于递归计算移动平均价格

![Untitled](Olympus%20RBS2%200%2072ad54c1c87a426a8b2c739533abe86b/Untitled%203.png)

L319 调用`_getCurrentPrice`来获取当前价格。

L330-L331确定了当`asset.storeMovingAverage`存在时，asset.cumulativeObs会由`asset.cumulativeObs`、`price`、`oldestPrice`三者决定。

在`_getCurrentPrice`函数中，L160 `if(asset.useMovingAverage) prices[numFeeds] **=** asset.cumulativeObs **/** asset.numObservations;`，这里说明存入prices数组中当前存入的值是`cumulativeObs`决定的，当函数返回的`price`就是由prices数组汇聚成的一个价格。因此出现了price参与移动平均价格的递归计算。

### 2.2 当 asset.useMovingAverage 为 true 时，_getCurrentPrice 在某些情况下可能会获得过时的价格

L319获取的价格不是来自外部预言机的新鲜价格，而是旧的移动平均价格，只要时间足够长以重写整个`asset.obs`数组，数组中的所有价格都会变得越来越接近和陈旧。如果这期间实际价格波动较大，就会存在较大的套利机会。

### 2.3 当调用 storePrice 后 Feed 关闭时，价格模块会报告错误的 MA/使用 MA 的资产当前价格

## 3. 🟡 SimplePriceFeedStrategy.sol::getMedianPriceIfDeviation：价格会被计算错误

在`getMedianPriceIfDeviation`函数中，L237-L246中表明，当不为0的价格数量小于3，会返回第一个价格。这里就是问题的根源。

![Untitled](Olympus%20RBS2%200%2072ad54c1c87a426a8b2c739533abe86b/Untitled%204.png)

当不为0的价格数量为2时，若返回第一个价格而不是平均数，则会造成中位数价格严重失真。

## 4. 🟡 有意回退价格源可能可以控制价格计算

攻击者可以利用一些方法实现回退价格源：

1. UniswapV3Price.sol#L214：如果出现重入，UniswapV3 价格源会回退
    
    ![Untitled](Olympus%20RBS2%200%2072ad54c1c87a426a8b2c739533abe86b/Untitled%205.png)
    
2. BalancerPoolTokenPrice.sol#L388，BalancerPoolTokenPrice.sol#L487，BalancerPoolTokenPrice.sol#L599，BalancerPoolTokenPrice.sol#L748：如果出现重入，余额池价格源会回退
    
    ![Untitled](Olympus%20RBS2%200%2072ad54c1c87a426a8b2c739533abe86b/Untitled%206.png)
    
3. BunniToken.sol#L155-L160 → L242-L266：当*deviationBps*错误时会回退：
    
    ![Untitled](Olympus%20RBS2%200%2072ad54c1c87a426a8b2c739533abe86b/Untitled%207.png)
    

对于 ERC20 代币价格，通常将 3 个以上的价格源与 Chainlink 价格源结合使用，也可以选择与 averageMovingPrice 结合使用。还有另外两个关键点：

1. OlympusPrice.v2.sol#L160：当使用平均移动价格时，平均移动价格会加入价格数组
2. 在价格计算策略中，当只有 2 个有效价格时，使用第一个非零价格

因此，当Chainlink价格源被操纵时，攻击者可以故意禁用上述三个价格源，来实现价格操纵。
当Chainlink价格源没使用时，攻击者可以操纵上述3个价格源中的一个并禁用其他源来实现价格操纵。

## 5. Balancer 稳定池中的代币的价格可能因为放大参数的更新导致错误

BalancerPoolTokenPrice.sol#L811 中通过调用`getLastInvariant`获得不变量的最后值（代表池中资产的总权重）以及放大系数（ampFactor），用于调整价格计算中的滑动率。这是Balancer稳定池特有的一个特性，用以维护池子内代币价格的稳定性。

![Untitled](Olympus%20RBS2%200%2072ad54c1c87a426a8b2c739533abe86b/Untitled%208.png)

当调用此函数时，获得了一个更新后的放大参数，那么最后一次不变量计算所使用的放大参数可能与当前放大参数不同。这可能导致计算获得的代币价格不准确，因为价格是依赖于放大参数的。如果放大参数改变了，理应重新计算价格以反映最新的市场情况。如果在计算某些重要价值，如代币价格、流动性提供者收益、交易滑点等时使用了错误或过时的放大参数，就可能导致代币价格计算的不准确，从而影响到交易决策、风险评估等方面。

## 6. 🟡 BalancerPoolTokenPrice.sol::getStablePoolTokenPrice：计算错误

![Untitled](Olympus%20RBS2%200%2072ad54c1c87a426a8b2c739533abe86b/Untitled%209.png)

代码旨在识别 Balancer 流动性池中所有基础代币的最低价格。然后，该最低价格用于池的估值。但是在实际测试过程中，wstETH/aETHc 矿池最低价格与Balancer 流动性池中所有基础代币的最低价格不一致。

## 7. 🔴 BunniPrice.sol::getBunniTokenPrice：返回错误

此函数目的是返回BunniToken的单价，但是返回的是总价，还需要除以供应量
