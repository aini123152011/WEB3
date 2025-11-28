# Week 3 - Day 4: 智能合约安全入门（下）

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐ (高级)

## 📚 今日学习目标

- ✅ 学习DoS攻击向量
- ✅ 掌握价格操纵攻击
- ✅ 了解闪电贷攻击
- ✅ 学习安全开发最佳实践
- ✅ 使用Slither进行静态分析

---

## 🚫 Part 1: DoS拒绝服务攻击 (1.5小时)

### 1.1 Gas限制DoS

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ❌ 易受Gas限制DoS攻击
 */
contract VulnerableAirdrop {
    address[] public recipients;
    
    function addRecipient(address recipient) public {
        recipients.push(recipient);
    }
    
    // ❌ 如果recipients太多，会超过gas限制
    function distributeTokens(uint256 amount) public {
        for (uint256 i = 0; i < recipients.length; i++) {
            payable(recipients[i]).transfer(amount);
        }
    }
}

/**
 * ✅ 使用拉取模式避免DoS
 */
contract SecureAirdrop {
    mapping(address => uint256) public allocations;
    mapping(address => bool) public claimed;
    
    function setAllocation(address recipient, uint256 amount) public {
        allocations[recipient] = amount;
    }
    
    // 用户主动领取
    function claim() public {
        require(!claimed[msg.sender], "Already claimed");
        require(allocations[msg.sender] > 0, "No allocation");
        
        claimed[msg.sender] = true;
        payable(msg.sender).transfer(allocations[msg.sender]);
    }
}
```

### 1.2 失败的外部调用DoS

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ❌ 外部调用失败导致DoS
 */
contract VulnerableRefund {
    mapping(address => uint256) public refunds;
    address[] public refundAddresses;
    
    function processRefunds() public {
        for (uint256 i = 0; i < refundAddresses.length; i++) {
            address refundAddress = refundAddresses[i];
            uint256 refundAmount = refunds[refundAddress];
            
            // ❌ 如果某个transfer失败，整个函数回滚
            payable(refundAddress).transfer(refundAmount);
            refunds[refundAddress] = 0;
        }
    }
}

/**
 * ✅ 独立处理每个退款
 */
contract SecureRefund {
    mapping(address => uint256) public refunds;
    
    function withdraw() public {
        uint256 refundAmount = refunds[msg.sender];
        require(refundAmount > 0, "No refund");
        
        refunds[msg.sender] = 0;
        
        (bool success, ) = payable(msg.sender).call{value: refundAmount}("");
        if (!success) {
            // 退款失败，恢复状态
            refunds[msg.sender] = refundAmount;
        }
    }
}
```

---

## 💰 Part 2: 价格操纵攻击 (2小时)

### 2.1 预言机操纵

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IUniswapV2Pair {
    function getReserves() external view returns (uint112, uint112, uint32);
}

/**
 * ❌ 直接使用即时价格
 */
contract VulnerablePriceOracle {
    IUniswapV2Pair public pair;
    
    constructor(address _pair) {
        pair = IUniswapV2Pair(_pair);
    }
    
    // ❌ 可被闪电贷操纵
    function getPrice() public view returns (uint256) {
        (uint112 reserve0, uint112 reserve1, ) = pair.getReserves();
        return (uint256(reserve1) * 1e18) / uint256(reserve0);
    }
}

/**
 * ✅ 使用TWAP（时间加权平均价格）
 */
contract SecurePriceOracle {
    IUniswapV2Pair public pair;
    
    uint256 public price0CumulativeLast;
    uint256 public price1CumulativeLast;
    uint32 public blockTimestampLast;
    
    uint256 public price0Average;
    uint256 public price1Average;
    
    constructor(address _pair) {
        pair = IUniswapV2Pair(_pair);
        (,, blockTimestampLast) = pair.getReserves();
    }
    
    function update() external {
        (uint112 reserve0, uint112 reserve1, uint32 blockTimestamp) = pair.getReserves();
        
        uint32 timeElapsed = blockTimestamp - blockTimestampLast;
        require(timeElapsed >= 1 minutes, "Too soon");
        
        // 计算累积价格
        price0Average = (price0CumulativeLast / timeElapsed);
        price1Average = (price1CumulativeLast / timeElapsed);
        
        blockTimestampLast = blockTimestamp;
    }
}
```

### 2.2 Chainlink价格预言机

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

/**
 * ✅ 使用Chainlink预言机
 */
contract ChainlinkPriceConsumer {
    AggregatorV3Interface internal priceFeed;
    
    constructor(address _priceFeed) {
        priceFeed = AggregatorV3Interface(_priceFeed);
    }
    
    function getLatestPrice() public view returns (int256) {
        (
            uint80 roundId,
            int256 price,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();
        
        // 验证数据有效性
        require(price > 0, "Invalid price");
        require(updatedAt > 0, "Round not complete");
        require(answeredInRound >= roundId, "Stale price");
        
        return price;
    }
}
```

---

## ⚡ Part 3: 闪电贷攻击 (2小时)

