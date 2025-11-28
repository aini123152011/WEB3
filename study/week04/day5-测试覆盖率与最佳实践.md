# Week 4 - Day 5: 测试覆盖率与最佳实践

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 掌握覆盖率测试
- ✅ 学习测试策略制定
- ✅ 理解CI/CD集成
- ✅ 掌握测试最佳实践
- ✅ 学习安全测试方法
- ✅ 理解性能测试技巧

---

## Part 1: 覆盖率测试 (2小时)

### 1.1 Hardhat覆盖率

```bash
# 安装
npm install --save-dev solidity-coverage

# 运行覆盖率
npx hardhat coverage

# 指定文件
npx hardhat coverage --testfiles "test/Token.test.js"

# 生成lcov报告
npx hardhat coverage --solcoverjs .solcover.js
```

`.solcover.js`配置:

```javascript
module.exports = {
  skipFiles: ['test/', 'mock/'],
  mocha: {
    timeout: 100000
  },
  providerOptions: {
    gasLimit: 10000000
  }
};
```

### 1.2 Foundry覆盖率

```bash
# 生成覆盖率报告
forge coverage

# 详细报告
forge coverage --report lcov
forge coverage --report summary

# 指定测试文件
forge coverage --match-path test/Token.t.sol
```

### 1.3 覆盖率目标

```markdown
## 覆盖率标准

- 行覆盖率（Line Coverage）: ≥95%
- 分支覆盖率（Branch Coverage）: ≥90%
- 函数覆盖率（Function Coverage）: 100%
- 语句覆盖率（Statement Coverage）: ≥95%

## 重点覆盖区域

1. **关键业务逻辑**: 100%
2. **资金处理函数**: 100%
3. **权限控制**: 100%
4. **错误处理**: ≥95%
5. **边界条件**: ≥90%
```

---

## Part 2: 测试策略制定 (1.5小时)

### 2.1 测试金字塔

```markdown
         /\
        /  \        E2E测试 (10%)
       /____\       - 完整用户流程
      /      \      - 集成多个合约
     /        \     
    /__________\    集成测试 (30%)
   /            \   - 多合约交互
  /              \  - 真实场景模拟
 /________________\ 
/                  \ 单元测试 (60%)
/__________________\ - 单个函数
                      - 隔离测试
```

### 2.2 测试分类

```solidity
// test/unit/Token.unit.t.sol
contract TokenUnitTest is Test {
    Token token;
    
    function setUp() public {
        token = new Token("Test", "TST", 1000000);
    }
    
    // 单元测试：测试单个函数
    function test_Transfer() public {
        address to = address(0x1);
        token.transfer(to, 100);
        assertEq(token.balanceOf(to), 100);
    }
}

// test/integration/DEX.integration.t.sol
contract DEXIntegrationTest is Test {
    Token tokenA;
    Token tokenB;
    DEX dex;
    
    function setUp() public {
        tokenA = new Token("A", "A", 1000000);
        tokenB = new Token("B", "B", 1000000);
        dex = new DEX(address(tokenA), address(tokenB));
    }
    
    // 集成测试：测试多合约交互
    function test_SwapFlow() public {
        tokenA.approve(address(dex), 10000);
        tokenB.approve(address(dex), 10000);
        dex.addLiquidity(10000, 10000);
        
        uint256 amountOut = dex.swap(1000, true);
        assertGt(amountOut, 0);
    }
}

// test/e2e/Platform.e2e.t.sol
contract PlatformE2ETest is Test {
    // E2E测试：完整业务流程
    function test_CompleteUserJourney() public {
        // 1. 用户注册
        // 2. 存款
        // 3. 交易
        // 4. 提款
        // 5. 验证最终状态
    }
}
```

### 2.3 测试优先级

```markdown
## P0 - 必须测试（阻塞发布）
- 资金安全相关
- 权限控制
- 核心业务逻辑
- 已知安全漏洞

## P1 - 应该测试（强烈推荐）
- 边界条件
- 错误处理
- 状态转换
- 事件触发

## P2 - 可以测试（时间允许）
- 辅助功能
- View函数
- 工具函数
- 优化逻辑

## P3 - 可选测试
- 极端边界
- 性能测试
- 压力测试
```

---

## Part 3: CI/CD集成 (1.5小时)

