# Week 3 - Day 5: DeFi协议学习

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐ (进阶)

## 📚 今日学习目标

- ✅ 理解AMM自动做市商原理
- ✅ 学习Uniswap V2/V3架构
- ✅ 掌握流动性挖矿
- ✅ 学习Staking质押机制
- ✅ 了解收益聚合器

---

## 💧 Part 1: AMM自动做市商 (2小时)

### 1.1 恒定乘积AMM (x * y = k)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 简化的Uniswap V2风格AMM
 * 使用恒定乘积公式: x * y = k
 */
contract SimpleAMM {
    address public token0;
    address public token1;
    
    uint256 public reserve0;
    uint256 public reserve1;
    
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    
    event Mint(address indexed sender, uint256 amount0, uint256 amount1);
    event Burn(address indexed sender, uint256 amount0, uint256 amount1);
    event Swap(address indexed sender, uint256 amountIn, uint256 amountOut, address tokenIn);
    
    constructor(address _token0, address _token1) {
        token0 = _token0;
        token1 = _token1;
    }
    
    // 添加流动性
    function addLiquidity(uint256 amount0, uint256 amount1) 
        external 
        returns (uint256 liquidity) 
    {
        // 转入代币
        IERC20(token0).transferFrom(msg.sender, address(this), amount0);
        IERC20(token1).transferFrom(msg.sender, address(this), amount1);
        
        if (totalSupply == 0) {
            // 第一次添加流动性
            liquidity = sqrt(amount0 * amount1);
        } else {
            // 后续添加流动性，按比例铸造LP代币
            liquidity = min(
                (amount0 * totalSupply) / reserve0,
                (amount1 * totalSupply) / reserve1
            );
        }
        
        require(liquidity > 0, "Insufficient liquidity minted");
        
        balanceOf[msg.sender] += liquidity;
        totalSupply += liquidity;
        
        _update(reserve0 + amount0, reserve1 + amount1);
        
        emit Mint(msg.sender, amount0, amount1);
    }
    
    // 移除流动性
    function removeLiquidity(uint256 liquidity) 
        external 
        returns (uint256 amount0, uint256 amount1) 
    {
        require(balanceOf[msg.sender] >= liquidity, "Insufficient LP tokens");
        
        // 计算可提取的代币数量
        amount0 = (liquidity * reserve0) / totalSupply;
        amount1 = (liquidity * reserve1) / totalSupply;
        
        require(amount0 > 0 && amount1 > 0, "Insufficient liquidity burned");
        
        balanceOf[msg.sender] -= liquidity;
        totalSupply -= liquidity;
        
        _update(reserve0 - amount0, reserve1 - amount1);
        
        // 转出代币
        IERC20(token0).transfer(msg.sender, amount0);
        IERC20(token1).transfer(msg.sender, amount1);
        
        emit Burn(msg.sender, amount0, amount1);
    }
    
    // 交换（简化版，未包含手续费）
    function swap(address tokenIn, uint256 amountIn) 
        external 
        returns (uint256 amountOut) 
    {
        require(
            tokenIn == token0 || tokenIn == token1,
            "Invalid token"
        );
        
        bool isToken0 = tokenIn == token0;
        
        (uint256 reserveIn, uint256 reserveOut) = isToken0 
            ? (reserve0, reserve1) 
            : (reserve1, reserve0);
        
        // 转入代币
        IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
        
        // 计算输出数量: amountOut = (amountIn * reserveOut) / (reserveIn + amountIn)
        // 包含0.3%手续费
        uint256 amountInWithFee = amountIn * 997;
        amountOut = (amountInWithFee * reserveOut) / (reserveIn * 1000 + amountInWithFee);
        
        // 转出代币
        address tokenOut = isToken0 ? token1 : token0;
        IERC20(tokenOut).transfer(msg.sender, amountOut);
        
        // 更新储备量
        _update(
            isToken0 ? reserveIn + amountIn : reserveOut - amountOut,
            isToken0 ? reserveOut - amountOut : reserveIn + amountIn
        );
        
        emit Swap(msg.sender, amountIn, amountOut, tokenIn);
    }
    
    // 更新储备量
    function _update(uint256 _reserve0, uint256 _reserve1) private {
        reserve0 = _reserve0;
        reserve1 = _reserve1;
    }
    
    // 辅助函数
    function sqrt(uint256 y) internal pure returns (uint256 z) {
        if (y > 3) {
            z = y;
            uint256 x = y / 2 + 1;
            while (x < z) {
                z = x;
                x = (y / x + x) / 2;
            }
        } else if (y != 0) {
            z = 1;
        }
    }
    
    function min(uint256 x, uint256 y) internal pure returns (uint256) {
        return x < y ? x : y;
    }
}

