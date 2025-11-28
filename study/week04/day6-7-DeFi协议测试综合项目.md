# Week 4 - Day 6-7: DeFi协议测试综合项目

**学习日期**: ___________
**预计用时**: 12-14小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 项目目标

实现完整DeFi协议测试套件：
- ✅ AMM自动做市商测试
- ✅ 流动性池管理测试
- ✅ 闪电贷功能测试
- ✅ 价格预言机测试
- ✅ 治理系统测试
- ✅ 95%+测试覆盖率

---

## 🎯 Day 6: 核心功能测试 (6-7小时)

### Part 1: 项目设置与结构 (1小时)

#### 1.1 项目结构

```
defi-protocol-test/
├── contracts/
│   ├── core/
│   │   ├── AMM.sol
│   │   ├── LiquidityPool.sol
│   │   └── FlashLoan.sol
│   ├── tokens/
│   │   ├── Token.sol
│   │   └── LPToken.sol
│   └── oracle/
│       └── PriceOracle.sol
├── test/
│   ├── unit/
│   │   ├── AMM.t.sol
│   │   ├── LiquidityPool.t.sol
│   │   └── FlashLoan.t.sol
│   ├── integration/
│   │   └── Protocol.t.sol
│   └── fuzz/
│       └── Invariants.t.sol
└── script/
    └── Deploy.s.sol
```

#### 1.2 测试配置

```toml
# foundry.toml
[profile.default]
src = "contracts"
out = "out"
libs = ["lib"]
solc = "0.8.20"

[profile.default.fuzz]
runs = 1000

[profile.default.invariant]
runs = 256
depth = 15
fail_on_revert = true
```

---

### Part 2: AMM测试套件 (2小时)

#### 2.1 单元测试

