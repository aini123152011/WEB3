# Week 12 Day 2: 构建 Web3 作品集

**预计学习时间**: 4-6小时  
**难度**: ⭐⭐

## 学习目标

通过本节课程,你将能够:
- 优化你的 GitHub Profile 以吸引 Web3 招聘者
- 构建和展示你的链上简历 (On-chain Resume)
- 找到适合的开源项目并进行首次贡献
- 了解 Hackathon 的参赛流程与获胜技巧

## 课程内容

### 1. GitHub: 你的第一张名片

在 Web3 行业，代码即法律，GitHub 即简历。

#### 1.1 优化 Profile 页面

-   **Pinned Repositories**: 置顶 3-6 个你最得意的项目。
    -   最好包含一个完整的 DApp（前端+合约）。
    -   包含一个具有一定技术深度的工具或库（如 Hardhat 插件）。
-   **Contribution Graph**: 保持活跃的提交记录（绿墙）。
-   **README.md**: 每个项目必须有高质量的文档。
    -   项目简介：一句话说清楚它解决了什么问题。
    -   演示 Demo：Live Demo 链接或 GIF 动图。
    -   技术栈：列出使用的工具 (Solidity, React, The Graph...)。
    -   运行指南：`npm install && npm start`。

#### 1.2 优质项目示例

如果你还没有拿得出手的项目，可以从以下方向入手：
1.  **DeFi 仪表盘**: 连接钱包，显示余额，集成 Uniswap 进行 Swap。
2.  **NFT Marketplace**: 铸造、上架、购买 NFT 的全流程。
3.  **DAO 治理工具**: 简单的提案和投票系统。
4.  **复刻经典项目**: 手写一份 Uniswap V2 核心代码并加注释。

### 2. 链上简历 (On-chain Resume)

Web3 原生简历是直接写在区块链上的。

#### 2.1 部署个人名片合约

编写一个简单的合约，存储你的基本信息，并部署到主网（或 Optimism/Polygon 等低 Gas 链）。

```solidity
contract MyResume {
    string public name = "Satoshi Nakamoto";
    string public email = "satoshi@bitcoin.org";
    string public github = "https://github.com/satoshi";
    string[] public skills = ["Solidity", "Rust", "Cryptography"];
    
    mapping(address => string) public endorsements; // 别人对你的评价

    function endorse(string memory message) public {
        endorsements[msg.sender] = message;
    }
}
```
在求职信中附上该合约地址，证明你懂合约开发。

#### 2.2 Web3 社交身份

-   **ENS**: 注册一个 `.eth` 域名（如 `yourname.eth`）。
-   **Lens Protocol / Farcaster**: 建立去中心化社交媒体账号。
-   **Gitcoin Passport**: 验证你的链上身份可信度。
-   **POAP (Proof of Attendance Protocol)**: 收集你参加过的活动徽章，证明你的社区参与度。

### 3. 参与开源贡献

#### 3.1 寻找项目

不要一开始就想改写以太坊内核。从简单的入手：
-   **Good First Issues**: 在 GitHub 搜索标签 `good first issue` 或 `help wanted`。
-   **Documentation**: 修复拼写错误，翻译文档（如 Ethereum.org 翻译计划）。
-   **UI/UX**: 修复前端 CSS Bug。

#### 3.2 贡献流程

1.  **Fork & Clone**: 复制代码到本地。
2.  **Branch**: 创建新分支 `fix/typo-in-readme`。
3.  **Commit & Push**: 提交修改。
4.  **Pull Request (PR)**: 提交 PR，并清晰描述修改内容。
5.  **Code Review**: 耐心等待维护者反馈，并进行修改。

### 4. 赢得 Hackathon (黑客松)

Hackathon 是 Web3 招聘者最看重的经历之一。常见平台：ETHGlobal, Devfolio, DoraHacks。

#### 4.1 参赛策略

-   **组队**: 找互补的队友（1 合约 + 1 前端 + 1 设计/产品）。
-   **选题**: 解决具体问题。DeFi, Infra, DAO, Public Goods 都是好方向。
-   **MVP (Minimum Viable Product)**: 不要追求完美，只要核心流程能跑通。
-   **Presentation**: 准备 2-3 分钟的演示视频，清晰展示产品价值。
-   **Sponsor Prizes**: 关注赞助商（如 Chainlink, Polygon, Graph）的特定奖项，针对性集成他们的技术，获奖概率更大。

### 5. 个人品牌建设

-   **Twitter (X)**: 关注行业大V，转发并评论高质量技术推文。分享你的学习进度 (#100DaysOfCode)。
-   **Mirror.xyz**: Web3 版的 Medium。发布你的技术文章、项目复盘或行业分析。
-   **Discord**: 混迹于你喜欢的项目社区，积极回答新人问题，可能会被官方注意到并招募。

## 实践练习

### 练习1: 完善 GitHub

1.  挑选你之前的 3 个最佳练习项目（如 DEX, NFT Market, DAO）。
2.  为它们分别编写详细的 `README.md`（包含截图）。
3.  将它们 Pin 到你的 GitHub 首页。

### 练习2: 寻找 Open Source Issue

1.  浏览 [OpenZeppelin](https://github.com/OpenZeppelin), [Scaffold-ETH](https://github.com/scaffold-eth/scaffold-eth-2), [Wagmi](https://github.com/wevm/wagmi) 等仓库。
2.  找到一个 `good first issue` 或者仅仅是发现文档中的一个 Typo。
3.  尝试提交一个 PR。

### 练习3: 注册 Link3 或 Lens

访问 [Link3.to](https://link3.to/) 或类似平台，创建一个聚合页，展示你的 GitHub, Twitter, Mirror 和链上成就。

## 学习检查清单

- [ ] GitHub 首页看起来专业且活跃
- [ ] 至少有 2 个含完整文档的置顶项目
- [ ] 拥有一个 ENS 或 Web3 社交账号
- [ ] 了解如何寻找和提交开源贡献
- [ ] 注册了一个即将到来的线上 Hackathon

## 扩展阅读

-   [How to get a job in Web3](https://adhras.hashnode.dev/how-to-get-a-job-in-web3)
-   [First Timers Only](https://www.firsttimersonly.com/) (开源贡献指南)
-   [ETHGlobal Hackathon Guides](https://ethglobal.com/)

## 下节预告

下一节我们将进入**技术面试准备 I**,包括:
- Solidity 语言高频考点 (存储布局, delegatecall, 继承)
- EVM 原理 (Opcode, Gas 计算, 内存管理)
- 智能合约 Gas 优化技巧
- 现场手写代码 (Whiteboard Coding)
