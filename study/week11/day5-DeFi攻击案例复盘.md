# Week 11 Day 5: DeFi 攻击案例复盘

**预计学习时间**: 6-8小时  
**难度**: ⭐⭐⭐⭐⭐

## 学习目标

通过本节课程,你将能够:
- 复盘历史上著名的 DeFi 攻击事件
- 理解这些攻击背后的具体漏洞原理
- 学习如何从他人的失败中吸取教训，避免重蹈覆辙
- 掌握分析攻击交易（Post-mortem）的方法

## 课程内容

### 1. 远古时代的教训: The DAO 与 Parity

#### 1.1 The DAO (2016) - Reentrancy

**事件**: 攻击者利用重入漏洞盗走 360 万 ETH，导致以太坊硬分叉（ETH vs ETC）。

**漏洞代码片段**:
```solidity
function splitDAO(...) {
    // ...
    // 1. Interaction: 提款
    p.recipient.call.value(p.amount)(transactionData);
    // 2. Effects: 更新余额
    balances[msg.sender] -= p.amount;
}
```
**教训**: 必须遵循 Checks-Effects-Interactions 模式。

#### 1.2 Parity Multisig (2017) - Library Initialization

**事件**: 攻击者初始化了 Parity 多签钱包的库合约（Library），变成了 Owner，然后自毁了库合约，导致所有依赖该库的多签钱包资金被永久锁定（约 5 亿美金）。

**漏洞原理**:
库合约通常是无状态的，但 Parity 的库合约没有防范 `initWallet` 被外部调用。攻击者调用了库合约的 `initWallet`，成为了 Owner，然后调用 `kill()`。

**教训**: 库合约和逻辑合约（Implementation）也需要初始化保护，或确保无法被 selfdestruct。现代 OpenZeppelin 合约使用 `_disableInitializers()` 在构造函数中锁定逻辑合约。

### 2. ERC777 与兼容性: Uniswap / Lendf.Me

#### 2.1 ERC777 Hook Reentrancy (2020)

**事件**: Lendf.Me (dForce) 被攻击，损失 2500 万美元。

**漏洞原理**:
Lendf.Me 支持 imBTC（一种 ERC777 代币）。ERC777 标准在转账时会回调接收者的 `tokensReceived` 钩子函数。
攻击者在 `supply()` 函数中存入 imBTC，Lendf.Me 记录了存款。但在 Lendf.Me 更新余额之前，ERC777 的回调触发了。攻击者在回调中再次调用 `withdraw()`。此时 Lendf.Me 的状态尚未更新，认为攻击者还有余额，允许提款。

**教训**:
-   不要盲目兼容所有 ERC20 变体（尤其是带回调的 Token）。
-   使用 `nonReentrant` 修饰符。

### 3. 预言机操纵: Harvest Finance / bZx

#### 3.1 闪电贷价格操纵 (2020)

**事件**: Harvest Finance 遭受闪电贷攻击，损失 2400 万美元。

**漏洞原理**:
Harvest 的策略是将资金存入 Curve 的流动性池。在计算 `sharePrice` 时，直接使用了 Curve 池当前的现货价格。
1.  攻击者通过闪电贷借入巨额 USDC。
2.  在 Curve 上大量兑换 USDT -> USDC，导致 USDC 价格暴跌。
3.  在 Harvest 中以极低的 USDC 价格存入（获得更多 Share）。
4.  在 Curve 上反向操作，拉回价格。
5.  在 Harvest 中提款（Share 价格恢复），赚取差价。

**教训**:
-   **永远不要使用 DEX 的现货价格作为预言机**。
-   使用 Chainlink 等去中心化预言机，或 TWAP（时间加权平均价格）。

### 4. 跨链桥危机: Poly Network / Wormhole

#### 4.1 Poly Network (2021) - 权限管理失误

**事件**: Poly Network 被盗 6.1 亿美元（后被归还）。

