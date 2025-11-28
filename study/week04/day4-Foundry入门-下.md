# Week 4 - Day 4: Foundry入门 - 下

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 学习目标

- ✅ 掌握高级Fuzz测试
- ✅ 学习Invariant测试
- ✅ 理解Fork测试
- ✅ 掌握Gas快照
- ✅ 学习部署脚本
- ✅ 理解Cast工具使用

---

## Part 1: 高级Fuzz测试 (2小时)

### 1.1 Fuzz测试配置

```toml
# foundry.toml
[profile.default.fuzz]
runs = 1000                  # 测试运行次数
max_test_rejects = 100000    # 最大拒绝次数
seed = "0x1234"              # 随机种子
```

### 1.2 参数约束

```solidity
contract FuzzTest is Test {
    Token token;
    
    function setUp() public {
        token = new Token("Test", "TST", 1000000);
    }
    
    function testFuzz_Transfer(uint256 amount, address to) public {
        // 约束参数范围
        amount = bound(amount, 0, token.totalSupply());
        vm.assume(to != address(0));
        vm.assume(to != address(token));
        
        uint256 balanceBefore = token.balanceOf(address(this));
        token.transfer(to, amount);
        
        assertEq(
            token.balanceOf(address(this)),
            balanceBefore - amount
        );
        assertEq(token.balanceOf(to), amount);
    }
    
    function testFuzz_TransferPreserveSupply(
        address from,
        address to,
        uint256 amount
    ) public {
        // 排除无效地址
        vm.assume(from != address(0));
        vm.assume(to != address(0));
        vm.assume(from != to);
        
        // 设置初始余额
        deal(address(token), from, 10000);
        amount = bound(amount, 0, 10000);
        
        uint256 supplyBefore = token.totalSupply();
        
        vm.prank(from);
        token.transfer(to, amount);
        
        // 总供应量不变
        assertEq(token.totalSupply(), supplyBefore);
    }
    
    function testFuzz_ApproveTransferFrom(
        address owner,
        address spender,
        address recipient,
        uint256 approveAmount,
        uint256 transferAmount
    ) public {
        vm.assume(owner != address(0));
        vm.assume(spender != address(0));
        vm.assume(recipient != address(0));
        vm.assume(owner != spender);
        
        deal(address(token), owner, 100000);
        approveAmount = bound(approveAmount, 0, 100000);
        transferAmount = bound(transferAmount, 0, approveAmount);
        
        vm.prank(owner);
        token.approve(spender, approveAmount);
        
        vm.prank(spender);
        token.transferFrom(owner, recipient, transferAmount);
        
        assertEq(
            token.allowance(owner, spender),
            approveAmount - transferAmount
        );
    }
}
```

### 1.3 结构化Fuzz测试

```solidity
struct TransferParams {
    address from;
    address to;
    uint256 amount;
}

contract StructuredFuzzTest is Test {
    Token token;
    
    function setUp() public {
        token = new Token("Test", "TST", 1000000);
    }
    
    function testFuzz_StructuredTransfer(TransferParams memory params) public {
        // 约束参数
        vm.assume(params.from != address(0));
        vm.assume(params.to != address(0));
        vm.assume(params.from != params.to);
        
        deal(address(token), params.from, 100000);
        params.amount = bound(params.amount, 0, 100000);
        
        vm.prank(params.from);
        token.transfer(params.to, params.amount);
        
        assertEq(token.balanceOf(params.to), params.amount);
    }
}
```

---

## Part 2: Invariant测试 (2小时)

### 2.1 基础Invariant测试

