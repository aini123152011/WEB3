# Week 9 - Day 2: 跨链桥架构

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 学习目标

- ✅ 深入Lock & Mint模型
- ✅ 掌握Burn & Redeem模型
- ✅ 学习Liquidity Pool跨链
- ✅ 了解LayerZero技术栈
- ✅ 设计消息传递接口

---

## Part 1: Lock & Mint 模型 (1.5小时)

这是最常见的资产跨链模式，适用于将原生资产（如BTC）跨到其他链（如ETH上的WBTC）。

### 1.1 工作流程

1. **源链 (Source)**: 用户将资产锁定在智能合约中。
2. **中继 (Relayer)**: 监听到锁定事件，生成证明。
3. **目标链 (Dest)**: 验证证明，铸造等量的包装资产 (Wrapped Token)。

### 1.2 合约实现

**源链锁定合约**：

```solidity
// SourceBridge.sol
contract SourceBridge {
    IERC20 public token;
    uint256 public nonce;

    event Locked(address indexed sender, address indexed recipient, uint256 amount, uint256 nonce);

    constructor(address _token) {
        token = IERC20(_token);
    }

    function lock(address recipient, uint256 amount) external {
        require(amount > 0, "Amount > 0");
        token.transferFrom(msg.sender, address(this), amount);
        
        emit Locked(msg.sender, recipient, amount, nonce++);
    }
}
```

**目标链铸造合约**：

```solidity
// DestBridge.sol
import "@openzeppelin/contracts/access/AccessControl.sol";

interface IMintableToken {
    function mint(address to, uint256 amount) external;
}

contract DestBridge is AccessControl {
    bytes32 public constant RELAYER_ROLE = keccak256("RELAYER_ROLE");
    IMintableToken public wrappedToken;
    mapping(uint256 => bool) public processedNonces;

    event Minted(address indexed recipient, uint256 amount, uint256 nonce);

    constructor(address _token) {
        wrappedToken = IMintableToken(_token);
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

    function mint(
        address recipient, 
        uint256 amount, 
        uint256 nonce
    ) external onlyRole(RELAYER_ROLE) {
        require(!processedNonces[nonce], "Nonce processed");
        processedNonces[nonce] = true;
        
        wrappedToken.mint(recipient, amount);
        emit Minted(recipient, amount, nonce);
    }
}
```

---

## Part 2: Burn & Redeem 模型 (1.5小时)

当用户想将包装资产跨回源链时，使用此模型。

### 2.1 工作流程

1. **目标链 (Dest)**: 用户销毁 (Burn) 包装资产。
2. **中继 (Relayer)**: 监听到销毁事件。
3. **源链 (Source)**: 验证后，释放 (Unlock) 等量的原生资产。

### 2.2 合约实现

**目标链销毁合约**：

```solidity
// DestBridge.sol (Addon)
interface IBurnableToken {
    function burn(uint256 amount) external;
}

function burn(uint256 amount) external {
    IBurnableToken(address(wrappedToken)).burnFrom(msg.sender, amount);
    emit Burned(msg.sender, amount, nonce++);
}
```

**源链解锁合约**：

```solidity
// SourceBridge.sol (Addon)
function unlock(
    address recipient, 
    uint256 amount, 
    uint256 nonce
) external onlyRole(RELAYER_ROLE) {
    require(!processedNonces[nonce], "Nonce processed");
    processedNonces[nonce] = true;
    
    token.transfer(recipient, amount);
    emit Unlocked(recipient, amount, nonce);
}
```

---

## Part 3: 流动性池模型 (Liquidity Pool) (2小时)

适用于原生资产在两条链上都存在的情况（如USDT在ETH和BSC上），不需要铸造包装代币。

### 3.1 原理

1. **源链**: 用户将USDT存入源链流动性池。
2. **中继**: 验证存款。
3. **目标链**: 目标链流动性池将USDT转给用户。

### 3.2 合约实现

