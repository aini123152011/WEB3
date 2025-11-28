# Week 11 Day 1: 常见智能合约漏洞分析 I

**预计学习时间**: 6-8小时  
**难度**: ⭐⭐⭐⭐

## 学习目标

通过本节课程,你将能够:
- 深入理解重入攻击 (Reentrancy) 的原理与防御
- 掌握整型溢出 (Overflow/Underflow) 的历史背景与现状
- 识别并修复权限控制 (Access Control) 漏洞
- 理解低级调用返回值检查的重要性

## 课程内容

### 1. 重入攻击 (Reentrancy)

这是最著名也是最具破坏力的攻击向量之一（导致了 The DAO 事件）。

#### 1.1 漏洞原理

当合约A调用外部合约B时，合约A将执行控制权移交给了B。如果B是一个恶意合约，它可以在A的函数完成执行（更新状态）之前，再次回调A的函数。

**漏洞代码示例**:
```solidity
// VulnerableBank.sol
mapping(address => uint256) public balances;

function withdraw() public {
    uint256 balance = balances[msg.sender];
    require(balance > 0);

    // 1. Interaction: 发送以太币 (移交控制权)
    (bool sent, ) = msg.sender.call{value: balance}("");
    require(sent, "Failed to send Ether");

    // 2. Effects: 更新余额 (太晚了!)
    balances[msg.sender] = 0;
}
```

**攻击合约**:
```solidity
// Attacker.sol
function attack() external payable {
    vulnerableBank.deposit{value: 1 ether}();
    vulnerableBank.withdraw();
}

// 回调函数
fallback() external payable {
    if (address(vulnerableBank).balance >= 1 ether) {
        vulnerableBank.withdraw(); // 重入!
    }
}
```

#### 1.2 防御措施

1.  **Check-Effects-Interactions 模式**:
    先检查条件，再更新状态（Effects），最后进行外部交互（Interactions）。

    ```solidity
    function withdraw() public {
        uint256 balance = balances[msg.sender];
        require(balance > 0);

        // 1. Effects
        balances[msg.sender] = 0;

        // 2. Interaction
        (bool sent, ) = msg.sender.call{value: balance}("");
        require(sent, "Failed to send Ether");
    }
    ```

2.  **使用 ReentrancyGuard**:
    OpenZeppelin 提供的 `nonReentrant` 修饰符。

    ```solidity
    import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

    function withdraw() public nonReentrant {
        // ...
    }
    ```

### 2. 整型溢出 (Integer Overflow/Underflow)

#### 2.1 原理

在 Solidity 0.8.0 之前，`uint256` 最大值加 1 会变成 0，最小值减 1 会变成最大值。这被称为溢出。

**漏洞代码 (Solidity < 0.8.0)**:
```solidity
function batchTransfer(address[] receivers, uint256 value) public {
    uint256 total = receivers.length * value; // 可能溢出变为一个小数值
    require(balances[msg.sender] >= total); // 检查通过

    balances[msg.sender] -= total;
    for (uint i = 0; i < receivers.length; i++) {
        balances[receivers[i]] += value; // 攻击者凭空制造了代币
    }
}
```

#### 2.2 现状与防御

-   **Solidity >= 0.8.0**: 编译器内置了溢出检查，发生溢出会自动回滚 (Panic)。
-   **Unchecked 块**: 如果你确定不会溢出且想节省 Gas，可以使用 `unchecked { ... }`。

### 3. 权限控制漏洞 (Access Control)

#### 3.1 tx.origin vs msg.sender

-   `msg.sender`: 直接调用当前合约的地址。
-   `tx.origin`: 发起这笔交易的原始地址（EOA）。

**漏洞**: 使用 `tx.origin` 做鉴权。
```solidity
function withdraw() public {
    require(tx.origin == owner); // 危险!
    payable(msg.sender).transfer(address(this).balance);
}
```
**攻击**: 诱骗 owner 调用恶意合约，恶意合约再调用 `withdraw`。此时 `tx.origin` 是 owner，但 `msg.sender` 是恶意合约。

**修复**: 永远使用 `msg.sender` 做鉴权。

#### 3.2 默认可见性

在旧版本 Solidity 中，函数默认是 `public`。如果开发者忘记写可见性，敏感函数可能被任意人调用。
（注：Solidity 0.5.0 之后强制要求显式声明可见性，此漏洞已很少见）。

#### 3.3 缺失的修饰符

最常见的错误是忘记加 `onlyOwner`。

```solidity
function setOwner(address newOwner) public { // 忘记加 onlyOwner
    owner = newOwner;
}
```

### 4. 未检查的返回值

低级调用 `call`, `delegatecall`, `staticcall` 在失败时不会自动 revert，而是返回 `false`。

**漏洞代码**:
```solidity
function safeTransfer(address to, uint256 value) public {
    // 如果转账失败，execution 继续，但用户以为成功了
    token.call(abi.encodeWithSignature("transfer(address,uint256)", to, value));
}
```

**防御**:
始终检查返回值，或使用 OpenZeppelin 的 `SafeERC20`。

## 实践练习

### 练习1: 复现重入攻击

1.  在 Remix 中部署 `VulnerableBank`。
2.  部署 `Attacker` 合约。
3.  向 Bank 存入 10 Ether。
4.  运行 Attack，观察 Bank 余额被掏空。
5.  修复 Bank 合约并重新测试，确保攻击失败。

### 练习2: 权限绕过

1.  编写一个使用 `tx.origin` 鉴权的 `Wallet` 合约。
2.  编写一个 `Phishing` 合约。
3.  模拟受害者调用 `Phishing` 合约，进而导致 `Wallet` 资金被盗。

### 练习3: SafeMath (历史考古)

1.  将 Solidity 编译器版本设为 0.6.0。
2.  演示一个整数溢出漏洞。
3.  引入 `SafeMath` 库修复它。
4.  将编译器切换回 0.8.0，观察不需要 SafeMath 也会自动 revert。

## 学习检查清单

- [ ] 能手绘重入攻击的时序图
- [ ] 熟练运用 Check-Effects-Interactions 模式
- [ ] 知道何时使用 `nonReentrant`
- [ ] 理解 Solidity 0.8.0 前后算术运算的差异
- [ ] 能够区分 `tx.origin` 和 `msg.sender` 的使用场景
- [ ] 养成检查低级调用返回值的习惯

## 扩展阅读

-   [Solidity Security Considerations](https://docs.soliditylang.org/en/v0.8.19/security-considerations.html)
-   [Reentrancy Attack - SWC-107](https://swcregistry.io/docs/SWC-107)
-   [Consensys Best Practices](https://consensys.github.io/smart-contract-best-practices/)

## 下节预告

下一节我们将学习**常见智能合约漏洞分析 II**,包括:
- 抢跑交易 (Front-running)
- 拒绝服务 (DoS) 攻击
- 随机数漏洞 (Bad Randomness)
- 签名重放 (Signature Replay)
