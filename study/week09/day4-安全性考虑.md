# Week 9 - Day 4: 跨链安全性

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 学习目标

- ✅ 理解跨链安全模型
- ✅ 掌握重放攻击防护
- ✅ 学习预言机操纵防御
- ✅ 实现紧急暂停机制
- ✅ 分析历史跨链桥攻击案例

---

## Part 1: 跨链安全模型 (1.5小时)

### 1.1 安全假设

跨链桥的安全性取决于最弱的一环。常见的安全假设：
- **验证者诚实假设**: 假设大部分验证者是诚实的 (Multisig Bridge)。
- **经济安全假设**: 假设作恶成本高于收益 (Optimistic Bridge)。
- **代码安全假设**: 假设智能合约没有漏洞。

### 1.2 风险分类

1. **验证风险**: 验证者私钥泄露 (Ronin Bridge)。
2. **协议风险**: 智能合约逻辑漏洞 (Wormhole, Nomad)。
3. **依赖风险**: 依赖的外部系统（如RPC、预言机）被操纵。

---

## Part 2: 重放攻击防护 (1.5小时)

跨链消息必须保证唯一性，防止同一笔交易在目标链被执行多次。

### 2.1 Nonce机制

```solidity
// ReplayProtection.sol
contract ReplayProtection {
    // 源链ID => 下一个预期的Nonce
    mapping(uint16 => uint64) public nextNonce;

    error InvalidNonce(uint64 nonce, uint64 expected);

    function _checkAndIncrementNonce(uint16 srcChainId, uint64 nonce) internal {
        uint64 expected = nextNonce[srcChainId];
        if (nonce != expected) {
            revert InvalidNonce(nonce, expected);
        }
        nextNonce[srcChainId]++;
    }
}
```

### 2.2 唯一哈希机制

对于无序消息，可以使用消息哈希来去重。

```solidity
// HashBasedProtection.sol
contract HashBasedProtection {
    mapping(bytes32 => bool) public processedMessages;

    error MessageAlreadyProcessed();

    function _checkAndMarkProcessed(bytes memory message) internal {
        bytes32 hash = keccak256(message);
        if (processedMessages[hash]) {
            revert MessageAlreadyProcessed();
        }
        processedMessages[hash] = true;
    }
}
```

---

## Part 3: 预言机操纵防御 (1.5小时)

跨链桥常使用预言机获取区块头或资产价格，必须防止操纵。

### 3.1 多预言机聚合

不要依赖单一预言机，使用多个来源并取中位数。

```solidity
// MultiOracle.sol
contract MultiOracle {
    function getPrice(address token) public view returns (uint256) {
        uint256 price1 = chainlinkOracle.getPrice(token);
        uint256 price2 = uniswapOracle.getPrice(token);
        uint256 price3 = api3Oracle.getPrice(token);
        
        // 简单的中位数逻辑
        if (price1 > price2) (price1, price2) = (price2, price1);
        if (price2 > price3) (price2, price3) = (price3, price2);
        if (price1 > price2) (price1, price2) = (price2, price1);
        
        return price2;
    }
}
```

### 3.2 异常值检测

如果新价格与旧价格偏差过大，拒绝更新或暂停系统。

```solidity
uint256 public lastPrice;
uint256 public constant MAX_DEVIATION = 10; // 10%

function updatePrice(uint256 newPrice) external {
    if (lastPrice > 0) {
        uint256 deviation = (abs(newPrice - lastPrice) * 100) / lastPrice;
        require(deviation <= MAX_DEVIATION, "Price deviation too high");
    }
    lastPrice = newPrice;
}
```

---

## Part 4: 紧急暂停与限额 (1.5小时)

### 4.1 紧急暂停 (Emergency Pause)

当检测到异常（如大量资金流出）时，允许管理员或自动化监控系统暂停合约。

```solidity
// EmergencyPausable.sol
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";

contract EmergencyPausable is Pausable, AccessControl {
    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");

    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(GUARDIAN_ROLE, msg.sender);
    }

    function pause() external onlyRole(GUARDIAN_ROLE) {
        _pause();
    }

    function unpause() external onlyRole(DEFAULT_ADMIN_ROLE) {
        _unpause();
    }

    function sensitiveAction() external whenNotPaused {
        // ...
    }
}
```

### 4.2 速率限制 (Rate Limiting)

限制单位时间内的跨链金额，防止黑客一次性盗走所有资金。

```solidity
// RateLimiter.sol
contract RateLimiter {
    uint256 public constant DAILY_LIMIT = 1000000 ether; // 1M USD
    uint256 public currentPeriodStart;
    uint256 public currentPeriodAmount;

    error RateLimitExceeded();

    function _checkRateLimit(uint256 amount) internal {
        if (block.timestamp >= currentPeriodStart + 1 days) {
            currentPeriodStart = block.timestamp;
            currentPeriodAmount = 0;
        }

        currentPeriodAmount += amount;
        if (currentPeriodAmount > DAILY_LIMIT) {
            revert RateLimitExceeded();
        }
    }
}
```

---

## Part 5: 历史攻击分析 (1小时)

### 5.1 Poly Network 攻击

- **原因**: 合约权限管理漏洞，黑客替换了Keeper公钥。
- **教训**: 严格审查特权函数的访问控制。

### 5.2 Wormhole 攻击

- **原因**: 签名验证逻辑漏洞，`verify_signatures` 函数未被正确调用。
- **教训**: 无论是Solidity还是Rust/Solana，核心验证逻辑必须经过形式化验证。

### 5.3 Ronin Bridge 攻击

- **原因**: 验证者私钥泄露（5/9 验证者被控）。
- **教训**: 去中心化验证网络的重要性，不能依赖单一实体的安全措施。

---

## 📝 今日作业

### 作业1: 实现速率限制器

编写一个RateLimiter合约：
1. 支持按Token设置不同的限额
2. 支持每小时/每天重置
3. 超过限额时触发事件并拒绝交易
4. 编写测试模拟大额攻击

### 作业2: 安全审计练习

审查以下代码并找出漏洞：
```solidity
function withdraw(bytes memory signature, uint256 amount) public {
    // Missing nonce check!
    require(verify(signature, msg.sender, amount), "Invalid sig");
    payable(msg.sender).transfer(amount);
}
```
修复漏洞并解释原因。

### 作业3: 监控脚本开发

编写一个监控脚本：
1. 监听跨链桥合约的 `Locked` 和 `Minted` 事件
2. 检查源链锁定金额与目标链铸造金额是否一致
3. 如果发现不一致（可能正在发生攻击），自动调用合约的 `pause()` 函数

---

## ✅ 检查清单

- [ ] 理解跨链桥的信任假设
- [ ] 能够实现Nonce防重放
- [ ] 掌握多预言机聚合逻辑
- [ ] 实现紧急暂停和速率限制
- [ ] 熟悉常见跨链攻击向量

---

## 📅 明日预告

明天学习Layer2集成：
- Arbitrum/Optimism 消息传递
- L1 -> L2 通信
- L2 -> L1 通信
- 跨层资产桥接

**🎉 完成Day 4！安全是跨链的生命线！**
