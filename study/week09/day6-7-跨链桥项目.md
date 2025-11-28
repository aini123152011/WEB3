# Week 9 - Day 6-7: 跨链桥项目

**学习日期**: ___________
**预计用时**: 12-14小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 项目目标

构建一个通用的代币跨链桥 (Token Bridge)：
- ✅ 支持 Lock & Mint 模式
- ✅ 兼容 Ethereum (L1) 和 Arbitrum (L2)
- ✅ 兼容 Goerli 和 Sepolia (通过LayerZero)
- ✅ 前端集成多链钱包和交易追踪
- ✅ 完整的安全防护（限额、暂停）

---

## Part 1: 合约架构设计 (2小时)

### 1.1 核心接口

```solidity
// interfaces/IBridge.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IBridge {
    event Bridged(
        address indexed sender,
        address indexed recipient,
        uint256 amount,
        uint256 destinationChainId,
        uint256 nonce
    );

    event Claimed(
        address indexed recipient,
        uint256 amount,
        uint256 sourceChainId,
        uint256 nonce
    );

    function bridge(
        uint256 destinationChainId,
        address recipient,
        uint256 amount
    ) external payable;
}
```

### 1.2 LayerZero Base

```solidity
// contracts/LzBridge.sol
import "@layerzerolabs/contracts/lzApp/NonblockingLzApp.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

abstract contract LzBridge is NonblockingLzApp, ReentrancyGuard {
    IERC20 public immutable token;
    uint256 public immutable localChainId;

    constructor(
        address _endpoint, 
        address _token, 
        uint256 _localChainId
    ) NonblockingLzApp(_endpoint) {
        token = IERC20(_token);
        localChainId = _localChainId;
    }

    function _bridge(
        uint16 _dstChainId,
        address _recipient,
        uint256 _amount
    ) internal {
        bytes memory payload = abi.encode(_recipient, _amount);
        
        _lzSend(
            _dstChainId,
            payload,
            payable(msg.sender),
            address(0x0),
            bytes("")
        );
    }

    function _nonblockingLzReceive(
        uint16 _srcChainId,
        bytes memory _srcAddress,
        uint64 _nonce,
        bytes memory _payload
    ) internal virtual override {
        (address recipient, uint256 amount) = abi.decode(_payload, (address, uint256));
        _unlockOrMint(recipient, amount);
    }

    function _unlockOrMint(address recipient, uint256 amount) internal virtual;
}
```

---

## Part 2: 核心合约实现 (3小时)

### 2.1 源链合约 (Lock)

```solidity
// contracts/SourceBridge.sol
import "./LzBridge.sol";

contract SourceBridge is LzBridge {
    constructor(
        address _endpoint, 
        address _token, 
        uint256 _localChainId
    ) LzBridge(_endpoint, _token, _localChainId) {}

    function bridge(
        uint16 _dstChainId,
        address _recipient,
        uint256 _amount
    ) external payable nonReentrant {
        // Lock tokens
        token.transferFrom(msg.sender, address(this), _amount);
        
        // Send message
        _bridge(_dstChainId, _recipient, _amount);
        
        emit Bridged(msg.sender, _recipient, _amount, _dstChainId, 0);
    }

    function _unlockOrMint(address recipient, uint256 amount) internal override {
        // Unlock tokens (when bridging back)
        token.transfer(recipient, amount);
        emit Claimed(recipient, amount, 0, 0);
    }
}
```

### 2.2 目标链合约 (Mint)

```solidity
// contracts/DestBridge.sol
import "./LzBridge.sol";

interface IMintableToken {
    function mint(address to, uint256 amount) external;
    function burn(uint256 amount) external;
}

contract DestBridge is LzBridge {
    constructor(
        address _endpoint, 
        address _token, 
        uint256 _localChainId
    ) LzBridge(_endpoint, _token, _localChainId) {}

    function bridge(
        uint16 _dstChainId,
        address _recipient,
        uint256 _amount
    ) external payable nonReentrant {
        // Burn tokens
        IMintableToken(address(token)).burnFrom(msg.sender, _amount);
        
        // Send message
        _bridge(_dstChainId, _recipient, _amount);
        
        emit Bridged(msg.sender, _recipient, _amount, _dstChainId, 0);
    }

    function _unlockOrMint(address recipient, uint256 amount) internal override {
        // Mint tokens
        IMintableToken(address(token)).mint(recipient, amount);
        emit Claimed(recipient, amount, 0, 0);
    }
}
```

---

## Part 3: 安全增强 (2小时)

### 3.1 速率限制

```solidity
// contracts/extensions/RateLimiter.sol
contract RateLimiter {
    uint256 public constant MAX_DAILY_LIMIT = 100000 * 10**18;
    uint256 public currentDailyAmount;
    uint256 public lastResetTimestamp;

    modifier rateLimit(uint256 amount) {
        if (block.timestamp >= lastResetTimestamp + 1 days) {
            currentDailyAmount = 0;
            lastResetTimestamp = block.timestamp;
        }
        
        currentDailyAmount += amount;
        require(currentDailyAmount <= MAX_DAILY_LIMIT, "Rate limit exceeded");
        _;
    }
}
```