```solidity
// test/unit/AMM.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../../contracts/core/AMM.sol";
import "../../contracts/tokens/Token.sol";

contract AMMUnitTest is Test {
    AMM public amm;
    Token public tokenA;
    Token public tokenB;
    
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    
    event LiquidityAdded(address indexed provider, uint256 amountA, uint256 amountB);
    event LiquidityRemoved(address indexed provider, uint256 amountA, uint256 amountB);
    event Swap(address indexed trader, bool aToB, uint256 amountIn, uint256 amountOut);
    
    function setUp() public {
        tokenA = new Token("TokenA", "TKA", 1000000 ether);
        tokenB = new Token("TokenB", "TKB", 1000000 ether);
        amm = new AMM(address(tokenA), address(tokenB));
        
        // 分配代币
        tokenA.transfer(alice, 100000 ether);
        tokenB.transfer(alice, 100000 ether);
        tokenA.transfer(bob, 50000 ether);
        tokenB.transfer(bob, 50000 ether);
    }
    
    // ========== 添加流动性测试 ==========
    
    function test_AddLiquidity_FirstProvider() public {
        vm.startPrank(alice);
        
        tokenA.approve(address(amm), 10000 ether);
        tokenB.approve(address(amm), 10000 ether);
        
        vm.expectEmit(true, false, false, true);
        emit LiquidityAdded(alice, 10000 ether, 10000 ether);
        
        uint256 liquidity = amm.addLiquidity(10000 ether, 10000 ether, 0);
        
        assertEq(amm.reserveA(), 10000 ether);
        assertEq(amm.reserveB(), 10000 ether);
        assertGt(liquidity, 0);
        
        vm.stopPrank();
    }
    
    function test_AddLiquidity_SubsequentProvider() public {
        // 第一次添加
        vm.startPrank(alice);
        tokenA.approve(address(amm), 10000 ether);
        tokenB.approve(address(amm), 10000 ether);
        amm.addLiquidity(10000 ether, 10000 ether, 0);
        vm.stopPrank();
        
        // 第二次添加
        vm.startPrank(bob);
        tokenA.approve(address(amm), 5000 ether);
        tokenB.approve(address(amm), 5000 ether);
        
        uint256 liquidity = amm.addLiquidity(5000 ether, 5000 ether, 0);
        
        assertEq(amm.reserveA(), 15000 ether);
        assertEq(amm.reserveB(), 15000 ether);
        assertGt(liquidity, 0);
        
        vm.stopPrank();
    }
    
    function test_AddLiquidity_RevertImbalancedRatio() public {
        // 先添加初始流动性 1:1
        vm.startPrank(alice);
        tokenA.approve(address(amm), 10000 ether);
        tokenB.approve(address(amm), 10000 ether);
        amm.addLiquidity(10000 ether, 10000 ether, 0);
        vm.stopPrank();
        
        // 尝试以2:1比例添加（应失败）
        vm.startPrank(bob);
        tokenA.approve(address(amm), 10000 ether);
        tokenB.approve(address(amm), 5000 ether);
        
        vm.expectRevert("Imbalanced ratio");
        amm.addLiquidity(10000 ether, 5000 ether, 0);
        
        vm.stopPrank();
    }
    
    // ========== 移除流动性测试 ==========
    
    function test_RemoveLiquidity_Success() public {
        // 添加流动性
        vm.startPrank(alice);
        tokenA.approve(address(amm), 10000 ether);
        tokenB.approve(address(amm), 10000 ether);
        uint256 liquidity = amm.addLiquidity(10000 ether, 10000 ether, 0);
        
        // 移除一半流动性
        uint256 balanceABefore = tokenA.balanceOf(alice);
        uint256 balanceBBefore = tokenB.balanceOf(alice);
        
        (uint256 amountA, uint256 amountB) = amm.removeLiquidity(liquidity / 2, 0, 0);
        
        assertGt(tokenA.balanceOf(alice), balanceABefore);
        assertGt(tokenB.balanceOf(alice), balanceBBefore);
        assertApproxEqRel(amountA, 5000 ether, 0.01e18); // 1%误差
        assertApproxEqRel(amountB, 5000 ether, 0.01e18);
        
        vm.stopPrank();
    }
    
    // ========== 交换测试 ==========
    
    function test_Swap_AtoB() public {
        // 添加流动性
        vm.startPrank(alice);
        tokenA.approve(address(amm), 10000 ether);
        tokenB.approve(address(amm), 10000 ether);
        amm.addLiquidity(10000 ether, 10000 ether, 0);
        vm.stopPrank();
        
        // Bob进行交换
        vm.startPrank(bob);
        tokenA.approve(address(amm), 1000 ether);
        
        uint256 balanceBBefore = tokenB.balanceOf(bob);
        
        vm.expectEmit(true, false, false, false);
        emit Swap(bob, true, 1000 ether, 0);
        
        uint256 amountOut = amm.swap(1000 ether, 0, true);
        
        assertGt(amountOut, 0);
        assertGt(tokenB.balanceOf(bob), balanceBBefore);
        
        vm.stopPrank();
    }
    
    function test_Swap_PreservesConstantProduct() public {
        // 添加流动性
        vm.startPrank(alice);
        tokenA.approve(address(amm), 10000 ether);
        tokenB.approve(address(amm), 10000 ether);
        amm.addLiquidity(10000 ether, 10000 ether, 0);
        vm.stopPrank();
        
        uint256 k = amm.reserveA() * amm.reserveB();
        
        // 交换
        vm.startPrank(bob);
        tokenA.approve(address(amm), 1000 ether);
        amm.swap(1000 ether, 0, true);
        vm.stopPrank();
        
        // 考虑手续费后，k应该增加或保持
        uint256 kAfter = amm.reserveA() * amm.reserveB();
        assertGe(kAfter, k);
    }
    
    function test_Swap_RevertSlippageTooHigh() public {
        vm.startPrank(alice);
        tokenA.approve(address(amm), 10000 ether);
        tokenB.approve(address(amm), 10000 ether);
        amm.addLiquidity(10000 ether, 10000 ether, 0);
        vm.stopPrank();
        
        vm.startPrank(bob);
        tokenA.approve(address(amm), 1000 ether);
        
        // 设置不合理的最小输出
        vm.expectRevert("Slippage too high");
        amm.swap(1000 ether, 1000 ether, true);
        
        vm.stopPrank();
    }
    
    // ========== 价格计算测试 ==========
    
    function test_GetPrice_Correct() public {
        vm.startPrank(alice);
        tokenA.approve(address(amm), 10000 ether);
        tokenB.approve(address(amm), 20000 ether);
        amm.addLiquidity(10000 ether, 20000 ether, 0);
        vm.stopPrank();
        
        uint256 price = amm.getPrice(true); // A的价格用B表示
        assertEq(price, 2 ether); // 1 A = 2 B
    }
    
    function test_GetAmountOut_Correct() public {
        vm.startPrank(alice);
        tokenA.approve(address(amm), 10000 ether);
        tokenB.approve(address(amm), 10000 ether);
        amm.addLiquidity(10000 ether, 10000 ether, 0);
        vm.stopPrank();
        
        uint256 amountOut = amm.getAmountOut(1000 ether, true);
        assertGt(amountOut, 0);
        assertLt(amountOut, 1000 ether); // 考虑手续费
    }
}
```

