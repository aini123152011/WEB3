# Week 4 - Day 3: Foundry入门 - 上

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 安装配置Foundry环境
- ✅ 学习Forge测试框架
- ✅ 掌握Solidity测试编写
- ✅ 理解Cheatcodes使用
- ✅ 学习Fuzzing测试
- ✅ 掌握测试辅助函数

---

## Part 1: Foundry安装与配置 (1小时)

### 1.1 安装Foundry

#### Windows安装

```powershell
# 方法1: 使用安装脚本（推荐）
curl -L https://foundry.paradigm.xyz | bash
foundryup

# 方法2: 从源码编译
git clone https://github.com/foundry-rs/foundry
cd foundry
cargo build --release
```

#### 验证安装

```bash
# 检查版本
forge --version
cast --version
anvil --version
chisel --version
```

---

### 1.2 创建Foundry项目

```bash
# 创建新项目
forge init my-foundry-project
cd my-foundry-project

# 项目结构
my-foundry-project/
├── foundry.toml        # 配置文件
├── src/                # 合约源码
│   └── Counter.sol
├── test/               # 测试文件
│   └── Counter.t.sol
├── script/             # 部署脚本
│   └── Counter.s.sol
└── lib/                # 依赖库
```

---

### 1.3 配置foundry.toml

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc = "0.8.20"
optimizer = true
optimizer_runs = 200
via_ir = false

# 测试配置
[profile.default.fuzz]
runs = 256
max_test_rejects = 65536

# Gas报告
gas_reports = ["*"]

# 测试输出
verbosity = 2

# Etherscan验证
[etherscan]
mainnet = { key = "${ETHERSCAN_API_KEY}" }
sepolia = { key = "${ETHERSCAN_API_KEY}" }

