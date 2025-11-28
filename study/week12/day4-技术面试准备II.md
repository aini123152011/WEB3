# Week 12 Day 4: 技术面试准备 II - 系统设计与安全

**预计学习时间**: 4-6小时  
**难度**: ⭐⭐⭐⭐⭐

## 学习目标

通过本节课程,你将能够:
- 应对 Web3 系统设计面试 (System Design)
- 快速识别代码片段中的安全漏洞
- 解释 Merkle Tree 等核心算法的原理
- 准备行为面试 (Behavioral Questions) 的最佳答案

## 课程内容

### 1. Web3 系统设计 (System Design)

面试官可能会问："请设计一个去中心化交易所 (DEX)" 或 "设计一个 NFT 市场"。

#### 1.1 设计框架

1.  **需求澄清**: 只需要 Swap 还是需要 Liquidity Provision？是 Order Book 还是 AMM？
2.  **核心组件**:
    -   **Smart Contracts**: 核心逻辑 (Pair, Factory, Router)。
    -   **Frontend**: 用户交互 (React, Ethers.js)。
    -   **Indexer**: 数据查询 (The Graph)。
    -   **Off-chain Services**: 订单撮合 (如果是 Order Book)、价格预言机。
3.  **数据流**: 用户签名 -> 广播交易 -> 合约执行 -> 抛出事件 -> Indexer 索引 -> 前端更新。
4.  **安全性考虑**: 权限控制、防重入、滑点保护。

#### 1.2 案例: 设计 NFT 市场

-   **合约**:
    -   `NFTListing`: 存储挂单信息 (Seller, TokenID, Price)。
    -   `buyItem()`: 原子化交易 (转钱给卖家，转 NFT 给买家)。
    -   `cancelListing()`: 取消挂单。
-   **存储选择**:
    -   挂单数据存在链上 (Gas 贵，去中心化) 还是链下数据库 (Gas 省，需要签名验证)？
    -   推荐: **链下订单 (Off-chain Orders)**。卖家签名 `(TokenID, Price, Nonce)`，买家带着签名调用合约 `matchOrder`。
-   **元数据**: IPFS / Arweave。

### 2. 安全审计面试题 (Spot the Bug)

面试官会给出一段代码，问你哪里有问题。

#### 题目 1: 错误的随机数
```solidity
function lottery() public {
    uint random = uint(keccak256(abi.encodePacked(block.timestamp, msg.sender)));
    if (random % 100 == 0) {
        payable(msg.sender).transfer(address(this).balance);
    }
}
```
**漏洞**: 矿工可以操纵 `block.timestamp`，或者攻击者可以在同一个区块内不断回滚交易直到猜中随机数。

#### 题目 2: 这里会发生死锁吗？
```solidity
function distribute() public {
    for(uint i=0; i<users.length; i++) {
        users[i].transfer(1 ether);
    }
}
```
**漏洞**: 如果某个 `users[i]` 是拒绝接收 ETH 的恶意合约，整个 `distribute` 会 revert。这是 DoS 攻击。应使用 `call` 并检查返回值，或者改为 Pull 模式。

#### 题目 3: 权限过大
```solidity
function setOwner(address newOwner) public {
    owner = newOwner;
}
```
**漏洞**: 缺少 `onlyOwner` 修饰符。任何人都可以夺取合约控制权。

### 3. 核心算法与数据结构

#### 3.1 Merkle Tree (默克尔树)

**应用**: 白名单验证 (Airdrop, NFT Mint)。
**原理**: 将所有白名单地址哈希成叶子节点，两两哈希向上合并，直到根节点 (Root)。合约只存 Root。
**验证**: 用户提供 Merkle Proof (路径上的哈希值)，合约计算 `keccak256(proof + leaf)` 是否等于 Root。

**面试题**: 如何用 Solidity 验证 Merkle Proof？
-   参考 OpenZeppelin `MerkleProof.verify(proof, root, leaf)`.

#### 3.2 签名算法 (ECDSA)

**应用**: 链下授权 (Permit), 登录验证。
**流程**:
1.  **Hash**: 对消息进行哈希 `h = keccak256(msg)`.
2.  **Sign**: 私钥对 `h` 签名得到 `(r, s, v)`.
3.  **Recover**: 公钥 (地址) = `ecrecover(h, v, r, s)`.
4.  **Verify**: 检查恢复的地址是否等于预期地址。

### 4. 行为面试 (Behavioral Questions)

#### 4.1 常见问题

-   **"Why Web3?"**: 不要只说"赚钱"。要谈论去中心化、无需许可、数据主权等价值观，或者具体的技术兴趣（如对 EVM 的痴迷）。
-   **"Tell me about a bug you fixed"**: 描述背景 (S)，任务 (T)，行动 (A)，结果 (R)。强调你如何定位问题（Debug 工具），以及修复后的防范措施（加测试）。
-   **"How do you stay updated?"**: 提到具体的来源 (Twitter, EthResear.ch, Hacker News)，表现你的学习热情。

#### 4.2 提问环节 (Reverse Interview)

面试结束时，你问面试官的问题也很重要：
-   "团队目前的审计流程是怎样的？" (显示你关注安全)
-   "你们如何处理技术债务？"
-   "未来的 Roadmap 是什么？"

## 实践练习

### 练习1: 默克尔树实战
1.  使用 JavaScript (`merkletreejs`) 生成一个包含 5 个地址的 Merkle Tree。
2.  获取 Root 和其中一个地址的 Proof。
3.  编写 Solidity 合约验证该 Proof。

### 练习2: 系统设计演练
找一个白板，画出 **Uniswap V2** 的架构图。
-   Factory 如何创建 Pair？
-   Add Liquidity 时 Token 是如何转移的？
-   Swap 时 `k=x*y` 是如何保证的？
-   Flash Swap 的回调逻辑在哪里？

## 学习检查清单

- [ ] 能清晰描述 DEX 或 NFT 市场的完整架构
- [ ] 能一眼看出常见的 Reentrancy, DoS, Overflow 漏洞
- [ ] 理解 Merkle Tree 的验证原理
- [ ] 准备好了 3 个精彩的行为面试故事 (STAR 原则)

## 扩展阅读

-   [System Design Primer](https://github.com/donnemartin/system-design-primer) (通用版，Web3 类似)
-   [Paradigm CTF Solutions](https://github.com/paradigmxyz/paradigm-ctf-2021) (高级安全题)
-   [Merkle Airdrop Starter](https://github.com/Anish-Agnihotri/merkle-airdrop-starter)

## 下节预告

下一节我们将学习**软技能与远程工作**,包括:
- 高效的异步沟通 (Async Communication)
- Web3 常用英语术语
- 如何在 DAO 中通过贡献获得声誉
- 远程工作的自我管理与健康