interface IERC20 {
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    function transfer(address to, uint256 amount) external returns (bool);
}
```

### 1.2 价格计算和滑点

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract AMMPriceCalculator {
    // 计算输出数量（考虑手续费）
    function getAmountOut(
        uint256 amountIn,
        uint256 reserveIn,
        uint256 reserveOut
    ) public pure returns (uint256 amountOut) {
        require(amountIn > 0, "Insufficient input amount");
        require(reserveIn > 0 && reserveOut > 0, "Insufficient liquidity");
        
        uint256 amountInWithFee = amountIn * 997;
        uint256 numerator = amountInWithFee * reserveOut;
        uint256 denominator = reserveIn * 1000 + amountInWithFee;
        
        amountOut = numerator / denominator;
    }
    
    // 计算需要的输入数量
    function getAmountIn(
        uint256 amountOut,
        uint256 reserveIn,
        uint256 reserveOut
    ) public pure returns (uint256 amountIn) {
        require(amountOut > 0, "Insufficient output amount");
        require(reserveIn > 0 && reserveOut > 0, "Insufficient liquidity");
        
        uint256 numerator = reserveIn * amountOut * 1000;
        uint256 denominator = (reserveOut - amountOut) * 997;
        
        amountIn = (numerator / denominator) + 1;
    }
    
    // 计算价格影响
    function calculatePriceImpact(
        uint256 amountIn,
        uint256 reserveIn,
        uint256 reserveOut
    ) public pure returns (uint256 impact) {
        // 当前价格
        uint256 priceBefore = (reserveOut * 1e18) / reserveIn;
        
        // 交换后价格
        uint256 amountOut = getAmountOut(amountIn, reserveIn, reserveOut);
        uint256 priceAfter = ((reserveOut - amountOut) * 1e18) / (reserveIn + amountIn);
        
        // 价格影响百分比
        if (priceBefore > priceAfter) {
            impact = ((priceBefore - priceAfter) * 10000) / priceBefore;
        } else {
            impact = 0;
        }
    }
}
```

---

## 🦄 Part 2: Uniswap深入 (1.5小时)

### 2.1 Uniswap V2核心概念

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * Uniswap V2风格的工厂合约
 */
contract UniswapV2Factory {
    mapping(address => mapping(address => address)) public getPair;
    address[] public allPairs;
    
    event PairCreated(address indexed token0, address indexed token1, address pair, uint256);
    
    function createPair(address tokenA, address tokenB) external returns (address pair) {
        require(tokenA != tokenB, "Identical addresses");
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
        require(token0 != address(0), "Zero address");
        require(getPair[token0][token1] == address(0), "Pair exists");
        
        // 部署新的交易对合约
        bytes memory bytecode = type(UniswapV2Pair).creationCode;
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        assembly {
            pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }
        
        UniswapV2Pair(pair).initialize(token0, token1);
        
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair;
        allPairs.push(pair);
        
        emit PairCreated(token0, token1, pair, allPairs.length);
    }
}

/**
 * Uniswap V2风格的交易对合约
 */
contract UniswapV2Pair {
    address public factory;
    address public token0;
    address public token1;
    
    uint112 private reserve0;
    uint112 private reserve1;
    uint32 private blockTimestampLast;
    
    uint256 public price0CumulativeLast;
    uint256 public price1CumulativeLast;
    
    uint256 public kLast;
    
    function initialize(address _token0, address _token1) external {
        require(msg.sender == factory, "Forbidden");
        token0 = _token0;
        token1 = _token1;
    }
    
    // 实现核心功能...
}
```

### 2.2 Uniswap V3集中流动性

```markdown
## Uniswap V3关键概念

### 1. 集中流动性
- LP可以选择价格范围提供流动性
- 资本效率更高
- 支持多个费率档位(0.05%, 0.3%, 1%)

### 2. NFT LP代币
- 每个流动性位置是独特的NFT
- 包含价格范围、流动性数量等信息

### 3. Tick系统
- 价格空间被分割成离散的tick
- 每个tick代表0.01%的价格变动