**漏洞原理**:
攻击者通过精心的构造数据，利用合约逻辑漏洞，成功调用了中继链上的 `putCurEpochConPubKeyBytes` 函数，将共识节点的公钥替换为自己的公钥。这让他可以伪造任意跨链消息。

**教训**: 跨链协议的鉴权逻辑极其复杂，任何微小的逻辑漏洞都可能导致全线崩溃。

#### 4.2 Wormhole (2022) - 签名验证绕过

**事件**: Solana 跨链桥 Wormhole 被盗 3.2 亿美元。

**漏洞原理**:
Solana 上的验证合约在检查 `sysvar` 账户（用于签名验证的系统账户）时，使用了一个被废弃的指令 `load_instruction_at`。攻击者伪造了一个假的 `sysvar` 账户，合约未能正确验证该账户的真实性，从而接受了攻击者伪造的签名。

**教训**: 深入理解底层链（如 Solana）的特性和弃用 API。

### 5. 如何分析攻击交易 (Post-mortem Analysis)

#### 5.1 工具准备

-   **Etherscan**: 查看交易 Logs, State Changes。
-   **Tenderly / Phalcon**: 逐行 Debug 交易执行堆栈。
-   **Ethtx.info**: 可视化交易流。

#### 5.2 分析步骤

1.  **找到攻击交易 Hash**。
2.  **查看资金流向**: 谁转走了多少钱？
3.  **查看调用堆栈**:
    -   入口函数是什么？
    -   调用了哪些外部合约？
    -   在哪一步发生了异常的资金转移？
4.  **定位漏洞代码**: 结合开源代码，找到导致异常逻辑的具体行数。
5.  **复现攻击**: 使用 Hardhat Forking 编写测试脚本，尝试在本地复现攻击。

## 实践练习

### 练习1: 复现 ERC777 重入攻击

1.  编写一个简易的 `Bank` 合约，支持 `deposit` 和 `withdraw`。
2.  编写一个 `FakeERC777` 代币，实现 `tokensToSend` 钩子。
3.  在钩子中回调 `Bank.withdraw`。
4.  验证是否能实现 "存 1 块，取 2 块"。

### 练习2: 分析 Uniswap V2 价格预言机操纵

1.  Fork 主网状态。
2.  编写脚本，在 Uniswap V2 的 ETH/USDC 池中进行巨额 Swap。
3.  观察 Swap 前后 `getReserves` 计算出的价格变化幅度。
4.  计算如果一个借贷协议使用该瞬间价格，攻击者能获利多少。

### 练习3: 使用 Tenderly 分析真实攻击

1.  访问 [Tenderly Public Dashboard](https://dashboard.tenderly.co/explorer)。
2.  搜索 "Cream Finance" 或 "Harvest Finance" 的攻击交易。
3.  浏览 Trace，尝试理解攻击者的每一步操作。

## 学习检查清单

- [ ] 理解 Reentrancy 的多种变体 (ETH, ERC777)
- [ ] 明白为什么 DEX 现货价格不能做预言机
- [ ] 了解初始化函数保护的重要性
- [ ] 掌握使用区块浏览器和 Debugger 分析交易的基本方法
- [ ] 能够解释“闪电贷”在攻击中扮演的角色（它是工具，不是漏洞本身）

## 扩展阅读

-   [Rekt.news](https://rekt.news/) (DeFi 攻击新闻与深度分析)
-   [SlowMist Medium](https://slowmist.medium.com/)
-   [PeckShield Alert](https://twitter.com/PeckShieldAlert)
-   [Immunefi Bug Bounties](https://immunefi.com/)

## 下节预告

下节是 **Week 11 Weekend Project: Capture The Flag (CTF) & 模拟审计**。
我们将通过一系列 CTF 挑战（如 Ethernaut）来实战演练漏洞挖掘，并对一个故意留有漏洞的项目进行完整的模拟审计，产出审计报告。