### 3.1 GitHub Actions配置

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Install Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npx hardhat test
      
      - name: Run coverage
        run: npx hardhat coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
          
  foundry-test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: recursive
      
      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1
      
      - name: Run tests
        run: forge test -vvv
      
      - name: Check coverage
        run: |
          forge coverage --report summary > coverage.txt
          cat coverage.txt
```

### 3.2 GitLab CI配置

```yaml
# .gitlab-ci.yml
stages:
  - test
  - coverage
  - deploy

test:hardhat:
  stage: test
  image: node:18
  script:
    - npm ci
    - npx hardhat test
  artifacts:
    reports:
      junit: test-results.xml

test:foundry:
  stage: test
  image: ghcr.io/foundry-rs/foundry:latest
  script:
    - forge test -vvv
  
coverage:
  stage: coverage
  image: node:18
  script:
    - npm ci
    - npx hardhat coverage
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'
  artifacts:
    paths:
      - coverage/
```

### 3.3 pre-commit钩子

```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "Running tests..."
npm test

echo "Checking coverage..."
npm run coverage:check

echo "Running linter..."
npm run lint

echo "Checking format..."
npm run format:check
```

---

## Part 4: 测试最佳实践 (1.5小时)

### 4.1 测试命名规范

```solidity
contract TokenTest is Test {
    // ✅ 好的命名：描述性强，清晰明了
    function test_Transfer_UpdatesBalances() public {}
    function test_Transfer_EmitsEvent() public {}
    function test_Transfer_RevertsWhenInsufficientBalance() public {}
    function testFuzz_Transfer_PreservesTotalSupply(uint256 amount) public {}
    
    // ❌ 不好的命名：不清晰
    function test_Transfer1() public {}
    function test_Token() public {}
    function test_Works() public {}
}
```

### 4.2 测试组织结构

```solidity
contract TokenTest is Test {
    Token token;
    address alice;
    address bob;
    
    // setUp: 每个测试前运行
    function setUp() public {
        token = new Token("Test", "TST", 1000000);
        alice = makeAddr("alice");
        bob = makeAddr("bob");
        
        deal(address(token), alice, 10000);
    }
    
    // 按功能分组
    // ========== Transfer Tests ==========
    
    function test_Transfer_Success() public {
        vm.prank(alice);
        token.transfer(bob, 1000);
        assertEq(token.balanceOf(bob), 1000);
    }
    
    function test_Transfer_RevertInsufficientBalance() public {
        vm.prank(alice);
        vm.expectRevert("Insufficient balance");
        token.transfer(bob, 20000);
    }
    
    // ========== Approval Tests ==========
    
    function test_Approve_Success() public {
        vm.prank(alice);
        token.approve(bob, 5000);
        assertEq(token.allowance(alice, bob), 5000);
    }
}
```

### 4.3 AAA模式（Arrange-Act-Assert）

```solidity
function test_Transfer_AAA() public {
    // Arrange（准备）
    address sender = alice;
    address recipient = bob;
    uint256 amount = 1000;
    uint256 senderBalanceBefore = token.balanceOf(sender);
    
    // Act（执行）
    vm.prank(sender);
    token.transfer(recipient, amount);
    
    // Assert（断言）
    assertEq(token.balanceOf(sender), senderBalanceBefore - amount);
    assertEq(token.balanceOf(recipient), amount);
}
```

### 4.4 测试数据管理

```solidity
contract TestData {
    // 使用常量
    uint256 constant INITIAL_SUPPLY = 1000000;
    uint256 constant LARGE_AMOUNT = 1000000 ether;
    address constant ZERO_ADDRESS = address(0);
    
    // 使用辅助函数
    function createUser(string memory name) internal returns (address) {
        address user = makeAddr(name);
        vm.deal(user, 100 ether);
        return user;
    }
    
    function createUsers(uint256 count) internal returns (address[] memory) {
        address[] memory users = new address[](count);
        for (uint256 i = 0; i < count; i++) {
            users[i] = createUser(string(abi.encodePacked("user", i)));
        }
        return users;
    }
}
```

---

## Part 5: 安全测试方法 (1小时)

### 5.1 常见漏洞测试

```solidity
contract SecurityTest is Test {
    // 测试重入攻击
    function test_ReentrancyProtection() public {
        ReentrancyAttacker attacker = new ReentrancyAttacker(vault);
        
        vm.expectRevert("ReentrancyGuard");
        attacker.attack();
    }
    
    // 测试整数溢出
    function test_OverflowProtection() public {
        vm.expectRevert(); // Solidity 0.8+自动检查
        token.transfer(address(0x1), type(uint256).max);
    }
    
    // 测试访问控制
    function test_OnlyOwner() public {
        vm.prank(alice);
        vm.expectRevert("Ownable: caller is not the owner");
        token.mint(alice, 1000);
    }
    
    // 测试前端运行
    function test_FrontRunningProtection() public {
        // 设置最小输出保护
        vm.prank(alice);
        dex.swap(1000, 900); // 最小接受900
        
        // 尝试插入交易
        vm.prank(frontrunner);
        dex.swap(5000, 0);
        
        // 验证保护生效
        vm.prank(alice);
        vm.expectRevert("Slippage too high");
        dex.executeSwap();
    }
}
```

### 5.2 模糊测试安全场景

```solidity
function testFuzz_NoUnauthorizedMint(address caller, uint256 amount) public {
    vm.assume(caller != owner);
    
    vm.prank(caller);
    vm.expectRevert();
    token.mint(caller, amount);
}