### 4. 灵活费率
- 不同交易对可设置不同费率
- 稳定币对通常使用0.05%
- 普通代币对使用0.3%
- 高风险代币对使用1%
```

---

## 💎 Part 3: 流动性挖矿 (1.5小时)

### 3.1 基础流动性挖矿

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 简单的流动性挖矿合约
 */
contract LiquidityMining {
    IERC20 public stakingToken;  // LP代币
    IERC20 public rewardToken;   // 奖励代币
    
    uint256 public rewardRate = 100;  // 每秒奖励数量
    uint256 public lastUpdateTime;
    uint256 public rewardPerTokenStored;
    
    mapping(address => uint256) public userRewardPerTokenPaid;
    mapping(address => uint256) public rewards;
    
    uint256 private _totalSupply;
    mapping(address => uint256) private _balances;
    
    constructor(address _stakingToken, address _rewardToken) {
        stakingToken = IERC20(_stakingToken);
        rewardToken = IERC20(_rewardToken);
    }
    
    // 每个代币累计的奖励
    function rewardPerToken() public view returns (uint256) {
        if (_totalSupply == 0) {
            return rewardPerTokenStored;
        }
        return rewardPerTokenStored + 
            (((block.timestamp - lastUpdateTime) * rewardRate * 1e18) / _totalSupply);
    }
    
    // 用户已赚取的奖励
    function earned(address account) public view returns (uint256) {
        return ((_balances[account] * 
            (rewardPerToken() - userRewardPerTokenPaid[account])) / 1e18) + 
            rewards[account];
    }
    
    // 更新奖励
    modifier updateReward(address account) {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = block.timestamp;
        
        if (account != address(0)) {
            rewards[account] = earned(account);
            userRewardPerTokenPaid[account] = rewardPerTokenStored;
        }
        _;
    }
    
    // 质押
    function stake(uint256 amount) external updateReward(msg.sender) {
        require(amount > 0, "Cannot stake 0");
        _totalSupply += amount;
        _balances[msg.sender] += amount;
        stakingToken.transferFrom(msg.sender, address(this), amount);
    }
    
    // 取消质押
    function withdraw(uint256 amount) external updateReward(msg.sender) {
        require(amount > 0, "Cannot withdraw 0");
        _totalSupply -= amount;
        _balances[msg.sender] -= amount;
        stakingToken.transfer(msg.sender, amount);
    }
    
    // 领取奖励
    function getReward() external updateReward(msg.sender) {
        uint256 reward = rewards[msg.sender];
        if (reward > 0) {
            rewards[msg.sender] = 0;
            rewardToken.transfer(msg.sender, reward);
        }
    }
    
    // 查询质押数量
    function balanceOf(address account) external view returns (uint256) {
        return _balances[account];
    }
    
    function totalSupply() external view returns (uint256) {
        return _totalSupply;
    }
}
```

### 3.2 多代币奖励挖矿

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 支持多种奖励代币的挖矿合约
 */