# RPC端点
[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"
```

---

## Part 2: Forge测试基础 (2小时)

### 2.1 第一个Forge测试

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/Counter.sol";

contract CounterTest is Test {
    Counter public counter;
    
    // setUp在每个测试前运行
    function setUp() public {
        counter = new Counter();
    }
    
    // test开头的函数会被执行
    function test_Increment() public {
        counter.increment();
        assertEq(counter.number(), 1);
    }
    
    function test_SetNumber(uint256 x) public {
        counter.setNumber(x);
        assertEq(counter.number(), x);
    }
    
    // testFail开头的函数期望失败
    function testFail_Decrement() public {
        counter.decrement(); // 应该revert因为number是0
    }
}
```

### 2.2 运行测试

```bash
# 运行所有测试
forge test

# 显示详细输出
forge test -vv

# 显示更详细输出（包括日志）
forge test -vvv

# 显示完整trace
forge test -vvvv

# 运行特定测试
forge test --match-test test_Increment

# 运行特定合约的测试
forge test --match-contract CounterTest

# Gas报告
forge test --gas-report
```

---

### 2.3 断言函数

```solidity
contract AssertionsTest is Test {
    function test_Assertions() public {
        // 相等断言
        assertEq(1, 1);
        assertEq("hello", "hello");
        
        // 不相等断言
        assertNotEq(1, 2);
        
        // 大于/小于
        assertGt(2, 1);  // greater than
        assertGe(2, 2);  // greater than or equal
        assertLt(1, 2);  // less than
        assertLe(1, 1);  // less than or equal
        
        // 布尔断言
        assertTrue(true);
        assertFalse(false);
        
        // 近似相等（用于浮点数比较）
        assertApproxEqAbs(100, 101, 1);      // 绝对误差
        assertApproxEqRel(100, 101, 0.01e18); // 相对误差1%
    }
}
```

---

## Part 3: Cheatcodes魔法函数 (2小时)

### 3.1 账户操纵

```solidity
contract CheatcodesTest is Test {
    ERC20 token;
    
    function setUp() public {
        token = new ERC20("Test", "TST");
    }
    
    function test_Prank() public {
        address alice = address(0x1);
        address bob = address(0x2);
        
        // 以alice身份执行下一次调用
        vm.prank(alice);
        token.transfer(bob, 100);
        
        // 以alice身份执行所有后续调用
        vm.startPrank(alice);
        token.approve(bob, 1000);
        token.transfer(bob, 50);
        vm.stopPrank();
    }
    
    function test_Deal() public {
        address alice = address(0x1);
        
        // 给alice设置ETH余额
        vm.deal(alice, 100 ether);
        assertEq(alice.balance, 100 ether);
        
        // 给alice设置代币余额
        deal(address(token), alice, 1000);
        assertEq(token.balanceOf(alice), 1000);
    }
    
    function test_Label() public {
        address alice = address(0x1);
        vm.label(alice, "Alice");
        
        // 在trace中会显示"Alice"而不是地址
    }
}
```

### 3.2 时间操纵

```solidity
contract TimeTest is Test {
    Auction auction;
    
    function setUp() public {
        auction = new Auction();
    }
    
    function test_TimeTravel() public {
        // 获取当前时间戳
        uint256 start = block.timestamp;
        
        // 前进1天
        vm.warp(start + 1 days);
        assertEq(block.timestamp, start + 1 days);
        
        // 增加区块号
        vm.roll(block.number + 100);
        assertEq(block.number, 101);
    }
    
    function test_Auction() public {
        // 设置时间到拍卖开始
        vm.warp(auction.startTime());
        auction.bid{value: 1 ether}();
        
        // 跳到拍卖结束
        vm.warp(auction.endTime());
        auction.finalize();
    }
}
```

### 3.3 预期失败测试

```solidity
contract RevertTest is Test {
    Token token;
    
    function setUp() public {
        token = new Token();
    }
    
    function test_RevertWithMessage() public {
        // 期望下一次调用revert并带有特定消息
        vm.expectRevert("Insufficient balance");
        token.transfer(address(0x1), 1000);
    }
    
    function test_RevertWithCustomError() public {
        // 期望自定义错误
        vm.expectRevert(
            abi.encodeWithSelector(
                Token.InsufficientBalance.selector,
                0,
                1000
            )
        );
        token.transfer(address(0x1), 1000);
    }
    
    function test_RevertAny() public {
        // 期望任何revert
        vm.expectRevert();
        token.transfer(address(0x1), 1000);
    }
}
```

### 3.4 事件测试

```solidity
contract EventTest is Test {
    Token token;
    
    function setUp() public {
        token = new Token();
    }
    
    function test_EmitEvent() public {
        address alice = address(0x1);
        
        // 期望触发事件
        vm.expectEmit(true, true, true, true);
        emit Transfer(address(this), alice, 100);
        
        token.transfer(alice, 100);
    }
    
    function test_MultipleEvents() public {
        address alice = address(0x1);
        address bob = address(0x2);
        
        // 第一个事件
        vm.expectEmit(true, true, true, true);
        emit Transfer(address(this), alice, 100);
        
        // 第二个事件
        vm.expectEmit(true, true, true, true);
        emit Transfer(address(this), bob, 50);
        
        token.batchTransfer(
            [alice, bob],
            [100, 50]
        );
    }
}
```

### 3.5 存储操纵

```solidity
contract StorageTest is Test {
    Counter counter;
    
    function setUp() public {
        counter = new Counter();
    }
    
    function test_Store() public {
        // 直接修改存储槽
        uint256 slot = 0; // number变量的槽位
        uint256 value = 999;
        
        vm.store(
            address(counter),
            bytes32(slot),
            bytes32(value)
        );
        
        assertEq(counter.number(), 999);
    }
    
    function test_Load() public {
        counter.setNumber(123);
        
        // 读取存储槽
        uint256 slot = 0;
        bytes32 value = vm.load(address(counter), bytes32(slot));
        
        assertEq(uint256(value), 123);
    }
}
```

---

## Part 4: ERC20代币测试实战 (1小时)

### 4.1 完整ERC20测试

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/MyToken.sol";

contract MyTokenTest is Test {
    MyToken public token;
    address public owner;
    address public alice;
    address public bob;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    function setUp() public {
        owner = address(this);
        alice = makeAddr("alice");
        bob = makeAddr("bob");
        
        token = new MyToken("MyToken", "MTK", 1000000);
        
        // 给测试账户一些代币
        token.transfer(alice, 10000);
        token.transfer(bob, 10000);
    }
    
    // 基础属性测试
    function test_Metadata() public {
        assertEq(token.name(), "MyToken");
        assertEq(token.symbol(), "MTK");
        assertEq(token.decimals(), 18);
    }
    
    function test_TotalSupply() public {
        assertEq(token.totalSupply(), 1000000);
    }
    
    function test_InitialBalance() public {
        assertEq(token.balanceOf(owner), 980000);
        assertEq(token.balanceOf(alice), 10000);
        assertEq(token.balanceOf(bob), 10000);
    }
    
    // 转账测试
    function test_Transfer() public {
        vm.prank(alice);
        token.transfer(bob, 1000);
        
        assertEq(token.balanceOf(alice), 9000);
        assertEq(token.balanceOf(bob), 11000);
    }
    
    function test_TransferEmitsEvent() public {
        vm.prank(alice);
        
        vm.expectEmit(true, true, false, true);
        emit Transfer(alice, bob, 1000);
        
        token.transfer(bob, 1000);
    }
    
    function test_TransferInsufficientBalance() public {
        vm.prank(alice);
        
        vm.expectRevert("Insufficient balance");
        token.transfer(bob, 20000);
    }
    
    function test_TransferToZeroAddress() public {
        vm.prank(alice);
        
        vm.expectRevert("Transfer to zero address");
        token.transfer(address(0), 1000);
    }
    
    // 授权测试
    function test_Approve() public {
        vm.prank(alice);
        token.approve(bob, 5000);
        
        assertEq(token.allowance(alice, bob), 5000);
    }
    
    function test_ApproveEmitsEvent() public {
        vm.prank(alice);
        
        vm.expectEmit(true, true, false, true);
        emit Approval(alice, bob, 5000);
        
        token.approve(bob, 5000);
    }
    
    function test_TransferFrom() public {
        // Alice授权Bob
        vm.prank(alice);
        token.approve(bob, 5000);
        
        // Bob从Alice转账到自己
        vm.prank(bob);
        token.transferFrom(alice, bob, 3000);
        
        assertEq(token.balanceOf(alice), 7000);
        assertEq(token.balanceOf(bob), 13000);
        assertEq(token.allowance(alice, bob), 2000);
    }
    
    function test_TransferFromInsufficientAllowance() public {
        vm.prank(alice);
        token.approve(bob, 1000);
        
        vm.prank(bob);
        vm.expectRevert("Insufficient allowance");
        token.transferFrom(alice, bob, 2000);
    }
    
    // Fuzz测试
    function testFuzz_Transfer(uint256 amount) public {
        amount = bound(amount, 0, 10000); // 限制范围
        
        vm.prank(alice);
        token.transfer(bob, amount);
        
        assertEq(token.balanceOf(alice), 10000 - amount);
        assertEq(token.balanceOf(bob), 10000 + amount);
    }
    
    function testFuzz_Approve(uint256 amount) public {
        vm.prank(alice);
        token.approve(bob, amount);
        
        assertEq(token.allowance(alice, bob), amount);
    }
}
```

---

## 📝 今日作业

### 作业1: ERC721 NFT测试

创建并测试ERC721合约：
1. 铸造测试
2. 转账测试
3. 授权测试
4. 元数据测试
5. Fuzz测试

### 作业2: DEX交换合约测试

创建简单DEX并测试：
1. 添加流动性
2. 移除流动性
3. 代币交换
4. 价格计算
5. 边界条件

### 作业3: 时间锁合约测试

实现时间锁并测试：
1. 锁定功能
2. 时间检查
3. 提取功能
4. 异常情况

---

## ✅ 检查清单

- [ ] Foundry安装成功
- [ ] 理解setUp机制
- [ ] 掌握所有断言
- [ ] 会使用Cheatcodes
- [ ] 完成所有作业

---

## 🎯 常见问题FAQ

### Q1: setUp和constructor的区别？

setUp在每个测试前运行，确保测试隔离。constructor只运行一次。

### Q2: 如何处理随机地址？

```solidity
address alice = makeAddr("alice");
vm.deal(alice, 1 ether);
```

### Q3: 如何测试revert？

```solidity
vm.expectRevert("Error message");
contract.function();
```

### Q4: 如何增加测试输出？

```bash
forge test -vv    # 基本输出
forge test -vvv   # 包括日志
forge test -vvvv  # 完整trace
```

---

## 📖 学习记录

**今日学到的最重要的3个知识点**:

1. ___________________________
2. ___________________________
3. ___________________________

**遇到的主要问题及解决方案**:

问题: ___________________________
解决: ___________________________

**明天的学习目标**:

- [ ] ___________________________
- [ ] ___________________________
- [ ] ___________________________

---

## 📅 明日预告

明天学习Foundry入门 - 下：
- 高级Fuzz测试
- Invariant测试
- Fork测试
- 部署脚本

**预习内容**: 了解Property-based testing概念

**🎉 完成Day 3！明天见！**
