# Week 11 Day 4: 人工审计方法论

**预计学习时间**: 6-8小时  
**难度**: ⭐⭐⭐⭐

## 学习目标

通过本节课程,你将能够:
- 掌握专业的智能合约审计流程
- 使用标准检查清单 (Checklist) 系统化审查代码
- 发现自动化工具无法识别的业务逻辑漏洞
- 撰写符合行业标准的审计报告

## 课程内容

### 1. 审计流程标准

一个专业的审计通常分为三个阶段：

#### 1.1 预审计 (Pre-audit)
-   **理解目标**: 阅读白皮书、设计文档，理解项目的业务逻辑和经济模型。
-   **确定范围**: 明确哪些合约在审计范围内，哪些是第三方依赖。
-   **工具扫描**: 运行 Slither/Mythril，快速发现低级错误，作为后续参考。

#### 1.2 详细审计 (Audit)
-   **逐行阅读**: 对核心逻辑进行逐行审查。
-   **场景模拟**: 构想极端场景（如闪电贷攻击、管理员跑路、预言机操纵）。
-   **交叉验证**: 团队成员之间交换审计部分，避免思维盲区。

#### 1.3 报告与修复 (Post-audit)
-   **初稿报告**: 提交给客户，包含所有发现的问题。
-   **修复验证**: 客户修复后，复查修复代码，确认漏洞已消除且未引入新 Bug。
-   **最终报告**: 发布公开的最终审计报告。

### 2. 审计检查清单 (Checklist)

使用检查清单可以防止遗漏常见问题。

#### 2.1 访问控制 (Access Control)
- [ ] 谁拥有 Owner/Admin 权限？权限是否过大？
- [ ] 是否有单一私钥风险？是否强制要求多签？
- [ ] 敏感函数（如 mint, pause, upgrade）是否有正确的 modifier？
- [ ] `tx.origin` 是否被误用于鉴权？

#### 2.2 资产安全 (Asset Security)
- [ ] 提款逻辑是否使用了 Check-Effects-Interactions 模式？
- [ ] ETH 转账是否处理了接收方拒绝接收的情况 (DoS)？
- [ ] ERC20 转账是否检查了返回值？(部分 Token 如 USDT 不符合标准)
- [ ] 费率计算是否会导致精度丢失 (Rounding Error)？

#### 2.3 外部依赖 (External Dependencies)
- [ ] 预言机是否使用了加权平均或 TWAP 防止操纵？
- [ ] 外部合约调用是否可能重入？
- [ ] 外部合约是否可升级/可替换？如果替换为恶意合约会怎样？

#### 2.4 业务逻辑 (Business Logic)
- [ ] 质押奖励计算是否随时间线性释放？是否有“抢跑分红”漏洞？
- [ ] 投票治理是否有锁定期？是否可以“闪电贷投票”？
- [ ] 随机数生成是否安全？

### 3. 业务逻辑漏洞案例

自动化工具通常擅长发现代码层面的错误（如重入、溢出），但很难理解业务意图。

#### 案例 1: 闪电贷价格操纵
**场景**: 一个 DeFi 借贷协议，使用 DEX 的瞬时价格（Spot Price）作为预言机。
**攻击**: 攻击者通过闪电贷在 DEX 上砸盘，导致价格瞬间暴跌，然后以极低价格在借贷协议中清算其他用户。
**审计点**: 检查预言机来源，必须使用 Chainlink 或 TWAP。

#### 案例 2: 错误的会计逻辑
**场景**: 一个分红合约，在用户转账代币时，先扣除发送者余额，再分红，最后增加接收者余额。
**漏洞**: 某些通缩型代币（转账带手续费）实际到账金额小于发送金额。如果合约按发送金额记账，会导致金库亏空。
**审计点**: 检查合约是否支持 Deflationary Tokens（通缩代币）。

### 4. 撰写审计报告

#### 4.1 严重等级分类

-   **Critical**: 资金直接损失、权限完全接管、合约锁死。必须立即修复。
-   **High**: 只有在特定困难条件下才能触发的资金损失，或严重影响核心功能。
-   **Medium**: 不会导致资金损失，但可能导致拒绝服务或逻辑错误。
-   **Low**: 违反最佳实践，代码风格问题，未使用的变量。
-   **Gas**: 优化建议。

#### 4.2 报告结构模板

1.  **Executive Summary**: 审计概况，发现的漏洞总数及分布。
2.  **Scope**: 审计的文件列表及 Commit Hash。
3.  **Findings**: 详细的漏洞描述。
    -   **Title**: [Critical] Reentrancy in withdraw function
    -   **Description**: 详细描述漏洞原理。
    -   **Impact**: 造成的后果（如：攻击者可掏空金库）。
    -   **Proof of Concept (PoC)**: 攻击代码片段。
    -   **Recommendation**: 修复建议。
    -   **Status**: Fixed / Acknowledged / Mitigated.

## 实践练习

### 练习1: 模拟预审计

选择一个开源的 DeFi 项目（如 Uniswap V2 或某个简单的 Fork 项目）。
1.  阅读其 README 和文档。
2.  列出其核心合约文件。
3.  运行 Slither，记录初步发现。

### 练习2: 寻找逻辑漏洞

分析以下代码片段：

```solidity
contract RewardPool {
    mapping(address => uint) public balances;
    uint public totalBalance;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
        totalBalance += msg.value;
    }

    function withdraw(uint amount) public {
        require(balances[msg.sender] >= amount);
        balances[msg.sender] -= amount;
        totalBalance -= amount;
        payable(msg.sender).transfer(amount);
    }

    // 分红逻辑：将合约内的“额外”资金平均分给所有人
    // 假设有人直接向合约转账 ETH (不通过 deposit)，这些钱被视为分红
    function distributeRewards() public {
        uint extra = address(this).balance - totalBalance;
        // ... 分发 extra ...
    }
}
```

**问题**: 这里的逻辑漏洞是什么？
**提示**: 如果我通过 `selfdestruct` 强制向合约发送 ETH，或者利用某些操作让 `address(this).balance` 暂时不等于 `totalBalance`，会发生什么？实际上这个逻辑在 DeFi 中很常见，但如果攻击者利用闪电贷存入巨额资金，分红计算可能会被稀释或操纵。

### 练习3: 撰写一份漏洞报告

基于练习 2 发现的问题，按照标准模板写一份漏洞报告。包含 Description, Impact, PoC, Recommendation。

## 学习检查清单

- [ ] 能够描述完整的审计生命周期
- [ ] 拥有自己的审计 Checklist 并不断更新
- [ ] 能够识别 Price Manipulation 和 Accounting Error 等逻辑漏洞
- [ ] 能够区分 Critical 和 High/Medium 的界限
- [ ] 能够撰写清晰、专业的审计报告

## 扩展阅读

-   [OpenZeppelin Audit Reports](https://blog.openzeppelin.com/security-audits/) (阅读优秀的报告是最好的学习方式)
-   [Consensys Audit Reports](https://consensys.net/diligence/audits/)
-   [Trail of Bits Publications](https://github.com/trailofbits/publications)
-   [DeFi Vulnerability Checklist](https://github.com/lz411/checklist)

## 下节预告

下一节我们将学习**DeFi 攻击案例复盘**,包括:
- The DAO 攻击 (Reentrancy 鼻祖)
- Uniswap/Lendf.Me 兼容性问题 (ERC777 钩子)
- Harvest Finance 攻击 (闪电贷操纵价格)
- Poly Network 攻击 (跨链桥鉴权漏洞)