#### 2.2 Fuzz测试

```solidity
contract AMMFuzzTest is Test {
    AMM public amm;
    Token public tokenA;
    Token public tokenB;
    
    function setUp() public {
        tokenA = new Token("TokenA", "TKA", type(uint128).max);
        tokenB = new Token("TokenB", "TKB", type(uint128).max);
        amm = new AMM(address(tokenA), address(tokenB));
    }
    
    function testFuzz_AddLiquidity(uint256 amountA, uint256 amountB) public {
        amountA = bound(amountA, 1000, type(uint128).max);
        amountB = bound(amountB, 1000, type(uint128).max);
        
        tokenA.approve(address(amm), amountA);
        tokenB.approve(address(amm), amountB);
        
        uint256 liquidity = amm.addLiquidity(amountA, amountB, 0);
        
        assertEq(amm.reserveA(), amountA);
        assertEq(amm.reserveB(), amountB);
        assertGt(liquidity, 0);
    }
    
    function testFuzz_Swap_PreservesK(uint256 amountIn) public {
        // 设置初始流动性
        uint256 initialA = 100000 ether;
        uint256 initialB = 100000 ether;
        
        tokenA.approve(address(amm), initialA);
        tokenB.approve(address(amm), initialB);
        amm.addLiquidity(initialA, initialB, 0);
        
        amountIn = bound(amountIn, 100, initialA / 10);
        
        uint256 k = amm.reserveA() * amm.reserveB();
        
        tokenA.approve(address(amm), amountIn);
        amm.swap(amountIn, 0, true);
        
        // 考虑手续费，k应该增加
        uint256 kAfter = amm.reserveA() * amm.reserveB();
        assertGe(kAfter, k);
    }
}
```

---

### Part 3: 闪电贷测试 (1.5小时)

```solidity
// test/unit/FlashLoan.t.sol
contract FlashLoanTest is Test {
    FlashLoan public flashLoan;
    Token public token;
    FlashBorrower borrower;
    
    address alice = makeAddr("alice");
    
    function setUp() public {
        token = new Token("Test", "TST", 1000000 ether);
        flashLoan = new FlashLoan(address(token));
        borrower = new FlashBorrower();
        
        // 向池子存入代币
        token.transfer(address(flashLoan), 100000 ether);
    }
    
    function test_FlashLoan_Success() public {
        uint256 amount = 10000 ether;
        uint256 fee = (amount * 9) / 10000; // 0.09% fee
        
        // 给借款人足够的代币支付手续费
        token.transfer(address(borrower), fee);
        
        vm.prank(alice);
        flashLoan.flashLoan(address(borrower), amount);
        
        // 验证手续费已收取
        assertEq(token.balanceOf(address(flashLoan)), 100000 ether + fee);
    }
    
    function test_FlashLoan_RevertNotRepaid() public {
        FlashBorrowerBad badBorrower = new FlashBorrowerBad();
        
        vm.expectRevert("Flash loan not repaid");
        flashLoan.flashLoan(address(badBorrower), 10000 ether);
    }
    
    function test_FlashLoan_Reentrancy() public {
        ReentrancyAttacker attacker = new ReentrancyAttacker(flashLoan);
        
        vm.expectRevert("ReentrancyGuard");
        attacker.attack();
    }
}

// 正常借款人
contract FlashBorrower {
    function onFlashLoan(uint256 amount, uint256 fee, bytes calldata) external {
        // 执行套利逻辑
        
        // 归还借款+手续费
        IERC20 token = IERC20(msg.sender);
        token.transfer(msg.sender, amount + fee);
    }
}

// 恶意借款人（不归还）
contract FlashBorrowerBad {
    function onFlashLoan(uint256, uint256, bytes calldata) external {
        // 不归还
    }
}
```