function testFuzz_TransferPreservesInvariant(
    address from,
    address to,
    uint256 amount
) public {
    vm.assume(from != address(0));
    vm.assume(to != address(0));
    vm.assume(from != to);
    
    deal(address(token), from, 100000);
    amount = bound(amount, 0, 100000);
    
    uint256 totalBefore = token.totalSupply();
    
    vm.prank(from);
    token.transfer(to, amount);
    
    assertEq(token.totalSupply(), totalBefore);
}
```

---

## Part 6: 性能与Gas测试 (0.5小时)

### 6.1 Gas基准测试

```solidity
contract GasBenchmark is Test {
    Token token;
    
    function setUp() public {
        token = new Token("Test", "TST", 1000000);
    }
    
    function testGas_Transfer() public {
        token.transfer(address(0x1), 100);
    }
    
    function testGas_Approve() public {
        token.approve(address(0x1), 1000);
    }
    
    function testGas_TransferFrom() public {
        token.approve(address(this), 1000);
        token.transferFrom(address(this), address(0x1), 100);
    }
}
```

运行Gas报告:

```bash
forge test --gas-report
forge snapshot
```

---

## 📝 今日作业

### 作业1: 提升覆盖率

为现有项目提升覆盖率到95%+：
1. 运行覆盖率分析
2. 识别未覆盖代码
3. 编写缺失测试
4. 验证覆盖率达标

### 作业2: 配置CI/CD

为项目配置自动化测试：
1. 创建GitHub Actions
2. 添加测试任务
3. 配置覆盖率上传
4. 添加状态徽章

### 作业3: 安全测试

编写安全测试用例：
1. 重入攻击测试
2. 访问控制测试
3. 整数溢出测试
4. 前端运行测试

---

## ✅ 检查清单

- [ ] 覆盖率≥95%
- [ ] CI/CD配置完成
- [ ] 遵循测试最佳实践
- [ ] 完成安全测试
- [ ] Gas报告生成

---

## 🎯 常见问题FAQ

### Q1: 覆盖率100%就安全吗？

不一定。覆盖率只衡量代码执行，不保证逻辑正确性。还需要：
- 边界测试
- 安全审计
- 模糊测试
- Invariant测试

### Q2: 如何提高测试效率？

1. 使用测试夹具
2. 并行运行测试
3. 只测试变更部分
4. 使用快照

### Q3: 测试多长时间合适？

- 单元测试: <1秒/测试
- 集成测试: <5秒/测试
- E2E测试: <30秒/测试

### Q4: 如何测试私有函数？

1. 测试公共接口
2. 使用internal改为public
3. 创建测试辅助合约

---

## 📖 学习记录

**今日学到的最重要的3个知识点**:

1. ___________________________
2. ___________________________
3. ___________________________

**遇到的主要问题及解决方案**:

问题: ___________________________
解决: ___________________________

---

## 📅 明日预告

明天周末综合项目：DeFi协议测试：
- AMM协议测试
- 流动性池测试
- 闪电贷测试
- 完整测试套件

**🎉 完成Day 5！明天见！**
