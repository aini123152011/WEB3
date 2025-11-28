# Week 12 Day 3: 技术面试准备 I - Solidity 与 EVM

**预计学习时间**: 4-6小时  
**难度**: ⭐⭐⭐⭐

## 学习目标

通过本节课程,你将能够:
- 回答高频 Solidity 面试题
- 深入解释 EVM 的工作原理 (Storage, Memory, Stack)
- 掌握智能合约 Gas 优化的高级技巧
- 在白板上徒手编写核心合约逻辑

## 课程内容

### 1. Solidity 高频考点

#### 1.1 Storage Layout (存储布局)
**Q: `uint128 a; uint128 b; uint256 c;` 占用几个 Slot？**
A: 2 个 Slot。`a` 和 `b` 会被打包进 Slot 0，`c` 占用 Slot 1。

**Q: 动态数组和 Mapping 如何存储？**
A:
-   **Dynamic Array**: 长度存储在 `keccak256(slot)` 位置，数据从 `keccak256(slot)` 开始连续存储。
-   **Mapping**: 值存储在 `keccak256(key . slot)` 位置。Mapping 本身不存储长度。

#### 1.2 Call vs DelegateCall
**Q: 区别是什么？**
A:
-   `call`: 代码在被调用者(Target)的上下文中执行，修改 Target 的 Storage。
-   `delegatecall`: 代码在调用者(Caller)的上下文中执行，修改 Caller 的 Storage，但使用 Target 的逻辑。常用于代理模式。

#### 1.3 Receive vs Fallback
**Q: 什么时候触发哪个？**
A:
-   `receive()`: `msg.data` 为空且附带了 ETH。
-   `fallback()`: `msg.data` 不为空（即找不到匹配函数），或者 `msg.data` 为空但没有定义 `receive()`。

#### 1.4 继承与 Shadowing
**Q: Solidity 支持多重继承吗？C3 Linearization 是什么？**
A: 支持。C3 线性化算法用于确定父合约的初始化顺序，遵循从右到左的深度优先搜索（在 `is` 关键字中是从右到左最"基"类）。

### 2. EVM 深度原理

#### 2.1 数据存储区域
**Q: 解释 Storage, Memory, Stack, Calldata 的区别？**
-   **Storage**: 永久存储，最贵 (SSTORE = 20k/2.9k gas)。
-   **Memory**: 临时存储，函数执行完清除，线性扩容成本。
-   **Stack**: EVM 计算栈，最大 1024 深度，访问最快。
-   **Calldata**: 交易输入数据，只读，比 Memory 便宜。

#### 2.2 Gas 计算
**Q: 为什么 `SSTORE` 有时候是 20000 gas，有时候是 2900 gas？**
A:
-   **20k**: 将 0 变为非 0 值 (初始化)。
-   **2.9k**: 将非 0 变为非 0 值 (修改)。
-   **退税**: 将非 0 变为 0 (清除) 会获得 Gas 退税。

#### 2.3 Opcodes
**Q: `EXTCODESIZE` 的作用？**
A: 检查地址是否包含代码（判断是否为合约）。但在 `constructor` 执行期间，该值返回 0。

### 3. Gas 优化技巧

1.  **变量打包 (Packing)**: 将多个小变量 (`uint64`, `address`) 放在一起以共享 Slot。
2.  **Unchecked**: 对于确定不会溢出的数学运算（如循环变量 `i++`），使用 `unchecked { i++ }`。
3.  **Calldata**: 外部函数的数组参数尽量用 `calldata` 而不是 `memory`。
4.  **Short Circuit**: `require(cond1 && cond2)` 中，将开销小的条件放在前面。
5.  **Immutable & Constant**: 编译时确定的值，直接替换进字节码，不读 Storage。
6.  **Caching**: 在循环中频繁读取 Storage 变量时，先缓存到 Stack (Memory) 变量中。

### 4. 白板编程 (Whiteboard Coding)

面试官可能会给你一个在线 IDE (如 CoderPad) 或让你共享屏幕写代码。

#### 挑战 1: 手写 ERC20 `transfer`
```solidity
// 关键点: 检查余额，更新余额，发事件，防溢出(0.8自带)，处理0值(可选)
function transfer(address to, uint256 amount) public returns (bool) {
    require(balanceOf[msg.sender] >= amount, "Insufficient balance");
    
    balanceOf[msg.sender] -= amount;
    balanceOf[to] += amount;
    
    emit Transfer(msg.sender, to, amount);
    return true;
}
```

#### 挑战 2: 实现 ReentrancyGuard
```solidity
// 关键点: 使用状态变量锁定
uint256 private constant _NOT_ENTERED = 1;
uint256 private constant _ENTERED = 2;
uint256 private _status;

modifier nonReentrant() {
    require(_status != _ENTERED, "ReentrancyGuard: reentrant call");
    _status = _ENTERED;
    _;
    _status = _NOT_ENTERED;
}
```

#### 挑战 3: 验证签名
```solidity
// 关键点: ecrecover, v r s 分割, 消息哈希前缀
function verify(bytes32 hash, uint8 v, bytes32 r, bytes32 s, address signer) public pure returns (bool) {
    // 加上以太坊前缀
    bytes32 prefixedHash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hash));
    return ecrecover(prefixedHash, v, r, s) == signer;
}
```

## 实践练习

### 练习1: Gas Golfing (Gas 高尔夫)
优化以下代码，使其 Gas 消耗降到最低：
```solidity
// 优化前
uint256[] public arr;
function sum() public view returns (uint256) {
    uint256 total = 0;
    for (uint256 i = 0; i < arr.length; i++) {
        total += arr[i];
    }
    return total;
}
```
**优化点**: 缓存 `arr.length`, `unchecked i++`, `calldata` (如果传参), 甚至汇编。

### 练习2: 模拟面试
找一个朋友或对着镜子，口述 "DelegateCall 漏洞是如何导致 Parity 钱包被黑的？"。要求逻辑清晰，术语准确。

## 学习检查清单

- [ ] 能解释 Solidity 存储槽的打包规则
- [ ] 理解 EVM 内存管理的生命周期
- [ ] 至少掌握 5 种以上的 Gas 优化手段
- [ ] 能徒手写出标准的 ERC20 核心逻辑
- [ ] 熟悉常见的 Opcode (SSTORE, SLOAD, CALL, REVERT)

## 扩展阅读

-   [Solidity Gas Optimization Tricks](https://github.com/iskd/solidity-gas-optimization)
-   [EVM Opcodes Reference](https://www.evm.codes/)
-   [DeFi Developer Roadmap - Interview Questions](https://github.com/OffcierCia/DeFi-Developer-Road-Map)

## 下节预告

下一节我们将进入**技术面试准备 II**,包括:
- 系统设计 (System Design): 如何设计一个 DEX / 借贷协议 / NFT 市场
- 安全审计面试题: 现场找漏洞
- Web3 常见算法: Merkle Tree, 签名算法
- 行为面试 (Behavioral Questions)