```solidity
// LiquidityBridge.sol
contract LiquidityBridge is AccessControl {
    IERC20 public token;
    uint256 public nonce;
    mapping(uint256 => bool) public processedNonces;

    // 流动性提供者 (LP) 存款
    function addLiquidity(uint256 amount) external {
        token.transferFrom(msg.sender, address(this), amount);
    }

    // 跨链转出
    function transferOut(address recipient, uint256 amount, uint256 chainId) external {
        token.transferFrom(msg.sender, address(this), amount);
        emit TransferOut(msg.sender, recipient, amount, chainId, nonce++);
    }

    // 跨链转入 (由Relayer调用)
    function transferIn(
        address recipient, 
        uint256 amount, 
        uint256 srcNonce
    ) external onlyRole(RELAYER_ROLE) {
        require(!processedNonces[srcNonce], "Processed");
        require(token.balanceOf(address(this)) >= amount, "Insufficient liquidity");
        
        processedNonces[srcNonce] = true;
        token.transfer(recipient, amount);
        
        emit TransferIn(recipient, amount, srcNonce);
    }
}
```

### 3.3 流动性再平衡 (Rebalancing)

如果目标链流动性不足，跨链将失败或延迟。通过激励机制鼓励LP在缺水的链上提供流动性。

---

## Part 4: LayerZero 集成 (1.5小时)

LayerZero提供了一个通用的消息传递接口，简化了跨链开发。

### 4.1 接口定义

```solidity
// ILayerZeroEndpoint.sol
interface ILayerZeroEndpoint {
    function send(
        uint16 _dstChainId, 
        bytes calldata _destination, 
        bytes calldata _payload, 
        address payable _refundAddress, 
        address _zroPaymentAddress, 
        bytes calldata _adapterParams
    ) external payable;
}
```

### 4.2 简单的消息跨链

```solidity
// LzApp.sol
import "@layerzerolabs/contracts/lzApp/NonblockingLzApp.sol";

contract MyCrossChainApp is NonblockingLzApp {
    constructor(address _lzEndpoint) NonblockingLzApp(_lzEndpoint) {}

    // 发送消息
    function sendMessage(
        uint16 _dstChainId, 
        string memory _message
    ) public payable {
        bytes memory payload = abi.encode(_message);
        _lzSend(
            _dstChainId, 
            payload, 
            payable(msg.sender), 
            address(0x0), 
            bytes("")
        );
    }

    // 接收消息
    function _nonblockingLzReceive(
        uint16 _srcChainId, 
        bytes memory _srcAddress, 
        uint64 _nonce, 
        bytes memory _payload
    ) internal override {
        string memory message = abi.decode(_payload, (string));
        // 处理消息...
    }
}
```

---

## Part 5: 跨链Gas费估算 (0.5小时)

跨链交易需要在源链支付目标链的执行费用。

```solidity
function estimateFees(
    uint16 _dstChainId, 
    string memory _message
) public view returns (uint256 nativeFee, uint256 zroFee) {
    bytes memory payload = abi.encode(_message);
    return lzEndpoint.estimateFees(
        _dstChainId, 
        address(this), 
        payload, 
        false, 
        bytes("")
    );
}
```

---

## 📝 今日作业

### 作业1: 实现Lock & Mint桥

1. 编写SourceBridge合约（锁定）
2. 编写DestBridge合约（铸造）
3. 部署Wrapped Token合约（只能由Bridge铸造）
4. 编写测试脚本模拟完整流程

### 作业2: LayerZero实战

1. 使用LayerZero实现一个跨链计数器（Omnichain Counter）
2. 在Goerli上增加计数，Sepolia上同步更新
3. 编写前端调用 `estimateFees` 并发送跨链交易

### 作业3: 流动性管理设计

设计一个流动性池跨链桥的激励机制：
1. 当某条链流动性低时，提高LP收益率
2. 收取跨链手续费分配给LP
3. 编写白皮书描述你的设计

---

## ✅ 检查清单

- [ ] 区分Lock/Mint和Liquidity Pool适用场景
- [ ] 理解Wrapped Token的铸造权限管理
- [ ] 掌握LayerZero的基本接口
- [ ] 能够估算跨链Gas费用
- [ ] 理解流动性再平衡的重要性

---

## 📅 明日预告

明天学习消息传递协议：
- 结构化消息设计
- 跨链NFT传输
- 跨链治理投票
- 错误处理与重试机制

**🎉 完成Day 2！你正在构建连接万链的桥梁！**