---

### Part 4: Invariant测试 (1.5小时)

```solidity
// test/fuzz/Invariants.t.sol
contract ProtocolInvariantTest is Test {
    AMM public amm;
    Token public tokenA;
    Token public tokenB;
    Handler public handler;
    
    function setUp() public {
        tokenA = new Token("TokenA", "TKA", type(uint128).max);
        tokenB = new Token("TokenB", "TKB", type(uint128).max);
        amm = new AMM(address(tokenA), address(tokenB));
        
        handler = new Handler(amm, tokenA, tokenB);
        targetContract(address(handler));
    }
    
    // 不变量1: K值只增不减
    function invariant_KNeverDecreases() public {
        assertGe(
            amm.reserveA() * amm.reserveB(),
            handler.minK()
        );
    }
    
    // 不变量2: 代币总量守恒
    function invariant_TokenConservation() public {
        uint256 totalInAMM = amm.reserveA() + amm.reserveB();
        uint256 totalInUsers = handler.getTotalUserBalance();
        
        assertEq(
            totalInAMM + totalInUsers,
            tokenA.totalSupply() + tokenB.totalSupply()
        );
    }
    
    // 不变量3: 价格合理
    function invariant_ReasonablePrice() public {
        if (amm.reserveA() > 0 && amm.reserveB() > 0) {
            uint256 price = amm.getPrice(true);
            assertLt(price, 1000 ether); // 价格不会太离谱
            assertGt(price, 0.001 ether);
        }
    }
}

contract Handler is Test {
    AMM public amm;
    Token public tokenA;
    Token public tokenB;
    address[] public users;
    
    uint256 public minK;
    
    constructor(AMM _amm, Token _tokenA, Token _tokenB) {
        amm = _amm;
        tokenA = _tokenA;
        tokenB = _tokenB;
        
        for (uint256 i = 0; i < 5; i++) {
            address user = address(uint160(i + 1));
            users.push(user);
            
            deal(address(tokenA), user, 100000 ether);
            deal(address(tokenB), user, 100000 ether);
        }
    }
    
    function addLiquidity(uint256 userIndex, uint256 amountA, uint256 amountB) public {
        userIndex = bound(userIndex, 0, users.length - 1);
        address user = users[userIndex];
        
        amountA = bound(amountA, 1000, tokenA.balanceOf(user));
        amountB = bound(amountB, 1000, tokenB.balanceOf(user));
        
        vm.startPrank(user);
        tokenA.approve(address(amm), amountA);
        tokenB.approve(address(amm), amountB);
        amm.addLiquidity(amountA, amountB, 0);
        vm.stopPrank();
        
        uint256 k = amm.reserveA() * amm.reserveB();
        if (k > minK) minK = k;
    }
    
    function swap(uint256 userIndex, uint256 amount, bool aToB) public {
        userIndex = bound(userIndex, 0, users.length - 1);
        address user = users[userIndex];
        
        Token tokenIn = aToB ? tokenA : tokenB;
        amount = bound(amount, 100, tokenIn.balanceOf(user) / 10);
        
        if (amm.reserveA() == 0 || amm.reserveB() == 0) return;
        
        vm.startPrank(user);
        tokenIn.approve(address(amm), amount);
        try amm.swap(amount, 0, aToB) {} catch {}
        vm.stopPrank();
    }
    
    function getTotalUserBalance() public view returns (uint256) {
        uint256 total = 0;
        for (uint256 i = 0; i < users.length; i++) {
            total += tokenA.balanceOf(users[i]);
            total += tokenB.balanceOf(users[i]);
        }
        return total;
    }
}
```

---

## 📝 Day 6作业

1. 完成所有单元测试
2. 运行覆盖率分析
3. 达到95%+覆盖率
4. 修复所有失败测试

---

## ✅ Day 6检查清单

- [ ] AMM测试完成
- [ ] 闪电贷测试完成
- [ ] Invariant测试完成
- [ ] 所有测试通过
- [ ] 覆盖率≥95%

---

## 📅 明日预告 (Day 7)

明天完成集成测试和部署：
- 端到端测试
- Gas优化测试
- 安全测试
- 部署到测试网

**🎉 完成Day 6！明天见！**
