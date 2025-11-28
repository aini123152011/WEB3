# Week 9 - Day 1: 跨链原理

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 学习目标

- ✅ 理解跨链通信原理
- ✅ 掌握Hash Time Locked Contracts (HTLC)
- ✅ 理解Notary Schemes (公证人机制)
- ✅ 学习Relays (中继) 机制
- ✅ 对比各种跨链方案

---

## Part 1: 跨链基础 (1.5小时)

### 1.1 为什么需要跨链？

区块链是封闭的系统，无法直接获取外部数据或与其他链通信。跨链技术旨在打破这种孤岛效应。

**主要场景**：
- **资产跨链**：将BTC转移到以太坊 (WBTC)。
- **跨链借贷**：在A链抵押，在B链借款。
- **跨链治理**：在主链投票决定侧链参数。

### 1.2 跨链互操作性难点 (Trilemma)

Vitalik提出的互操作性不可能三角：
1. **Trustless (去信任)**: 不依赖第三方。
2. **Extensible (可扩展)**: 支持任何区块链。
3. **Generalizable (通用性)**: 支持任意数据传输。

---

## Part 2: 哈希时间锁定合约 (HTLC) (2小时)

HTLC是最早的原子交换(Atomic Swap)技术，不需要可信第三方。

### 2.1 原理

1. Alice生成随机数 `s`，计算哈希 `h = hash(s)`。
2. Alice在链A部署合约，锁定资金，条件是提供 `s` 或超时退款。
3. Bob确认链A合约后，在链B部署类似合约，使用相同的 `h`。
4. Alice在链B提供 `s` 取走资金。
5. Bob从链B看到 `s`，在链A使用 `s` 取走资金。

### 2.2 合约实现

```solidity
// HTLC.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract HTLC {
    struct Swap {
        address sender;
        address recipient;
        bytes32 secretHash;
        uint256 amount;
        uint256 timelock;
        bool claimed;
        bool refunded;
    }

    mapping(bytes32 => Swap) public swaps;

    event SwapCreated(bytes32 indexed id, address sender, address recipient, uint256 amount, bytes32 secretHash, uint256 timelock);
    event SwapClaimed(bytes32 indexed id, string secret);
    event SwapRefunded(bytes32 indexed id);

    function createSwap(
        bytes32 _id,
        address _recipient,
        bytes32 _secretHash,
        uint256 _timelock
    ) external payable {
        require(msg.value > 0, "No funds");
        require(swaps[_id].amount == 0, "Swap exists");
        require(_timelock > block.timestamp, "Invalid timelock");

        swaps[_id] = Swap({
            sender: msg.sender,
            recipient: _recipient,
            secretHash: _secretHash,
            amount: msg.value,
            timelock: _timelock,
            claimed: false,
            refunded: false
        });

        emit SwapCreated(_id, msg.sender, _recipient, msg.value, _secretHash, _timelock);
    }

    function claim(bytes32 _id, string calldata _secret) external {
        Swap storage swap = swaps[_id];
        require(swap.amount > 0, "Swap not found");
        require(!swap.claimed, "Already claimed");
        require(!swap.refunded, "Already refunded");
        require(keccak256(abi.encodePacked(_secret)) == swap.secretHash, "Invalid secret");

        swap.claimed = true;
        payable(swap.recipient).transfer(swap.amount);

        emit SwapClaimed(_id, _secret);
    }

    function refund(bytes32 _id) external {
        Swap storage swap = swaps[_id];
        require(swap.amount > 0, "Swap not found");
        require(!swap.claimed, "Already claimed");
        require(!swap.refunded, "Already refunded");
        require(block.timestamp >= swap.timelock, "Timelock not expired");

        swap.refunded = true;
        payable(swap.sender).transfer(swap.amount);

        emit SwapRefunded(_id);
    }
}
```

---

## Part 3: 公证人机制 (Notary Schemes) (1.5小时)

