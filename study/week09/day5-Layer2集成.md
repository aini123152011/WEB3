# Week 9 - Day 5: Layer2集成

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 理解Optimistic Rollup与ZK Rollup
- ✅ 掌握Arbitrum消息传递
- ✅ 掌握Optimism消息传递
- ✅ 实现L1 <-> L2资产桥接
- ✅ 了解Polygon zkEVM集成

---

## Part 1: Rollup基础 (1.5小时)

### 1.1 Layer2 扩展方案

- **Optimistic Rollup (Arbitrum, Optimism)**: 假设交易有效，通过欺诈证明(Fraud Proof)挑战无效交易。提款需要7天挑战期。
- **ZK Rollup (zkSync, StarkNet)**: 生成零知识证明(Validity Proof)验证交易有效性。安全性更高，提款更快，但计算成本高。

### 1.2 消息传递生命周期

1. **L1 -> L2 (Deposit)**: L1合约发起，几分钟内在L2执行。
2. **L2 -> L1 (Withdraw)**: L2发起，需等待挑战期(Op)或证明生成(ZK)，然后由Relayer在L1执行。

---

## Part 2: Arbitrum消息传递 (1.5小时)

### 2.1 L1 -> L2 (Inbox)

```solidity
// L1Sender.sol
import "@arbitrum/nitro-contracts/src/bridge/IInbox.sol";

contract L1Sender {
    IInbox public inbox;

    constructor(address _inbox) {
        inbox = IInbox(_inbox);
    }

    function sendToL2(
        address l2Target,
        bytes memory data,
        uint256 maxSubmissionCost,
        uint256 maxGas,
        uint256 gasPriceBid
    ) public payable {
        inbox.createRetryableTicket{value: msg.value}(
            l2Target,
            0, // l2CallValue
            maxSubmissionCost,
            msg.sender, // excessFeeRefundAddress
            msg.sender, // callValueRefundAddress
            maxGas,
            gasPriceBid,
            data
        );
    }
}
```

### 2.2 L2 -> L1 (Outbox)

```solidity
// L2Sender.sol
import "@arbitrum/nitro-contracts/src/precompiles/ArbSys.sol";

contract L2Sender {
    ArbSys public constant ARB_SYS = ArbSys(address(100));

    function sendToL1(address l1Target, bytes memory data) public payable {
        ARB_SYS.sendTxToL1{value: msg.value}(
            l1Target,
            data
        );
    }
}
```

---

## Part 3: Optimism消息传递 (1.5小时)

### 3.1 L1 -> L2 (CrossDomainMessenger)

```solidity
// L1Sender.sol
import "@eth-optimism/contracts/libraries/bridge/ICrossDomainMessenger.sol";

contract L1Sender {
    ICrossDomainMessenger public messenger;

    constructor(address _messenger) {
        messenger = ICrossDomainMessenger(_messenger);
    }

    function sendToL2(address l2Target, bytes memory message, uint32 gasLimit) public {
        messenger.sendMessage(
            l2Target,
            message,
            gasLimit
        );
    }
}
```

### 3.2 L2 -> L1

```solidity
// L2Sender.sol
import "@eth-optimism/contracts/libraries/bridge/ICrossDomainMessenger.sol";

contract L2Sender {
    ICrossDomainMessenger public messenger;

    constructor(address _messenger) {
        messenger = ICrossDomainMessenger(_messenger); // L2 CrossDomainMessenger
    }

    function sendToL1(address l1Target, bytes memory message, uint32 gasLimit) public {
        messenger.sendMessage(
            l1Target,
            message,
            gasLimit
        );
    }
}
```

---

## Part 4: 资产桥接 (1.5小时)

### 4.1 标准桥接 (Standard Bridge)

L2官方通常提供标准桥接合约，支持ETH和ERC20。

```javascript
// 使用Optimism SDK进行桥接
const crossChainMessenger = new CrossChainMessenger({
  l1ChainId: 1,
  l2ChainId: 10,
  l1SignerOrProvider: l1Signer,
  l2SignerOrProvider: l2Signer,
});

// 存款 ETH
const tx = await crossChainMessenger.depositETH(ethers.utils.parseEther('1.0'));
await tx.wait();

// 存款 ERC20
const tx2 = await crossChainMessenger.depositERC20(
  l1TokenAddress,
  l2TokenAddress,
  ethers.utils.parseEther('100')
);
await tx2.wait();
```

### 4.2 自定义桥接

如果需要自定义逻辑（如由L2铸造新代币），可以基于消息传递构建。

```solidity
// L1TokenBridge.sol
function deposit(address to, uint256 amount) external {
    token.transferFrom(msg.sender, address(this), amount);
    
    bytes memory message = abi.encodeWithSignature(
        "mint(address,uint256)", 
        to, 
        amount
    );
    
    messenger.sendMessage(l2TokenBridge, message, 1000000);
}

// L2TokenBridge.sol
function mint(address to, uint256 amount) external {
    require(msg.sender == address(messenger), "Only messenger");
    require(messenger.xDomainMessageSender() == l1TokenBridge, "Only L1 bridge");
    
    l2Token.mint(to, amount);
}
```

---

## 📝 今日作业

### 作业1: Arbitrum Greeter

1. 在Goerli (L1) 部署 `GreeterL1`
2. 在Arbitrum Goerli (L2) 部署 `GreeterL2`
3. 编写脚本，从L1发送消息更新L2的问候语
4. 验证L2状态是否更新

### 作业2: Optimism 桥接

1. 使用 `optimism-sdk` 编写脚本
2. 将Goerli ETH 存入 Optimism Goerli
3. 查询存款状态和到账时间

### 作业3: 提款挑战期模拟

编写一个测试，模拟Optimistic Rollup的提款流程：
1. L2发起提款
2. 模拟等待挑战期（在测试网通常较短）
3. L1执行 `finalizeWithdrawal`
4. 验证资金是否到账

---

## ✅ 检查清单

- [ ] 理解Op Rollup与ZK Rollup的区别
- [ ] 掌握Arbitrum的Retryable Ticket机制
- [ ] 能够使用Optimism SDK进行资产跨链
- [ ] 理解L1/L2消息传递的Gas支付方式
- [ ] 知道如何查询跨链交易状态

---

## 📅 周末预告

周末进行跨链桥综合项目：
- 构建一个通用代币桥
- 支持 ETH <-> Arbitrum <-> Optimism
- 前端集成多链钱包
- 实时追踪跨链进度

**🎉 完成Day 5！L2扩容方案已掌握！**