### 3.1 闪电贷基础

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IFlashLoanReceiver {
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external returns (bool);
}

/**
 * 简化的闪电贷提供者
 */
contract FlashLoanProvider {
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function flashLoan(
        address receiver,
        uint256 amount,
        bytes calldata params
    ) external {
        uint256 balanceBefore = address(this).balance;
        require(balanceBefore >= amount, "Insufficient liquidity");
        
        uint256 premium = (amount * 9) / 10000; // 0.09%费用
        
        // 发送资金给接收者
        payable(receiver).transfer(amount);
        
        // 调用接收者的executeOperation
        require(
            IFlashLoanReceiver(receiver).executeOperation(
                address(0),  // ETH
                amount,
                premium,
                msg.sender,
                params
            ),
            "Flash loan execution failed"
        );
        
        // 确认还款
        require(
            address(this).balance >= balanceBefore + premium,
            "Flash loan not repaid"
        );
    }
}
```

### 3.2 闪电贷攻击示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ❌ 易受闪电贷攻击的借贷协议
 */
contract VulnerableLending {
    mapping(address => uint256) public deposits;
    mapping(address => uint256) public borrows;
    
    uint256 public constant COLLATERAL_RATIO = 150; // 150%抵押率
    
    function deposit() public payable {
        deposits[msg.sender] += msg.value;
    }
    
    function borrow(uint256 amount) public {
        uint256 maxBorrow = (deposits[msg.sender] * 100) / COLLATERAL_RATIO;
        require(borrows[msg.sender] + amount <= maxBorrow, "Insufficient collateral");
        
        borrows[msg.sender] += amount;
        payable(msg.sender).transfer(amount);
    }
    
    function repay() public payable {
        require(borrows[msg.sender] >= msg.value, "Overpay");
        borrows[msg.sender] -= msg.value;
    }
}

/**
 * 闪电贷攻击合约
 */
contract FlashLoanAttacker is IFlashLoanReceiver {
    FlashLoanProvider public provider;
    VulnerableLending public lending;
    
    constructor(address _provider, address _lending) {
        provider = FlashLoanProvider(_provider);
        lending = VulnerableLending(_lending);
    }
    
    function attack(uint256 loanAmount) external {
        // 1. 借入闪电贷
        provider.flashLoan(address(this), loanAmount, "");
    }
    
    function executeOperation(
        address,
        uint256 amount,
        uint256 premium,
        address,
        bytes calldata
    ) external override returns (bool) {
        // 2. 使用借来的资金存款
        lending.deposit{value: amount}();
        
        // 3. 基于大额存款借出最大额度
        uint256 borrowAmount = (amount * 100) / lending.COLLATERAL_RATIO();
        lending.borrow(borrowAmount);
        
        // 4. 归还闪电贷
        uint256 repayAmount = amount + premium;
        payable(address(provider)).transfer(repayAmount);
        
        // 5. 利润 = borrowAmount - repayAmount
        return true;
    }
    
    receive() external payable {}
}
```

---

## 🛡️ Part 4: 安全开发最佳实践 (1.5小时)

### 4.1 安全检查清单

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ✅ 安全合约模板
 */
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

contract SecureContract is ReentrancyGuard, Ownable, Pausable {
    // 1. ✅ 使用最新的Solidity版本
    // 2. ✅ 使用OpenZeppelin标准库
    
    // 状态变量
    mapping(address => uint256) public balances;
    
    // 3. ✅ 使用事件记录重要操作
    event Deposit(address indexed user, uint256 amount);
    event Withdrawal(address indexed user, uint256 amount);
    
    constructor() Ownable(msg.sender) {}
    
    // 4. ✅ 添加暂停功能
    function deposit() external payable whenNotPaused {
        require(msg.value > 0, "Amount must be positive");
        
        balances[msg.sender] += msg.value;
        
        emit Deposit(msg.sender, msg.value);
    }
    
    // 5. ✅ 使用CEI模式和重入保护
    function withdraw(uint256 amount) external nonReentrant whenNotPaused {
        // Checks
        require(amount > 0, "Amount must be positive");
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        // Effects
        balances[msg.sender] -= amount;
        
        // Interactions
        (bool success, ) = payable(msg.sender).call{value: amount}("");
        require(success, "Transfer failed");
        
        emit Withdrawal(msg.sender, amount);
    }
    
    // 6. ✅ 紧急暂停
    function pause() external onlyOwner {
        _pause();
    }
    
    function unpause() external onlyOwner {
        _unpause();
    }
    
    // 7. ✅ 限制gas使用
    function limitedOperation() external view returns (uint256) {
        // 避免无界循环
        return balances[msg.sender];
    }
}
```

### 4.2 代码审查检查点

```markdown
## 安全审计检查清单

### 重入攻击
- [ ] 所有外部调用后更新状态
- [ ] 使用ReentrancyGuard或CEI模式
- [ ] 避免call()中的value参数

### 访问控制
- [ ] 所有敏感函数有权限检查
- [ ] 使用msg.sender而非tx.origin
- [ ] 正确实现所有权转移