### 3.2 紧急暂停

```solidity
// contracts/extensions/PausableBridge.sol
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract PausableBridge is Pausable, Ownable {
    function pause() external onlyOwner {
        _pause();
    }

    function unpause() external onlyOwner {
        _unpause();
    }

    modifier whenNotPausedMod() {
        require(!paused(), "Bridge paused");
        _;
    }
}
```

---

## Part 4: 前端集成 (3小时)

### 4.1 跨链费用估算

```javascript
// hooks/useBridgeFees.js
import { useState, useEffect } from 'react';
import { ethers } from 'ethers';

export function useBridgeFees(bridgeContract, dstChainId, recipient, amount) {
    const [fees, setFees] = useState('0');

    useEffect(() => {
        if (!bridgeContract || !dstChainId || !amount) return;

        const estimate = async () => {
            const payload = ethers.AbiCoder.defaultAbiCoder().encode(
                ['address', 'uint256'],
                [recipient || ethers.ZeroAddress, amount]
            );
            
            // LayerZero estimateFees
            const [nativeFee] = await bridgeContract.estimateFees(
                dstChainId,
                address(this), // destination contract
                payload,
                false,
                "0x"
            );
            
            setFees(nativeFee);
        };

        estimate();
    }, [bridgeContract, dstChainId, amount]);

    return fees;
}
```

### 4.2 桥接表单组件

```javascript
// components/BridgeForm.jsx
import { useState } from 'react';
import { useBridgeFees } from '../hooks/useBridgeFees';

export default function BridgeForm({ srcChain, dstChain }) {
    const [amount, setAmount] = useState('');
    const fees = useBridgeFees(contract, dstChain.id, account, amount);

    const handleBridge = async () => {
        try {
            // 1. Approve
            await token.approve(bridgeAddress, amount);
            
            // 2. Bridge
            await bridge.bridge(
                dstChain.lzId,
                account,
                ethers.parseEther(amount),
                { value: fees }
            );
        } catch (err) {
            console.error(err);
        }
    };

    return (
        <div className="p-4 bg-white rounded-lg shadow">
            <input 
                value={amount}
                onChange={e => setAmount(e.target.value)}
                placeholder="Amount"
                className="border p-2 rounded w-full mb-4"
            />
            <div className="text-sm text-gray-500 mb-4">
                Estimated Fees: {ethers.formatEther(fees)} ETH
            </div>
            <button 
                onClick={handleBridge}
                className="bg-blue-600 text-white w-full py-2 rounded"
            >
                Bridge Assets
            </button>
        </div>
    );
}
```

---

## Part 5: 测试与部署 (2小时)

### 5.1 测试脚本

```javascript
// test/Bridge.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("CrossChainBridge", function () {
    it("Should lock tokens on source chain", async function () {
        const [owner] = await ethers.getSigners();
        // ... deployment setup ...
        
        await token.approve(sourceBridge.address, 100);
        
        await expect(sourceBridge.bridge(1, owner.address, 100))
            .to.emit(sourceBridge, "Bridged")
            .withArgs(owner.address, owner.address, 100, 1, 0);
            
        expect(await token.balanceOf(sourceBridge.address)).to.equal(100);
    });
});
```

### 5.2 LayerZero 配置

```javascript
// scripts/configureLayerZero.js
async function main() {
    const sourceBridge = await ethers.getContractAt("SourceBridge", SRC_ADDR);
    const destBridge = await ethers.getContractAt("DestBridge", DST_ADDR);

    // Set Trusted Remote
    // 远程合约地址需要是 packed bytes (remote + local)
    const remoteAndLocal = ethers.solidityPacked(
        ["address", "address"],
        [DST_ADDR, SRC_ADDR]
    );
    
    await sourceBridge.setTrustedRemote(DST_LZ_ID, remoteAndLocal);
}
```

---

## 📝 项目总结

### 核心功能
1. **资产锁定/释放**: 在源链安全锁定资产。
2. **资产铸造/销毁**: 在目标链对应铸造/销毁。
3. **消息传递**: 使用LayerZero作为底层通信设施。
4. **安全防护**: 包含速率限制和紧急暂停开关。

### 扩展方向
- 支持更多链（Solana, Cosmos等）
- 集成流动性池模式（用于原生资产跨链）
- 增加中继器监控和自动重试机制

---

## ✅ 检查清单

- [ ] 合约部署到 Goerli 和 Sepolia
- [ ] LayerZero Trusted Remote 配置正确
- [ ] 前端能正确估算跨链费用
- [ ] 成功完成一次跨链转账
- [ ] 测试速率限制功能

---

## 📅 下周预告

下周进入 CI/CD 与部署：
- GitHub Actions 自动化测试
- 智能合约自动化部署
- 前端自动化部署 (Vercel)
- 监控与报警系统

**🎉 恭喜完成Week 9！你已经打通了多链宇宙！**
