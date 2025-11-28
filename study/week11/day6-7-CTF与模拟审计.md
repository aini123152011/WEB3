# Week 11 Day 6-7: CTF 实战与模拟审计项目

**预计学习时间**: 10-12小时  
**难度**: ⭐⭐⭐⭐⭐

## 项目概述

本周末项目分为两个部分。第一部分通过 Capture The Flag (CTF) 挑战赛的形式，锻炼你的漏洞挖掘直觉。第二部分则是一个完整的模拟审计项目，你将扮演安全审计员的角色，对一个包含多个漏洞的 DeFi 协议进行审计，并产出专业的审计报告。

## 第一部分: CTF 挑战 (4小时)

我们将使用经典的 [Ethernaut](https://ethernaut.openzeppelin.com/) 平台进行练习。

### 挑战 1: Re-entrancy (Level 10)

**目标**: 偷走合约中的所有资金。
**提示**:
- 观察 `withdraw` 函数中的 `call` 调用位置。
- 构造一个恶意合约，在 `fallback` 函数中递归调用 `withdraw`。
- 计算好递归深度，防止 Gas 耗尽。

### 挑战 2: Privacy (Level 12)

**目标**: 解锁 (Unlock) 合约。
**提示**:
- 理解 Solidity 的存储布局 (Storage Layout)。
- `bool` 占 1 byte, `uint256` 占 32 bytes。它们是如何打包 (Packing) 的？
- 使用 `web3.eth.getStorageAt` 读取私有变量。
- 即使变量声明为 `private`，在链上也是公开可见的。

### 挑战 3: Denial (Level 20)

**目标**: 让 Owner 无法提取资金 (DoS)。
**提示**:
- 观察 `withdraw` 函数中的 `partner.call{value:amount}("")`。
- `call` 如果耗尽 Gas 可能会导致整个交易失败（取决于是否检查了返回值，或者是否有 Gas 限制）。
- 构造一个恶意合约，在 `receive` 函数中执行 `while(true) {}` 消耗所有 Gas。

## 第二部分: 模拟审计项目 (8小时)

### 项目背景: VulnerableDeFi

这是一个简易的借贷与质押协议，包含 `LendingPool.sol` 和 `StakingRewards.sol`。开发者声称它"非常安全"，但实际上埋藏了至少 5 个漏洞。

### 合约代码

#### 1. LendingPool.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract LendingPool {
    mapping(address => uint256) public balances;
    IERC20 public token;

    constructor(address _token) {
        token = IERC20(_token);
    }

    function deposit(uint256 amount) public {
        token.transferFrom(msg.sender, address(this), amount);
        balances[msg.sender] += amount;
    }

    function withdraw(uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        // 漏洞 1: 先交互，后更新状态 (Reentrancy)
        token.transfer(msg.sender, amount);
        balances[msg.sender] -= amount;
    }

    // 闪电贷功能
    function flashLoan(uint256 amount) public {
        uint256 balanceBefore = token.balanceOf(address(this));
        require(balanceBefore >= amount, "Not enough liquidity");

        token.transfer(msg.sender, amount);

        // 漏洞 2: 没有任何回调调用，只是检查余额？
        // 实际上，攻击者无法在这一步还款，因为 execution 还在 LendingPool 中
        // 这里需要调用 msg.sender 的某个接口，让它执行逻辑
        // IFlashLoanReceiver(msg.sender).executeOperation(amount);
        
        // 假设这里本来想写回调，但开发者忘记了，或者写错了
        // 如果没有回调，这就不是闪电贷，而是直接送钱
        // 但如果这是意图实现 "乐观转账"，则必须在函数结束前检查余额
        
        uint256 balanceAfter = token.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan not repaid");
    }
}
```

#### 2. StakingRewards.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/Ownable.sol";

contract StakingRewards is Ownable {
    mapping(address => uint256) public stakes;
    uint256 public totalStaked;
    uint256 public rewardRate = 100; // 每秒奖励

    function stake() public payable {
        stakes[msg.sender] += msg.value;
        totalStaked += msg.value;
    }

    function unstake(uint256 amount) public {
        require(stakes[msg.sender] >= amount, "Insufficient stake");
        stakes[msg.sender] -= amount;
        totalStaked -= amount;
        payable(msg.sender).transfer(amount);
    }

    // 漏洞 3: 所有人都可以调用 setRewardRate
    // 忘记加 onlyOwner
    function setRewardRate(uint256 _rate) public {
        rewardRate = _rate;
    }

    // 漏洞 4: 奖励计算溢出风险 (如果 stake 时间很长)
    // 漏洞 5: 逻辑错误，这里并没有真正发奖励，只是计算了一个数值返回
    function getReward(address user) public view returns (uint256) {
        // 简单的示例，实际会有 lastUpdateTime
        return stakes[user] * rewardRate; 
    }
    
    // 假设有一个 claimReward 函数
    function claimReward() public {
        uint256 reward = getReward(msg.sender);
        // 漏洞 6: 合约里有钱发奖励吗？stake 的钱是本金
        // 如果直接动用本金发奖励，最后取款的人将无法取回本金 (庞氏骗局)
        payable(msg.sender).transfer(reward);
    }
}
```

### 审计任务

1.  **预审计**: 阅读代码，理解业务逻辑。
2.  **漏洞挖掘**: 找出代码中的漏洞（至少 5 个）。
3.  **分类**: 为每个漏洞评级 (Critical, High, Medium, Low)。
4.  **报告撰写**: 编写一份 Markdown 格式的审计报告。

### 审计报告模板

```markdown
# VulnerableDeFi Audit Report

## Executive Summary
本报告对 VulnerableDeFi 协议进行了全面的安全审计...

## Findings

### [Critical] Reentrancy in LendingPool.withdraw
**Description**: `withdraw` 函数在更新 `balances` 之前就进行了 `token.transfer`。如果 `token` 是 ERC777 或其他带回调的代币，攻击者可以重入 `withdraw` 函数。
**Impact**: 攻击者可以掏空 LendingPool 的所有流动性。
**Recommendation**: 使用 Checks-Effects-Interactions 模式，或添加 `nonReentrant` 修饰符。

### [High] Unprotected Access Control in setRewardRate
**Description**: ...
...
```

## 交付物

1.  **CTF 解题脚本**: Ethernaut 挑战的攻击合约代码和 JS 脚本。
2.  **审计报告**: `Audit-Report.md`，包含你发现的所有漏洞及其修复建议。
3.  **修复后的合约**: `FixedLendingPool.sol` 和 `FixedStakingRewards.sol`。

## 总结与反思

通过这个项目，你不仅站在攻击者的角度体验了“黑客”的快感，更重要的是站在防御者的角度，学会了如何系统性地发现和修复问题。安全不是一个状态，而是一个过程。保持敬畏之心，永远不要假设你的代码是完美的。

下周是课程的最后一周，我们将聚焦于**Web3 职业发展与面试准备**，助你顺利拿到 Offer。
