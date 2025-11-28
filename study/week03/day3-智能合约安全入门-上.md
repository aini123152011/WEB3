# Week 3 - Day 3: 智能合约安全入门（上）

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐ (高级)

## 📚 今日学习目标

- ✅ 了解常见安全漏洞
- ✅ 掌握重入攻击原理和防御
- ✅ 理解整数溢出问题
- ✅ 学习访问控制漏洞
- ✅ 掌握前端运行攻击

---

## 🔐 Part 1: 重入攻击 (2小时)

### 1.1 重入攻击原理

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ❌ 存在重入漏洞的合约
 */
contract VulnerableBank {
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    // 漏洞：先转账，后更新余额
    function withdraw(uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        // ❌ 危险：外部调用在状态更新之前
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        
        // 余额更新在外部调用之后
        balances[msg.sender] -= amount;
    }
    
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}

/**
 * 攻击合约
 */
contract ReentrancyAttack {
    VulnerableBank public bank;
    uint256 public constant AMOUNT = 1 ether;
    
    constructor(address _bankAddress) {
        bank = VulnerableBank(_bankAddress);
    }
    
    // 开始攻击
    function attack() public payable {
        require(msg.value >= AMOUNT, "Need at least 1 ETH");
        
        // 先存款
        bank.deposit{value: AMOUNT}();
        
        // 开始重入攻击
        bank.withdraw(AMOUNT);
    }
    
    // fallback函数：接收ETH时再次调用withdraw
    receive() external payable {
        if (address(bank).balance >= AMOUNT) {
            bank.withdraw(AMOUNT);
        }
    }
    
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

### 1.2 防御方法1：Checks-Effects-Interactions

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ✅ 使用CEI模式防御
 */
contract SecureBank1 {
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) public {
        // 1. Checks：检查条件
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        // 2. Effects：更新状态
        balances[msg.sender] -= amount;
        
        // 3. Interactions：外部交互
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### 1.3 防御方法2：重入锁

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ✅ 使用重入锁防御
 */
contract SecureBank2 {
    mapping(address => uint256) public balances;
    
    bool private locked;
    
    modifier noReentrancy() {
        require(!locked, "No reentrancy");
        locked = true;
        _;
        locked = false;
    }
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) public noReentrancy {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        balances[msg.sender] -= amount;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### 1.4 OpenZeppelin的ReentrancyGuard

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/**
 * ✅ 使用OpenZeppelin的ReentrancyGuard
 */
contract SecureBank3 is ReentrancyGuard {
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) public nonReentrant {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        balances[msg.sender] -= amount;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

---

## 🔢 Part 2: 整数溢出 (1.5小时)

### 2.1 整数溢出原理

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;  // 注意：0.8.0之前的版本

/**
 * ❌ Solidity 0.7.6及之前版本存在溢出问题
 */
contract VulnerableOverflow {
    mapping(address => uint256) public balances;
    uint256 public totalSupply;
    
    function deposit() public payable {
        // 可能溢出！
        balances[msg.sender] += msg.value;
        totalSupply += msg.value;
    }
    
    function transfer(address to, uint256 amount) public {
        // 可能下溢！
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}

/**
 * 攻击示例
 */
contract OverflowAttack {
    VulnerableOverflow public vulnerable;
    
    constructor(address _vulnerable) {
        vulnerable = VulnerableOverflow(_vulnerable);
    }
    
    function attack() public {
        // 尝试使余额下溢
        // 如果balances[address(this)] = 0
        // 0 - 1 = 2^256 - 1 (下溢)
        vulnerable.transfer(msg.sender, 1);
    }
}
```

### 2.2 Solidity 0.8.0+自动检查

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ✅ Solidity 0.8.0+自动检查溢出
 */
contract SafeFromOverflow {
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        // 自动检查溢出
        balances[msg.sender] += msg.value;
    }
    
    function transfer(address to, uint256 amount) public {
        // 自动检查下溢
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}
```

### 2.3 使用unchecked（谨慎！）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract UncheckedExample {
    // ✅ 安全使用unchecked
    function safeUnchecked(uint256 n) public pure returns (uint256) {
        uint256 sum = 0;
        for (uint256 i = 0; i < n;) {
            sum += i;
            unchecked {
                i++;  // i不会溢出到最大值
            }
        }
        return sum;
    }
    
    // ❌ 危险的unchecked使用
    function dangerousUnchecked(uint256 a, uint256 b) public pure returns (uint256) {
        unchecked {
            return a - b;  // 如果a < b会下溢！
        }
    }
}
```

---

## 🔑 Part 3: 访问控制漏洞 (1.5小时)

### 3.1 未保护的函数

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ❌ 缺少访问控制
 */
contract VulnerableAccess {
    address public owner;
    uint256 public value;
    
    constructor() {
        owner = msg.sender;
    }
    
    // ❌ 任何人都可以调用！
    function setValue(uint256 _value) public {
        value = _value;
    }
    
    // ❌ 任何人都可以提取！
    function withdraw() public {
        payable(msg.sender).transfer(address(this).balance);
    }
}

/**
 * ✅ 正确的访问控制
 */
contract SecureAccess {
    address public owner;
    uint256 public value;
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    // ✅ 只有owner可以调用
    function setValue(uint256 _value) public onlyOwner {
        value = _value;
    }
    
    // ✅ 只有owner可以提取
    function withdraw() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }
    
    // 转移所有权
    function transferOwnership(address newOwner) public onlyOwner {
        require(newOwner != address(0), "Invalid address");
        owner = newOwner;
    }
}
```

### 3.2 基于角色的访问控制

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ✅ 使用RBAC（基于角色的访问控制）
 */
contract RoleBasedAccess {
    mapping(bytes32 => mapping(address => bool)) private roles;
    
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN");
    bytes32 public constant MINTER_ROLE = keccak256("MINTER");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER");
    
    event RoleGranted(bytes32 indexed role, address indexed account);
    event RoleRevoked(bytes32 indexed role, address indexed account);
    
    constructor() {
        _grantRole(ADMIN_ROLE, msg.sender);
    }
    
    modifier onlyRole(bytes32 role) {
        require(hasRole(role, msg.sender), "Access denied");
        _;
    }
    
    function hasRole(bytes32 role, address account) public view returns (bool) {
        return roles[role][account];
    }
    
    function grantRole(bytes32 role, address account) public onlyRole(ADMIN_ROLE) {
        _grantRole(role, account);
    }
    
    function revokeRole(bytes32 role, address account) public onlyRole(ADMIN_ROLE) {
        _revokeRole(role, account);
    }
    
    function _grantRole(bytes32 role, address account) private {
        if (!roles[role][account]) {
            roles[role][account] = true;
            emit RoleGranted(role, account);
        }
    }
    
    function _revokeRole(bytes32 role, address account) private {
        if (roles[role][account]) {
            roles[role][account] = false;
            emit RoleRevoked(role, account);
        }
    }
    
    // 示例：只有MINTER可以铸造
    function mint(address to, uint256 amount) public onlyRole(MINTER_ROLE) {
        // 铸造逻辑
    }
    
    // 示例：只有PAUSER可以暂停
    function pause() public onlyRole(PAUSER_ROLE) {
        // 暂停逻辑
    }
}
```

### 3.3 OpenZeppelin的AccessControl

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/AccessControl.sol";

/**
 * ✅ 使用OpenZeppelin的AccessControl
 */
contract MyContract is AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    
    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(MINTER_ROLE, msg.sender);
    }
    
    function mint(address to, uint256 amount) public onlyRole(MINTER_ROLE) {
        // 铸造逻辑
    }
}
```

---

## 🎯 Part 4: 前端运行攻击 (1小时)

### 4.1 前端运行原理

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ❌ 容易受到前端运行攻击
 */
contract VulnerableAuction {
    address public highestBidder;
    uint256 public highestBid;
    
    function bid() public payable {
        require(msg.value > highestBid, "Bid too low");
        
        // 退还之前的最高出价
        if (highestBidder != address(0)) {
            payable(highestBidder).transfer(highestBid);
        }
        
        highestBidder = msg.sender;
        highestBid = msg.value;
    }
}

/**
 * 攻击者可以：
 * 1. 监控mempool中的出价交易
 * 2. 以更高的gas price抢先出价
 * 3. 成为最高出价者
 */
```

### 4.2 防御方法：提交-揭示模式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ✅ 使用提交-揭示模式
 */
contract SecureAuction {
    struct Bid {
        bytes32 commitment;  // 出价的哈希
        uint256 deposit;     // 存款
        bool revealed;       // 是否已揭示
    }
    
    mapping(address => Bid) public bids;
    
    address public highestBidder;
    uint256 public highestBid;
    
    uint256 public commitEndTime;
    uint256 public revealEndTime;
    
    constructor(uint256 _commitDuration, uint256 _revealDuration) {
        commitEndTime = block.timestamp + _commitDuration;
        revealEndTime = commitEndTime + _revealDuration;
    }
    
    // 第一阶段：提交出价的哈希
    function commitBid(bytes32 commitment) public payable {
        require(block.timestamp < commitEndTime, "Commit period ended");
        require(bids[msg.sender].commitment == bytes32(0), "Already committed");
        
        bids[msg.sender] = Bid({
            commitment: commitment,
            deposit: msg.value,
            revealed: false
        });
    }
    
    // 第二阶段：揭示真实出价
    function revealBid(uint256 amount, bytes32 secret) public {
        require(block.timestamp >= commitEndTime, "Commit period not ended");
        require(block.timestamp < revealEndTime, "Reveal period ended");
        
        Bid storage bid = bids[msg.sender];
        require(bid.commitment != bytes32(0), "No commitment");
        require(!bid.revealed, "Already revealed");
        
        // 验证承诺
        bytes32 commitment = keccak256(abi.encodePacked(amount, secret));
        require(commitment == bid.commitment, "Invalid reveal");
        
        bid.revealed = true;
        
        // 检查是否是最高出价
        if (amount > highestBid && bid.deposit >= amount) {
            highestBidder = msg.sender;
            highestBid = amount;
        }
    }
    
    // 提取未中标的存款
    function withdraw() public {
        require(block.timestamp >= revealEndTime, "Auction not ended");
        
        Bid storage bid = bids[msg.sender];
        require(bid.revealed, "Not revealed");
        require(msg.sender != highestBidder, "Winner cannot withdraw");
        
        uint256 refund = bid.deposit;
        bid.deposit = 0;
        
        payable(msg.sender).transfer(refund);
    }
}
```

---

## 🛡️ Part 5: 其他常见漏洞 (1小时)

### 5.1 tx.origin vs msg.sender

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ❌ 使用tx.origin进行授权
 */
contract VulnerableTxOrigin {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    // ❌ 危险！使用tx.origin
    function transfer(address to, uint256 amount) public {
        require(tx.origin == owner, "Not owner");
        payable(to).transfer(amount);
    }
}

/**
 * 攻击合约
 */
contract TxOriginAttack {
    VulnerableTxOrigin public vulnerable;
    
    constructor(address _vulnerable) {
        vulnerable = VulnerableTxOrigin(_vulnerable);
    }
    
    // 诱骗owner调用这个函数
    function attack() public {
        // tx.origin仍然是owner
        // 但msg.sender是攻击合约
        vulnerable.transfer(address(this), 1 ether);
    }
    
    receive() external payable {}
}

/**
 * ✅ 正确使用msg.sender
 */
contract SecureMsgSender {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    // ✅ 使用msg.sender
    function transfer(address to, uint256 amount) public {
        require(msg.sender == owner, "Not owner");
        payable(to).transfer(amount);
    }
}
```

### 5.2 委托调用漏洞

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ❌ 不安全的delegatecall
 */
contract VulnerableDelegateCall {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    // ❌ 任何人都可以通过delegatecall修改storage
    function execute(address target, bytes memory data) public {
        target.delegatecall(data);
    }
}

/**
 * 攻击合约
 */
contract DelegateCallAttack {
    address public owner;  // storage slot 0
    
    function pwn() public {
        owner = msg.sender;  // 修改调用者的storage slot 0
    }
}

/**
 * ✅ 安全的delegatecall
 */
contract SecureDelegateCall {
    address public owner;
    mapping(address => bool) public trustedTargets;
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    function addTrustedTarget(address target) public onlyOwner {
        trustedTargets[target] = true;
    }
    
    // ✅ 只能delegatecall可信目标
    function execute(address target, bytes memory data) public onlyOwner {
        require(trustedTargets[target], "Target not trusted");
        (bool success, ) = target.delegatecall(data);
        require(success, "Delegatecall failed");
    }
}
```

### 5.3 随机数漏洞

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ❌ 不安全的随机数生成
 */
contract VulnerableRandom {
    // ❌ 矿工可以操纵这些值
    function badRandom() public view returns (uint256) {
        return uint256(keccak256(abi.encodePacked(
            block.timestamp,
            block.difficulty,
            msg.sender
        )));
    }
}

/**
 * ✅ 使用Chainlink VRF获取真随机数
 */
import "@chainlink/contracts/src/v0.8/VRFConsumerBase.sol";

contract SecureRandom is VRFConsumerBase {
    bytes32 internal keyHash;
    uint256 internal fee;
    uint256 public randomResult;
    
    constructor(
        address vrfCoordinator,
        address link,
        bytes32 _keyHash,
        uint256 _fee
    ) VRFConsumerBase(vrfCoordinator, link) {
        keyHash = _keyHash;
        fee = _fee;
    }
    
    function getRandomNumber() public returns (bytes32 requestId) {
        require(LINK.balanceOf(address(this)) >= fee, "Not enough LINK");
        return requestRandomness(keyHash, fee);
    }
    
    function fulfillRandomness(bytes32 requestId, uint256 randomness) 
        internal 
        override 
    {
        randomResult = randomness;
    }
}
```

---

## 📝 今日作业

### 作业1: 安全审计

审计以下合约并找出所有漏洞：
```solidity
contract AuditMe {
    mapping(address => uint256) public balances;
    address public owner;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw() public {
        uint256 amount = balances[msg.sender];
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success);
        balances[msg.sender] = 0;
    }
    
    function transferOwnership(address newOwner) public {
        require(tx.origin == owner);
        owner = newOwner;
    }
}
```

### 作业2: 实现安全的拍卖

实现一个安全的荷兰式拍卖：
1. 价格随时间递减
2. 防止前端运行
3. 安全的退款机制
4. 访问控制

### 作业3: 重入攻击演练

1. 部署漏洞合约到测试网
2. 编写攻击合约
3. 执行攻击
4. 修复漏洞并重新测试

---

## ✅ 今日检查清单

- [ ] 理解重入攻击原理
- [ ] 掌握多种防御方法
- [ ] 了解整数溢出问题
- [ ] 掌握访问控制最佳实践
- [ ] 理解前端运行攻击
- [ ] 完成所有作业

---

## 🆘 常见问题FAQ

### Q1: 为什么要先更新状态再转账？

A: 这是CEI模式的核心。如果先转账，接收方的fallback函数可能再次调用withdraw，此时余额还未更新，导致重入。

### Q2: Solidity 0.8.0+还需要SafeMath吗？

A: 不需要。0.8.0+自动检查溢出，除非使用unchecked块。

### Q3: tx.origin和msg.sender有什么区别？

A:
- tx.origin: 交易的原始发起者（EOA）
- msg.sender: 直接调用者（可能是合约）

永远不要用tx.origin做授权！

---

## 📅 明日预告

明天继续学习智能合约安全：
- 更多攻击向量
- 安全开发最佳实践
- 使用Slither进行静态分析
- 模糊测试

**🎉 完成Day 3！保持警惕！**