### 整数安全
- [ ] 使用Solidity 0.8.0+
- [ ] 谨慎使用unchecked
- [ ] 检查除零错误

### Gas优化
- [ ] 避免无界循环
- [ ] 使用拉取模式而非推送
- [ ] 缓存storage读取

### 外部调用
- [ ] 检查call/delegatecall返回值
- [ ] 限制gas消耗
- [ ] 避免暴露delegatecall

### 随机数
- [ ] 不使用block变量生成随机数
- [ ] 使用Chainlink VRF
- [ ] 提交-揭示模式

### 价格预言机
- [ ] 使用可信预言机（Chainlink）
- [ ] 实现TWAP防止操纵
- [ ] 验证价格数据新鲜度
```

---

## 🔍 Part 5: Slither静态分析 (1小时)

### 5.1 安装Slither

```bash
# 安装Slither
pip3 install slither-analyzer

# 验证安装
slither --version
```

### 5.2 使用Slither

```bash
# 分析单个合约
slither contracts/MyContract.sol

# 分析整个项目
slither .

# 输出JSON格式
slither . --json output.json

# 只显示高危漏洞
slither . --filter-paths "node_modules|test"

# 生成报告
slither . --print human-summary
```

### 5.3 Slither检测示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 包含多个Slither会检测的问题
 */
contract SlitherExample {
    address public owner;
    
    // ❌ Slither警告：未初始化的状态变量
    uint256 public uninitializedVar;
    
    constructor() {
        owner = msg.sender;
    }
    
    // ❌ Slither警告：tx.origin用于授权
    function dangerousAuth() public {
        require(tx.origin == owner, "Not owner");
    }
    
    // ❌ Slither警告：外部调用前状态修改
    function reentrancyVulnerable() public {
        uint256 balance = address(this).balance;
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success);
        // 状态在外部调用后才修改
    }
    
    // ❌ Slither警告：未检查的低级调用
    function uncheckedCall(address target) public {
        target.call("");  // 未检查返回值
    }
    
    // ❌ Slither警告：严格相等性比较
    function timestampComparison() public view returns (bool) {
        return block.timestamp == 123456;
    }
}
```

### 5.4 修复Slither警告

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * ✅ 修复后的合约
 */
contract FixedContract is ReentrancyGuard {
    address public owner;
    uint256 public initializedVar = 0;  // ✅ 初始化
    
    constructor() {
        owner = msg.sender;
    }
    
    // ✅ 使用msg.sender
    function safeAuth() public view {
        require(msg.sender == owner, "Not owner");
    }
    
    // ✅ CEI模式 + 重入保护
    function safeWithdraw() public nonReentrant {
        uint256 balance = address(this).balance;
        
        // 先更新状态（虽然这里没有状态）
        
        // 再外部调用
        (bool success, ) = msg.sender.call{value: balance}("");
        require(success, "Transfer failed");
    }
    
    // ✅ 检查返回值
    function checkedCall(address target) public returns (bool) {
        (bool success, ) = target.call("");
        require(success, "Call failed");
        return success;
    }
    
    // ✅ 使用范围比较
    function betterTimestampCheck() public view returns (bool) {
        return block.timestamp > 123456;
    }
}
```

---

## 📝 今日作业

### 作业1: 闪电贷攻击实战

1. 在测试网部署一个简单的借贷协议
2. 实现闪电贷攻击合约
3. 执行攻击并分析
4. 修复漏洞

### 作业2: Slither审计报告

1. 使用Slither分析之前的作业合约
2. 修复所有高危和中危问题
3. 生成审计报告
4. 编写修复文档

### 作业3: 安全DeFi协议

设计并实现一个安全的借贷协议：
1. 防止闪电贷攻击
2. 使用可信价格预言机
3. 实现紧急暂停
4. 完整的访问控制
5. 通过Slither审计

---

## ✅ 今日检查清单

- [ ] 理解DoS攻击向量
- [ ] 掌握价格操纵防御
- [ ] 了解闪电贷原理
- [ ] 学会使用Slither
- [ ] 掌握安全开发实践
- [ ] 完成所有作业

---

## 🆘 常见问题FAQ

### Q1: 如何防止闪电贷攻击？

A:
1. 使用可信价格预言机（Chainlink）
2. 实现TWAP避免即时价格操纵
3. 限制单笔交易影响
4. 添加时间延迟机制

### Q2: Slither的误报如何处理？

A: 
1. 仔细review警告原因
2. 如果确认安全，添加注释说明
3. 使用slither.config.json配置忽略
4. 升级Slither到最新版本

### Q3: 生产环境必须的安全措施？

A:
1. 多次专业审计
2. 漏洞赏金计划
3. 紧急暂停机制
4. 时间锁和多签
5. 监控和告警系统

---

## 📅 明日预告

明天学习DeFi协议：
- Uniswap原理
- AMM自动做市商
- 流动性挖矿
- Staking质押

**🎉 完成Day 4！安全第一！**