```solidity
contract InvariantTest is Test {
    Token token;
    Handler handler;
    
    function setUp() public {
        token = new Token("Test", "TST", 1000000);
        handler = new Handler(token);
        
        // 设置目标合约
        targetContract(address(handler));
    }
    
    // 不变量：总供应量始终相等
    function invariant_TotalSupply() public {
        uint256 totalBalance = 0;
        
        address[] memory users = handler.getUsers();
        for (uint256 i = 0; i < users.length; i++) {
            totalBalance += token.balanceOf(users[i]);
        }
        
        assertEq(totalBalance, token.totalSupply());
    }
    
    // 不变量：余额非负
    function invariant_BalanceNonNegative() public {
        address[] memory users = handler.getUsers();
        
        for (uint256 i = 0; i < users.length; i++) {
            assertGe(token.balanceOf(users[i]), 0);
        }
    }
}

contract Handler is Test {
    Token token;
    address[] public users;
    
    constructor(Token _token) {
        token = _token;
        
        // 创建测试用户
        for (uint256 i = 0; i < 5; i++) {
            address user = address(uint160(i + 1));
            users.push(user);
            deal(address(token), user, 10000);
        }
    }
    
    function transfer(uint256 userIndex, uint256 amount) public {
        userIndex = bound(userIndex, 0, users.length - 1);
        address from = users[userIndex];
        address to = users[(userIndex + 1) % users.length];
        
        amount = bound(amount, 0, token.balanceOf(from));
        
        vm.prank(from);
        token.transfer(to, amount);
    }
    
    function approve(uint256 ownerIndex, uint256 spenderIndex, uint256 amount) public {
        ownerIndex = bound(ownerIndex, 0, users.length - 1);
        spenderIndex = bound(spenderIndex, 0, users.length - 1);
        
        address owner = users[ownerIndex];
        address spender = users[spenderIndex];
        
        vm.prank(owner);
        token.approve(spender, amount);
    }
    
    function getUsers() external view returns (address[] memory) {
        return users;
    }
}
```

### 2.2 复杂Invariant测试

```solidity
contract StakingInvariantTest is Test {
    StakingContract staking;
    StakingHandler handler;
    
    function setUp() public {
        staking = new StakingContract();
        handler = new StakingHandler(staking);
        
        targetContract(address(handler));
    }
    
    // 不变量：合约ETH余额 = 所有质押总和
    function invariant_TotalStaked() public {
        assertEq(
            address(staking).balance,
            handler.ghost_totalStaked()
        );
    }
    
    // 不变量：用户质押不能超过其存款
    function invariant_StakeNotExceedDeposit() public {
        address[] memory users = handler.getUsers();
        
        for (uint256 i = 0; i < users.length; i++) {
            assertLe(
                staking.stakes(users[i]),
                handler.getUserTotalDeposit(users[i])
            );
        }
    }
    
    // 不变量：奖励计算正确
    function invariant_RewardCalculation() public {
        uint256 totalRewards = 0;
        address[] memory users = handler.getUsers();
        
        for (uint256 i = 0; i < users.length; i++) {
            totalRewards += staking.calculateReward(users[i]);
        }
        
        // 总奖励不超过奖励池
        assertLe(totalRewards, staking.rewardPool());
    }
}

contract StakingHandler is Test {
    StakingContract staking;
    address[] public users;
    
    // Ghost变量追踪状态
    uint256 public ghost_totalStaked;
    mapping(address => uint256) public ghost_userDeposits;
    
    constructor(StakingContract _staking) {
        staking = _staking;
        
        for (uint256 i = 0; i < 5; i++) {
            address user = address(uint160(i + 1));
            users.push(user);
            vm.deal(user, 100 ether);
        }
    }
    
    function stake(uint256 userIndex, uint256 amount) public {
        userIndex = bound(userIndex, 0, users.length - 1);
        address user = users[userIndex];
        
        amount = bound(amount, 0, user.balance);
        
        vm.prank(user);
        staking.stake{value: amount}();
        
        ghost_totalStaked += amount;
        ghost_userDeposits[user] += amount;
    }
    
    function unstake(uint256 userIndex, uint256 amount) public {
        userIndex = bound(userIndex, 0, users.length - 1);
        address user = users[userIndex];
        
        uint256 staked = staking.stakes(user);
        amount = bound(amount, 0, staked);
        
        if (amount == 0) return;
        
        vm.prank(user);
        staking.unstake(amount);
        
        ghost_totalStaked -= amount;
    }
    
    function getUsers() external view returns (address[] memory) {
        return users;
    }
    
    function getUserTotalDeposit(address user) external view returns (uint256) {
        return ghost_userDeposits[user];
    }
}
```

---

## Part 3: Fork测试 (1.5小时)

### 3.1 Fork主网测试