公证人机制依赖一组受信任的节点（多签验证者）来验证跨链事件。

### 3.1 工作流程

1. 用户在源链锁定资产。
2. 公证人（Relayers）监听源链事件。
3. 公证人在目标链签名确认。
4. 目标链合约验证签名并铸造资产。

### 3.2 简单多签桥实现

```solidity
// SimpleBridge.sol
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract SimpleBridge {
    using ECDSA for bytes32;

    address public validator;
    mapping(bytes32 => bool) public processedNonces;

    event Deposit(address indexed sender, uint256 amount, uint256 nonce);
    event Withdraw(address indexed recipient, uint256 amount, uint256 nonce);

    constructor(address _validator) {
        validator = _validator;
    }

    function deposit(uint256 nonce) external payable {
        require(!processedNonces[bytes32(nonce)], "Nonce used");
        processedNonces[bytes32(nonce)] = true;
        
        emit Deposit(msg.sender, msg.value, nonce);
    }

    function withdraw(
        uint256 amount,
        uint256 nonce,
        bytes calldata signature
    ) external {
        require(!processedNonces[bytes32(nonce)], "Nonce used");
        
        bytes32 message = keccak256(abi.encodePacked(msg.sender, amount, nonce));
        bytes32 ethSignedMessageHash = message.toEthSignedMessageHash();
        
        require(ethSignedMessageHash.recover(signature) == validator, "Invalid sig");
        
        processedNonces[bytes32(nonce)] = true;
        payable(msg.sender).transfer(amount);
        
        emit Withdraw(msg.sender, amount, nonce);
    }
}
```

---

## Part 4: 中继与轻客户端 (Relays) (1小时)

最安全的跨链方式，目标链运行源链的轻客户端（Light Client），验证区块头和Merkle Proof。

### 4.1 优缺点

- **优点**：去信任，安全性等同于底层链。
- **缺点**：Gas费用极高（特别是验证PoW链）。

### 4.2 Optimistic Bridge (乐观验证)

类似Optimistic Rollup，假设交易有效，给出一个挑战期（Challenge Period）。如果有人提交欺诈证明，则回滚交易并惩罚作恶者。

---

## Part 5: 现代跨链协议 (1小时)

### 5.1 LayerZero

LayerZero是一个全链互操作性协议，连接任何两条链。

**核心组件**：
- **Oracle**: 传递区块头。
- **Relayer**: 传递交易证明。
- **Endpoint**: 链上通信接口。

### 5.2 Axelar

提供跨链网关协议（CGP）和跨链传输协议（CTP）。

---

## 📝 今日作业

### 作业1: 实现HTLC原子交换脚本

编写脚本模拟两个不同网络（如Goerli和Sepolia）之间的原子交换：
1. 生成Secret
2. 在Chain A部署HTLC并锁定资金
3. 在Chain B部署HTLC并锁定资金
4. 完成交换流程

### 作业2: 开发多签桥后端

编写一个Node.js服务作为Validator：
1. 监听Chain A的Deposit事件
2. 生成签名
3. 提供API供前端获取签名并在Chain B提款

### 作业3: 跨链调研报告

对比 LayerZero, Multichain (Anyswap), Wormhole, Axelar：
1. 安全模型
2. 信任假设
3. 费用结构
4. 历史安全事故分析

---

## ✅ 检查清单

- [ ] 理解HTLC的工作原理和局限性
- [ ] 掌握ECDSA签名验证
- [ ] 理解公证人机制的风险
- [ ] 了解轻客户端验证原理
- [ ] 熟悉主流跨链协议架构

---

## 📅 明日预告

明天学习跨链桥架构实战：
- Lock & Mint 模型
- Burn & Redeem 模型
- 流动性池模型 (Liquidity Pool)
- 消息传递接口设计

**🎉 完成Day 1！你已经迈出了跨链世界的第一步！**