contract MultiRewardMining {
    IERC20 public stakingToken;
    
    struct RewardData {
        IERC20 token;
        uint256 rewardRate;
        uint256 rewardPerTokenStored;
        uint256 lastUpdateTime;
    }
    
    RewardData[] public rewardTokens;
    
    mapping(address => mapping(uint256 => uint256)) public userRewardPerTokenPaid;
    mapping(address => mapping(uint256 => uint256)) public rewards;
    
    uint256 private _totalSupply;
    mapping(address => uint256) private _balances;
    
    constructor(address _stakingToken) {
        stakingToken = IERC20(_stakingToken);
    }
    
    // 添加新的奖励代币
    function addRewardToken(address token, uint256 rewardRate) external {
        rewardTokens.push(RewardData({
            token: IERC20(token),
            rewardRate: rewardRate,
            rewardPerTokenStored: 0,
            lastUpdateTime: block.timestamp
        }));
    }
    
    // 质押
    function stake(uint256 amount) external {
        require(amount > 0, "Cannot stake 0");
        
        updateRewards(msg.sender);
        
        _totalSupply += amount;
        _balances[msg.sender] += amount;
        stakingToken.transferFrom(msg.sender, address(this), amount);
    }
    
    // 领取所有奖励
    function getRewards() external {
        updateRewards(msg.sender);
        
        for (uint256 i = 0; i < rewardTokens.length; i++) {
            uint256 reward = rewards[msg.sender][i];
            if (reward > 0) {
                rewards[msg.sender][i] = 0;
                rewardTokens[i].token.transfer(msg.sender, reward);
            }
        }
    }
    
    // 更新所有奖励
    function updateRewards(address account) internal {
        for (uint256 i = 0; i < rewardTokens.length; i++) {
            RewardData storage rewardData = rewardTokens[i];
            
            rewardData.rewardPerTokenStored = rewardPerToken(i);
            rewardData.lastUpdateTime = block.timestamp;
            
            if (account != address(0)) {
                rewards[account][i] = earned(account, i);
                userRewardPerTokenPaid[account][i] = rewardData.rewardPerTokenStored;
            }
        }
    }
    
    function rewardPerToken(uint256 index) public view returns (uint256) {
        RewardData memory rewardData = rewardTokens[index];
        if (_totalSupply == 0) {
            return rewardData.rewardPerTokenStored;
        }
        return rewardData.rewardPerTokenStored + 
            (((block.timestamp - rewardData.lastUpdateTime) * rewardData.rewardRate * 1e18) / _totalSupply);
    }
    
    function earned(address account, uint256 index) public view returns (uint256) {
        return ((_balances[account] * 
            (rewardPerToken(index) - userRewardPerTokenPaid[account][index])) / 1e18) + 
            rewards[account][index];
    }
}
```

---

## 🔒 Part 4: Staking质押 (1小时)

### 4.1 基础Staking

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 简单的Staking合约
 */
contract SimpleStaking {
    IERC20 public stakingToken;
    
    uint256 public constant LOCK_PERIOD = 7 days;
    uint256 public constant APY = 10;  // 10% APY
    
    struct Stake {
        uint256 amount;
        uint256 timestamp;
        uint256 reward;
    }
    
    mapping(address => Stake) public stakes;
    
    constructor(address _stakingToken) {
        stakingToken = IERC20(_stakingToken);
    }
    
    // 质押
    function stake(uint256 amount) external {
        require(amount > 0, "Cannot stake 0");
        require(stakes[msg.sender].amount == 0, "Already staked");
        
        stakingToken.transferFrom(msg.sender, address(this), amount);
        
        stakes[msg.sender] = Stake({
            amount: amount,
            timestamp: block.timestamp,
            reward: 0
        });
    }
    
    // 解除质押
    function unstake() external {
        Stake storage userStake = stakes[msg.sender];
        require(userStake.amount > 0, "No stake found");
        require(
            block.timestamp >= userStake.timestamp + LOCK_PERIOD,
            "Stake is locked"
        );
        
        uint256 reward = calculateReward(msg.sender);
        uint256 total = userStake.amount + reward;
        
        delete stakes[msg.sender];
        
        stakingToken.transfer(msg.sender, total);
    }
    
    // 计算奖励
    function calculateReward(address account) public view returns (uint256) {
        Stake memory userStake = stakes[account];
        if (userStake.amount == 0) return 0;
        
        uint256 stakingDuration = block.timestamp - userStake.timestamp;
        uint256 reward = (userStake.amount * APY * stakingDuration) / (365 days * 100);
        
        return reward;
    }
}
```

---

## 📝 今日作业

### 作业1: 实现AMM DEX

创建一个完整的AMM DEX：
1. 工厂合约和交易对合约
2. 添加/移除流动性
3. 代币交换
4. 价格预言机
5. 手续费分配给LP

### 作业2: 流动性挖矿平台

实现流动性挖矿平台：
1. 支持多个池子
2. 不同池子不同奖励率
3. 紧急提取功能
4. 奖励倍数加成
5. 管理员功能

### 作业3: Staking系统

开发Staking系统：
1. 多种锁定期选择
2. 不同期限不同收益
3. 提前解锁（扣除部分奖励）
4. 复利功能
5. 奖励代币通胀控制

---

## ✅ 今日检查清单

- [ ] 理解AMM原理
- [ ] 掌握Uniswap架构
- [ ] 学会流动性挖矿
- [ ] 掌握Staking机制
- [ ] 完成所有作业

---

## 🆘 常见问题FAQ

### Q1: AMM的无常损失是什么？

A: 当代币价格变化时，LP持有的代币价值可能低于直接持有。价格波动越大，无常损失越大。

### Q2: Uniswap V2和V3的主要区别？

A:
- V2: 全范围流动性，简单但资本效率低
- V3: 集中流动性，资本效率高但管理复杂

### Q3: 如何防止流动性挖矿中的闪电贷攻击？

A:
1. 快照余额而非实时查询
2. 时间加权平均
3. 延迟奖励分发

---

## 📅 明日预告

明天开始周末综合项目：
- TokenSwap合约实战
- 集成多个DeFi协议
- 前后端完整开发

**🎉 完成Day 5！DeFi的世界真精彩！**
