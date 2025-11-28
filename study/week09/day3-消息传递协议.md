# Week 9 - Day 3: 消息传递协议

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 学习目标

- ✅ 设计通用跨链消息格式
- ✅ 实现跨链NFT传输 (ONFT)
- ✅ 掌握跨链治理投票
- ✅ 处理跨链异常与重试
- ✅ 优化跨链Gas消耗

---

## Part 1: 结构化消息设计 (1.5小时)

### 1.1 为什么需要标准格式？

不同链的编码方式可能不同（如EVM vs Solana），需要统一的消息结构。

### 1.2 消息协议设计

```solidity
// IMessageProtocol.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IMessageProtocol {
    enum MessageType {
        TOKEN_TRANSFER,
        NFT_TRANSFER,
        GOVERNANCE_VOTE,
        CUSTOM_CALL
    }

    struct CrossChainMessage {
        uint16 srcChainId;
        address srcAddress;
        uint64 nonce;
        MessageType msgType;
        bytes payload;
    }
}
```

### 1.3 消息编解码

```solidity
// MessageLib.sol
library MessageLib {
    function encodeTokenTransfer(
        address token,
        address recipient,
        uint256 amount
    ) internal pure returns (bytes memory) {
        return abi.encode(token, recipient, amount);
    }

    function decodeTokenTransfer(bytes memory payload) 
        internal 
        pure 
        returns (address token, address recipient, uint256 amount) 
    {
        return abi.decode(payload, (address, address, uint256));
    }
}
```

---

## Part 2: 跨链NFT传输 (ONFT) (2小时)

使用LayerZero的ONFT标准（Omnichain Non-Fungible Token）。

### 2.1 ONFT标准接口

```solidity
// IONFT721.sol
interface IONFT721 {
    function sendFrom(
        address _from,
        uint16 _dstChainId,
        bytes calldata _toAddress,
        uint256 _tokenId,
        address payable _refundAddress,
        address _zroPaymentAddress,
        bytes calldata _adapterParams
    ) external payable;
}
```

### 2.2 ONFT实现

```solidity
// MyONFT.sol
import "@layerzerolabs/contracts/token/onft/ONFT721.sol";

contract MyONFT is ONFT721 {
    constructor(
        string memory _name,
        string memory _symbol,
        address _lzEndpoint
    ) ONFT721(_name, _symbol, _lzEndpoint) {}

    // 跨链发送逻辑（由父类处理）
    // 源链：Burn
    // 目标链：Mint
    
    // 估算费用
    function estimateSendFee(
        uint16 _dstChainId,
        bytes memory _toAddress,
        uint256 _tokenId,
        bool _useZro,
        bytes memory _adapterParams
    ) public view override returns (uint256 nativeFee, uint256 zroFee) {
        return super.estimateSendFee(
            _dstChainId, 
            _toAddress, 
            _tokenId, 
            _useZro, 
            _adapterParams
        );
    }
}
```

---

## Part 3: 跨链治理投票 (1.5小时)

### 3.1 场景描述

DAO在以太坊主网，但用户持有Token在Polygon上。需要将Polygon上的投票权重同步到以太坊。

### 3.2 投票聚合器

```solidity
// CrossChainGovernor.sol
contract CrossChainGovernor is NonblockingLzApp {
    struct Proposal {
        uint256 id;
        uint256 forVotes;
        uint256 againstVotes;
    }
    
    mapping(uint256 => Proposal) public proposals;

    // 接收来自其他链的投票结果
    function _nonblockingLzReceive(
        uint16 _srcChainId,
        bytes memory _srcAddress,
        uint64 _nonce,
        bytes memory _payload
    ) internal override {
        (uint256 proposalId, uint256 forVotes, uint256 againstVotes) = 
            abi.decode(_payload, (uint256, uint256, uint256));
            
        proposals[proposalId].forVotes += forVotes;
        proposals[proposalId].againstVotes += againstVotes;
        
        emit VoteReceived(_srcChainId, proposalId, forVotes, againstVotes);
    }
}
```

### 3.3 侧链投票合约