```solidity
contract ForkTest is Test {
    IERC20 USDC = IERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);
    IERC20 DAI = IERC20(0x6B175474E89094C44Da98b954EedeAC495271d0F);
    IUniswapV2Router router = IUniswapV2Router(0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D);
    
    uint256 mainnetFork;
    
    function setUp() public {
        // Fork主网
        mainnetFork = vm.createFork(vm.envString("MAINNET_RPC_URL"));
        vm.selectFork(mainnetFork);
        
        // 验证fork成功
        assertEq(vm.activeFork(), mainnetFork);
    }
    
    function test_SwapOnUniswap() public {
        address trader = makeAddr("trader");
        
        // 给trader一些USDC
        deal(address(USDC), trader, 1000e6); // 1000 USDC
        
        vm.startPrank(trader);
        
        // 批准router
        USDC.approve(address(router), 1000e6);
        
        // 准备交换路径
        address[] memory path = new address[](2);
        path[0] = address(USDC);
        path[1] = address(DAI);
        
        // 执行交换
        uint256 daiBefore = DAI.balanceOf(trader);
        
        router.swapExactTokensForTokens(
            1000e6,
            0,
            path,
            trader,
            block.timestamp
        );
        
        uint256 daiAfter = DAI.balanceOf(trader);
        
        // 验证收到DAI
        assertGt(daiAfter, daiBefore);
        
        vm.stopPrank();
    }
    
    function test_ForkAtBlock() public {
        // Fork特定区块
        uint256 blockNumber = 15_000_000;
        uint256 fork = vm.createFork(
            vm.envString("MAINNET_RPC_URL"),
            blockNumber
        );
        vm.selectFork(fork);
        
        assertEq(block.number, blockNumber);
    }
}
```

### 3.2 多Fork测试

```solidity
contract MultiForkTest is Test {
    uint256 mainnetFork;
    uint256 optimismFork;
    
    IERC20 mainnetUSDC;
    IERC20 optimismUSDC;
    
    function setUp() public {
        // Fork多个网络
        mainnetFork = vm.createFork(vm.envString("MAINNET_RPC_URL"));
        optimismFork = vm.createFork(vm.envString("OPTIMISM_RPC_URL"));
        
        // 主网USDC
        vm.selectFork(mainnetFork);
        mainnetUSDC = IERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);
        
        // Optimism USDC
        vm.selectFork(optimismFork);
        optimismUSDC = IERC20(0x7F5c764cBc14f9669B88837ca1490cCa17c31607);
    }
    
    function test_CompareBalances() public {
        address holder = address(0x123);
        
        // 检查主网余额
        vm.selectFork(mainnetFork);
        uint256 mainnetBalance = mainnetUSDC.balanceOf(holder);
        
        // 检查Optimism余额
        vm.selectFork(optimismFork);
        uint256 optimismBalance = optimismUSDC.balanceOf(holder);
        
        console.log("Mainnet balance:", mainnetBalance);
        console.log("Optimism balance:", optimismBalance);
    }
}
```

---

## Part 4: Gas快照与优化 (1小时)

### 4.1 Gas快照

```bash
# 生成Gas快照
forge snapshot

# 比较快照
forge snapshot --diff .gas-snapshot

# 检查Gas变化
forge snapshot --check
```

### 4.2 Gas报告

```bash
# 生成Gas报告
forge test --gas-report

# 保存到文件
forge test --gas-report > gas-report.txt
```

```solidity
contract GasTest is Test {
    Token token;
    
    function setUp() public {
        token = new Token("Test", "TST", 1000000);
    }
    
    function test_GasUsage_Transfer() public {
        uint256 gasBefore = gasleft();
        
        token.transfer(address(0x1), 100);
        
        uint256 gasUsed = gasBefore - gasleft();
        console.log("Gas used:", gasUsed);
        
        // 断言Gas使用在合理范围内
        assertLt(gasUsed, 100000);
    }
    
    function test_CompareGas_SingleVsBatch() public {
        address[] memory recipients = new address[](10);
        for (uint256 i = 0; i < 10; i++) {
            recipients[i] = address(uint160(i + 1));
        }
        
        // 单独转账
        uint256 gasSnapshot = gasleft();
        for (uint256 i = 0; i < 10; i++) {
            token.transfer(recipients[i], 100);
        }
        uint256 gasSingle = gasSnapshot - gasleft();
        
        // 批量转账
        vm.snapshotGasLastCall("batchTransfer");
        token.batchTransfer(recipients, 100);
        
        console.log("Single transfers gas:", gasSingle);
    }
}
```

---

## Part 5: 部署脚本 (1.5小时)

### 5.1 基础部署脚本

```solidity
// script/DeployToken.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "../src/Token.sol";

contract DeployToken is Script {
    function run() external {
        // 获取私钥
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        
        // 开始广播交易
        vm.startBroadcast(deployerPrivateKey);
        
        // 部署合约
        Token token = new Token("MyToken", "MTK", 1000000);
        
        console.log("Token deployed at:", address(token));
        
        vm.stopBroadcast();
    }
}
```

运行部署:

```bash
# 模拟部署
forge script script/DeployToken.s.sol

# 实际部署到Sepolia
forge script script/DeployToken.s.sol \
    --rpc-url $SEPOLIA_RPC_URL \
    --broadcast \
    --verify
```

### 5.2 复杂部署脚本

```solidity
contract DeployDEX is Script {
    function run() external returns (
        Token tokenA,
        Token tokenB,
        DEX dex
    ) {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerPrivateKey);
        
        console.log("Deploying from:", deployer);
        console.log("Deployer balance:", deployer.balance);
        
        vm.startBroadcast(deployerPrivateKey);
        
        // 1. 部署代币A
        tokenA = new Token("TokenA", "TKA", 1000000);
        console.log("TokenA deployed:", address(tokenA));
        
        // 2. 部署代币B
        tokenB = new Token("TokenB", "TKB", 1000000);
        console.log("TokenB deployed:", address(tokenB));
        
        // 3. 部署DEX
        dex = new DEX(address(tokenA), address(tokenB));
        console.log("DEX deployed:", address(dex));
        
        // 4. 添加初始流动性
        tokenA.approve(address(dex), 10000);
        tokenB.approve(address(dex), 10000);
        dex.addLiquidity(10000, 10000);
        
        console.log("Initial liquidity added");
        
        vm.stopBroadcast();
        
        // 5. 验证部署
        require(dex.tokenA() == address(tokenA), "TokenA mismatch");
        require(dex.tokenB() == address(tokenB), "TokenB mismatch");
        
        console.log("Deployment successful!");
    }
}
```

---

## Part 6: Cast工具使用 (1小时)

### 6.1 查询区块链数据

```bash
# 获取区块信息
cast block latest

# 获取账户余额
cast balance 0x...

# 获取代币余额
cast call TOKEN_ADDRESS \
    "balanceOf(address)(uint256)" \
    YOUR_ADDRESS

# 获取合约代码
cast code CONTRACT_ADDRESS

# 获取存储槽
cast storage CONTRACT_ADDRESS 0
```

### 6.2 发送交易

```bash
# 发送ETH
cast send RECIPIENT \
    --value 1ether \
    --private-key $PRIVATE_KEY

# 调用合约函数
cast send TOKEN_ADDRESS \
    "transfer(address,uint256)" \
    RECIPIENT \
    1000000000000000000 \
    --private-key $PRIVATE_KEY

# 估算Gas
cast estimate TOKEN_ADDRESS \
    "transfer(address,uint256)" \
    RECIPIENT \
    1000000000000000000
```

### 6.3 工具函数

```bash
# 地址转换
cast --to-checksum-address 0xabc...

# 单位转换
cast --to-wei 1 ether
cast --from-wei 1000000000000000000

# Keccak256哈希
cast keccak "Transfer(address,address,uint256)"

# ABI编码
cast abi-encode "transfer(address,uint256)" 0x... 100

# 签名生成
cast sig "transfer(address,uint256)"
```

---

## 📝 今日作业

### 作业1: Invariant测试实战

为ERC20代币编写Invariant测试：
1. 总供应量不变
2. 余额总和等于总供应
3. 授权逻辑正确
4. 至少5个不变量

### 作业2: Fork测试实战

Fork主网测试Uniswap：
1. 测试代币交换
2. 测试添加流动性
3. 测试移除流动性
4. Gas对比分析

### 作业3: 完整项目部署

部署完整DEX项目：
1. 编写部署脚本
2. 部署到测试网
3. 验证合约
4. 添加初始流动性

---

## ✅ 检查清单

- [ ] 掌握Fuzz测试
- [ ] 理解Invariant测试
- [ ] 会Fork测试
- [ ] 会写部署脚本
- [ ] 掌握Cast工具

---

## 🎯 常见问题FAQ

### Q1: Fuzz测试runs设置多少合适？

开发阶段256-1000次，生产前10000+次。

### Q2: Invariant测试怎么写？

1. 定义不变量
2. 创建Handler
3. 运行随机操作
4. 验证不变量

### Q3: Fork测试有什么限制？

需要RPC节点，可能较慢，注意区块高度。

### Q4: 如何优化Gas？

1. 使用Gas快照
2. 对比优化前后
3. 识别热点函数
4. 应用优化技巧

---

## 📅 明日预告

明天学习测试覆盖率与最佳实践：
- 测试策略制定
- 覆盖率提升
- CI/CD集成
- 测试最佳实践

**🎉 完成Day 4！明天见！**