```solidity
// SideChainVoting.sol
contract SideChainVoting is NonblockingLzApp {
    function castVoteAndSync(
        uint256 proposalId, 
        bool support, 
        uint16 mainChainId
    ) external payable {
        // 1. 本地计票
        uint256 weight = token.balanceOf(msg.sender);
        // ... record vote ...

        // 2. 发送结果到主链
        bytes memory payload = abi.encode(
            proposalId, 
            support ? weight : 0, 
            support ? 0 : weight
        );
        
        _lzSend(
            mainChainId,
            payload,
            payable(msg.sender),
            address(0x0),
            bytes("")
        );
    }
}
```

---

## Part 4: 错误处理与重试 (1小时)

跨链消息可能因为Gas不足或逻辑错误而失败。

### 4.1 存储失败消息

```solidity
// ErrorHandler.sol
contract ErrorHandler is NonblockingLzApp {
    mapping(bytes32 => bytes) public failedMessages;

    event MessageFailed(uint16 srcChainId, bytes srcAddress, uint64 nonce, bytes payload);

    // 覆盖默认的失败处理
    function nonblockingLzReceive(
        uint16 _srcChainId,
        bytes calldata _srcAddress,
        uint64 _nonce,
        bytes calldata _payload
    ) public override {
        // 尝试执行
        try this.processMessage(_srcChainId, _srcAddress, _nonce, _payload) {
            // Success
        } catch {
            // Failure: Store message hash
            bytes32 hash = keccak256(abi.encode(_srcChainId, _srcAddress, _nonce, _payload));
            failedMessages[hash] = _payload;
            emit MessageFailed(_srcChainId, _srcAddress, _nonce, _payload);
        }
    }

    // 手动重试
    function retryMessage(
        uint16 _srcChainId,
        bytes calldata _srcAddress,
        uint64 _nonce,
        bytes calldata _payload
    ) external payable {
        bytes32 hash = keccak256(abi.encode(_srcChainId, _srcAddress, _nonce, _payload));
        require(failedMessages[hash].length > 0, "Message not failed");
        
        // Clear error first (prevent reentrancy)
        delete failedMessages[hash];
        
        // Process
        processMessage(_srcChainId, _srcAddress, _nonce, _payload);
    }
}
```

---

## Part 5: Gas优化 (1小时)

### 5.1 批量发送

将多条消息打包成一条跨链消息发送。

```solidity
function batchSend(
    uint16 _dstChainId, 
    bytes[] calldata _payloads
) external payable {
    bytes memory batchPayload = abi.encode(_payloads);
    _lzSend(_dstChainId, batchPayload, ...);
}
```

### 5.2 参数配置

使用 `adapterParams` 调整目标链Gas Limit。

```javascript
// version: 1
// gasLimit: 200000
const adapterParams = ethers.solidityPacked(
    ["uint16", "uint256"], 
    [1, 200000]
);
```

---

## 📝 今日作业

### 作业1: 开发ONFT

1. 使用LayerZero实现一个ERC721跨链合约
2. 部署到Goerli和Sepolia
3. 在Goerli铸造NFT并跨链发送到Sepolia
4. 验证元数据URI是否保持一致

### 作业2: 实现跨链投票系统

1. 编写主链Governor合约
2. 编写侧链Voting合约
3. 实现投票权重同步逻辑
4. 编写测试脚本模拟跨链投票

### 作业3: 异常处理机制

完善跨链桥的异常处理：
1. 模拟目标链合约revert情况
2. 验证failedMessages是否正确存储
3. 实现前端手动重试界面

---

## ✅ 检查清单

- [ ] 理解ONFT的Burn/Mint机制
- [ ] 掌握跨链消息的编解码
- [ ] 能够处理跨链交易失败
- [ ] 知道如何调整adapterParams
- [ ] 理解跨链治理的复杂性

---

## 📅 明日预告

明天学习跨链安全性：
- 验证者攻击与防御
- 重放攻击防护
- 预言机操纵风险
- 紧急暂停机制

**🎉 完成Day 3！跨链通信专家！**
