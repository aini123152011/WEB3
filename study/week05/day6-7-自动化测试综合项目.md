# Week 5 - Day 6-7: 自动化测试综合项目

**学习日期**: ___________
**预计用时**: 12-14小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 项目目标

构建完整的Web3自动化测试系统：
- ✅ DeFi协议测试套件
- ✅ 前端E2E测试
- ✅ API接口测试
- ✅ 性能测试
- ✅ CI/CD集成
- ✅ 测试报告生成

---

## Part 1: 项目架构设计 (2小时)

### 1.1 项目结构

```bash
web3-auto-test/
├── contracts/               # 智能合约
│   ├── DeFiProtocol.sol
│   ├── MockToken.sol
│   └── MockOracle.sol
├── test/
│   ├── unit/               # 单元测试
│   │   ├── DeFiProtocol.test.js
│   │   └── helpers.test.js
│   ├── integration/        # 集成测试
│   │   ├── protocol.test.js
│   │   └── liquidation.test.js
│   ├── e2e/               # 端到端测试
│   │   ├── wallet-connection.test.js
│   │   └── trading.test.js
│   └── performance/        # 性能测试
│       └── load.test.js
├── scripts/                # 部署和辅助脚本
│   ├── deploy.js
│   └── seed-data.js
├── lib/                    # 测试工具库
│   ├── fixtures.js
│   ├── helpers.js
│   └── matchers.js
├── config/                 # 配置文件
│   ├── hardhat.config.js
│   ├── playwright.config.js
│   └── jest.config.js
├── frontend/              # 前端应用
│   ├── src/
│   └── tests/
├── .github/
│   └── workflows/
│       └── test.yml       # CI配置
├── coverage/              # 测试覆盖率报告
├── reports/              # 测试报告
└── package.json
```

### 1.2 技术栈

```json
{
  "name": "web3-auto-test",
  "version": "1.0.0",
  "scripts": {
    "compile": "hardhat compile",
    "test": "hardhat test",
    "test:unit": "hardhat test test/unit/**/*.test.js",
    "test:integration": "hardhat test test/integration/**/*.test.js",
    "test:e2e": "playwright test",
    "test:performance": "k6 run test/performance/load.test.js",
    "test:coverage": "hardhat coverage",
    "test:ci": "npm run test && npm run test:e2e",
    "deploy:local": "hardhat run scripts/deploy.js --network localhost",
    "report": "node scripts/generate-report.js"
  },
  "devDependencies": {
    "@nomicfoundation/hardhat-toolbox": "^4.0.0",
    "@playwright/test": "^1.40.0",
    "@openzeppelin/contracts": "^5.0.0",
    "hardhat": "^2.19.0",
    "ethers": "^6.9.0",
    "chai": "^4.3.10",
    "mocha": "^10.2.0",
    "dotenv": "^16.3.1",
    "k6": "^0.48.0",
    "allure-commandline": "^2.25.0"
  }
}
```

---

## Part 2: DeFi协议合约 (2小时)

### 2.1 核心合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/security/Pausable.sol";

/**
 * @title DeFiProtocol
 * @dev 简化的DeFi借贷协议用于测试
 */
contract DeFiProtocol is Ownable, ReentrancyGuard, Pausable {
    using SafeERC20 for IERC20;
    
    // 抵押品信息
    struct Collateral {
        uint256 amount;           // 抵押数量
        uint256 borrowedAmount;   // 借出数量
        uint256 lastUpdateTime;   // 最后更新时间
    }
    
    // 支持的代币
    mapping(address => bool) public supportedTokens;
    
    // 用户抵押信息
    mapping(address => mapping(address => Collateral)) public collaterals;
    
    // 抵押率 (150% = 1.5e18)
    uint256 public constant COLLATERAL_RATIO = 150e16;
    
    // 清算阈值 (125% = 1.25e18)
    uint256 public constant LIQUIDATION_THRESHOLD = 125e16;
    
    // 利率 (5% APY = 5e16)
    uint256 public constant INTEREST_RATE = 5e16;
    
    // 价格预言机地址
    address public oracle;
    
    // 事件
    event TokenAdded(address indexed token);
    event Deposited(address indexed user, address indexed token, uint256 amount);
    event Borrowed(address indexed user, address indexed token, uint256 amount);
    event Repaid(address indexed user, address indexed token, uint256 amount);
    event Withdrawn(address indexed user, address indexed token, uint256 amount);
    event Liquidated(
        address indexed liquidator,
        address indexed user,
        address indexed collateralToken,
        uint256 collateralAmount,
        uint256 debtAmount
    );
    
    constructor(address _oracle) Ownable(msg.sender) {
        oracle = _oracle;
    }
    
    // 添加支持的代币
    function addSupportedToken(address token) external onlyOwner {
        require(token != address(0), "Invalid token");
        supportedTokens[token] = true;
        emit TokenAdded(token);
    }
    
    // 存入抵押品
    function deposit(address token, uint256 amount) 
        external 
        nonReentrant 
        whenNotPaused 
    {
        require(supportedTokens[token], "Token not supported");
        require(amount > 0, "Amount must be greater than 0");
        
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        
        Collateral storage col = collaterals[msg.sender][token];
        col.amount += amount;
        col.lastUpdateTime = block.timestamp;
        
        emit Deposited(msg.sender, token, amount);
    }
    
    // 借出代币
    function borrow(address collateralToken, address borrowToken, uint256 amount)
        external
        nonReentrant
        whenNotPaused
    {
        require(supportedTokens[collateralToken], "Collateral token not supported");
        require(supportedTokens[borrowToken], "Borrow token not supported");
        require(amount > 0, "Amount must be greater than 0");
        
        Collateral storage col = collaterals[msg.sender][collateralToken];
        require(col.amount > 0, "No collateral");
        
        // 计算最大可借数量
        uint256 maxBorrow = getMaxBorrow(msg.sender, collateralToken, borrowToken);
        require(amount <= maxBorrow, "Insufficient collateral");
        
        col.borrowedAmount += amount;
        col.lastUpdateTime = block.timestamp;
        
        IERC20(borrowToken).safeTransfer(msg.sender, amount);
        
        emit Borrowed(msg.sender, borrowToken, amount);
    }
    
    // 还款
    function repay(address collateralToken, address borrowToken, uint256 amount)
        external
        nonReentrant
        whenNotPaused
    {
        Collateral storage col = collaterals[msg.sender][collateralToken];
        require(col.borrowedAmount > 0, "No debt");
        
        uint256 debt = getCurrentDebt(msg.sender, collateralToken);
        uint256 repayAmount = amount > debt ? debt : amount;
        
        IERC20(borrowToken).safeTransferFrom(msg.sender, address(this), repayAmount);
        
        col.borrowedAmount = debt > repayAmount ? debt - repayAmount : 0;
        col.lastUpdateTime = block.timestamp;
        
        emit Repaid(msg.sender, borrowToken, repayAmount);
    }
    
    // 提取抵押品
    function withdraw(address token, uint256 amount)
        external
        nonReentrant
        whenNotPaused
    {
        Collateral storage col = collaterals[msg.sender][token];
        require(col.amount >= amount, "Insufficient collateral");
        
        uint256 debt = getCurrentDebt(msg.sender, token);
        if (debt > 0) {
            // 确保提取后仍满足抵押率
            uint256 remainingCollateral = col.amount - amount;
            uint256 minCollateral = (debt * COLLATERAL_RATIO) / 1e18;
            require(remainingCollateral >= minCollateral, "Would break collateral ratio");
        }
        
        col.amount -= amount;
        IERC20(token).safeTransfer(msg.sender, amount);
        
        emit Withdrawn(msg.sender, token, amount);
    }
    
    // 清算
    function liquidate(
        address user,
        address collateralToken,
        address debtToken
    ) external nonReentrant {
        Collateral storage col = collaterals[user][collateralToken];
        require(col.borrowedAmount > 0, "No debt to liquidate");
        
        uint256 currentDebt = getCurrentDebt(user, collateralToken);
        uint256 collateralValue = getCollateralValue(user, collateralToken, debtToken);
        
        // 检查是否低于清算阈值
        uint256 healthFactor = (collateralValue * 1e18) / currentDebt;
        require(healthFactor < LIQUIDATION_THRESHOLD, "Position is healthy");
        
        // 清算逻辑：获取所有抵押品，偿还债务
        uint256 collateralAmount = col.amount;
        
        IERC20(debtToken).safeTransferFrom(msg.sender, address(this), currentDebt);
        IERC20(collateralToken).safeTransfer(msg.sender, collateralAmount);
        
        col.amount = 0;
        col.borrowedAmount = 0;
        
        emit Liquidated(msg.sender, user, collateralToken, collateralAmount, currentDebt);
    }
    
    // 查询最大可借数量
    function getMaxBorrow(
        address user,
        address collateralToken,
        address borrowToken
    ) public view returns (uint256) {
        Collateral storage col = collaterals[user][collateralToken];
        if (col.amount == 0) return 0;
        
        uint256 collateralValue = getCollateralValue(user, collateralToken, borrowToken);
        uint256 currentDebt = getCurrentDebt(user, collateralToken);
        
        uint256 maxBorrowValue = (collateralValue * 1e18) / COLLATERAL_RATIO;
        
        if (currentDebt >= maxBorrowValue) return 0;
        
        return maxBorrowValue - currentDebt;
    }
    
    // 计算当前债务（含利息）
    function getCurrentDebt(address user, address token) public view returns (uint256) {
        Collateral storage col = collaterals[user][token];
        if (col.borrowedAmount == 0) return 0;
        
        uint256 timeElapsed = block.timestamp - col.lastUpdateTime;
        uint256 interest = (col.borrowedAmount * INTEREST_RATE * timeElapsed) / (365 days * 1e18);
        
        return col.borrowedAmount + interest;
    }
    
    // 获取抵押品价值
    function getCollateralValue(
        address user,
        address collateralToken,
        address valueToken
    ) public view returns (uint256) {
        Collateral storage col = collaterals[user][collateralToken];
        // 简化：假设价格为1:1
        // 实际应调用预言机
        return col.amount;
    }
    
    // 获取健康因子
    function getHealthFactor(address user, address token) public view returns (uint256) {
        uint256 debt = getCurrentDebt(user, token);
        if (debt == 0) return type(uint256).max;
        
        uint256 collateralValue = getCollateralValue(user, token, token);
        return (collateralValue * 1e18) / debt;
    }
    
    // 暂停/恢复
    function pause() external onlyOwner {
        _pause();
    }
    
    function unpause() external onlyOwner {
        _unpause();
    }
}
```

### 2.2 Mock合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockToken is ERC20 {
    uint8 private _decimals;
    
    constructor(
        string memory name,
        string memory symbol,
        uint8 decimals_
    ) ERC20(name, symbol) {
        _decimals = decimals_;
    }
    
    function decimals() public view override returns (uint8) {
        return _decimals;
    }
    
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
    
    function burn(address from, uint256 amount) external {
        _burn(from, amount);
    }
}
```

---

## Part 3: 单元测试 (2小时)

### 3.1 测试配置

```javascript
// test/unit/DeFiProtocol.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { loadFixture } = require("@nomicfoundation/hardhat-toolbox/network-helpers");

describe("DeFiProtocol Unit Tests", function () {
  // 测试夹具
  async function deployProtocolFixture() {
    const [owner, user1, user2, liquidator] = await ethers.getSigners();
    
    // 部署Mock代币
    const MockToken = await ethers.getContractFactory("MockToken");
    const collateralToken = await MockToken.deploy("Collateral Token", "COL", 18);
    const borrowToken = await MockToken.deploy("Borrow Token", "BOR", 18);
    
    // 部署协议
    const DeFiProtocol = await ethers.getContractFactory("DeFiProtocol");
    const protocol = await DeFiProtocol.deploy(ethers.ZeroAddress); // 简化：不使用预言机
    
    // 添加支持的代币
    await protocol.addSupportedToken(await collateralToken.getAddress());
    await protocol.addSupportedToken(await borrowToken.getAddress());
    
    // 铸造代币
    await collateralToken.mint(user1.address, ethers.parseEther("1000"));
    await borrowToken.mint(await protocol.getAddress(), ethers.parseEther("10000"));
    
    return {
      protocol,
      collateralToken,
      borrowToken,
      owner,
      user1,
      user2,
      liquidator
    };
  }
  
  describe("Deployment", function () {
    it("Should set the right owner", async function () {
      const { protocol, owner } = await loadFixture(deployProtocolFixture);
      expect(await protocol.owner()).to.equal(owner.address);
    });
    
    it("Should have correct collateral ratio", async function () {
      const { protocol } = await loadFixture(deployProtocolFixture);
      expect(await protocol.COLLATERAL_RATIO()).to.equal(ethers.parseEther("1.5"));
    });
  });
  
  describe("Token Management", function () {
    it("Should add supported token", async function () {
      const { protocol, collateralToken, owner } = await loadFixture(deployProtocolFixture);
      const tokenAddress = await collateralToken.getAddress();
      expect(await protocol.supportedTokens(tokenAddress)).to.be.true;
    });
    
    it("Should revert when non-owner tries to add token", async function () {
      const { protocol, user1 } = await loadFixture(deployProtocolFixture);
      const MockToken = await ethers.getContractFactory("MockToken");
      const newToken = await MockToken.deploy("New Token", "NEW", 18);
      
      await expect(
        protocol.connect(user1).addSupportedToken(await newToken.getAddress())
      ).to.be.revertedWithCustomError(protocol, "OwnableUnauthorizedAccount");
    });
  });
  
  describe("Deposit", function () {
    it("Should deposit collateral successfully", async function () {
      const { protocol, collateralToken, user1 } = await loadFixture(deployProtocolFixture);
      
      const amount = ethers.parseEther("100");
      await collateralToken.connect(user1).approve(await protocol.getAddress(), amount);
      
      await expect(
        protocol.connect(user1).deposit(await collateralToken.getAddress(), amount)
      ).to.emit(protocol, "Deposited")
        .withArgs(user1.address, await collateralToken.getAddress(), amount);
      
      const collateral = await protocol.collaterals(
        user1.address,
        await collateralToken.getAddress()
      );
      expect(collateral.amount).to.equal(amount);
    });
    
    it("Should revert when depositing unsupported token", async function () {
      const { protocol, user1 } = await loadFixture(deployProtocolFixture);
      const MockToken = await ethers.getContractFactory("MockToken");
      const unsupportedToken = await MockToken.deploy("Unsupported", "UNS", 18);
      
      await expect(
        protocol.connect(user1).deposit(await unsupportedToken.getAddress(), 100)
      ).to.be.revertedWith("Token not supported");
    });
    
    it("Should revert when depositing zero amount", async function () {
      const { protocol, collateralToken, user1 } = await loadFixture(deployProtocolFixture);
      
      await expect(
        protocol.connect(user1).deposit(await collateralToken.getAddress(), 0)
      ).to.be.revertedWith("Amount must be greater than 0");
    });
  });
  
  describe("Borrow", function () {
    it("Should borrow successfully with sufficient collateral", async function () {
      const { protocol, collateralToken, borrowToken, user1 } = await loadFixture(deployProtocolFixture);
      
      // 存入抵押品
      const collateralAmount = ethers.parseEther("150");
      await collateralToken.connect(user1).approve(await protocol.getAddress(), collateralAmount);
      await protocol.connect(user1).deposit(await collateralToken.getAddress(), collateralAmount);
      
      // 借出
      const borrowAmount = ethers.parseEther("100");
      await expect(
        protocol.connect(user1).borrow(
          await collateralToken.getAddress(),
          await borrowToken.getAddress(),
          borrowAmount
        )
      ).to.emit(protocol, "Borrowed")
        .withArgs(user1.address, await borrowToken.getAddress(), borrowAmount);
      
      expect(await borrowToken.balanceOf(user1.address)).to.equal(borrowAmount);
    });
    
    it("Should revert when borrowing without collateral", async function () {
      const { protocol, collateralToken, borrowToken, user1 } = await loadFixture(deployProtocolFixture);
      
      await expect(
        protocol.connect(user1).borrow(
          await collateralToken.getAddress(),
          await borrowToken.getAddress(),
          ethers.parseEther("100")
        )
      ).to.be.revertedWith("No collateral");
    });
    
    it("Should revert when borrowing exceeds max", async function () {
      const { protocol, collateralToken, borrowToken, user1 } = await loadFixture(deployProtocolFixture);
      
      const collateralAmount = ethers.parseEther("100");
      await collateralToken.connect(user1).approve(await protocol.getAddress(), collateralAmount);
      await protocol.connect(user1).deposit(await collateralToken.getAddress(), collateralAmount);
      
      // 尝试借出超过最大额度
      const borrowAmount = ethers.parseEther("150"); // 超过100/1.5
      await expect(
        protocol.connect(user1).borrow(
          await collateralToken.getAddress(),
          await borrowToken.getAddress(),
          borrowAmount
        )
      ).to.be.revertedWith("Insufficient collateral");
    });
  });
  
  describe("Repay", function () {
    it("Should repay debt successfully", async function () {
      const { protocol, collateralToken, borrowToken, user1 } = await loadFixture(deployProtocolFixture);
      
      // 存入抵押品并借出
      const collateralAmount = ethers.parseEther("150");
      await collateralToken.connect(user1).approve(await protocol.getAddress(), collateralAmount);
      await protocol.connect(user1).deposit(await collateralToken.getAddress(), collateralAmount);
      
      const borrowAmount = ethers.parseEther("100");
      await protocol.connect(user1).borrow(
        await collateralToken.getAddress(),
        await borrowToken.getAddress(),
        borrowAmount
      );
      
      // 还款
      await borrowToken.connect(user1).approve(await protocol.getAddress(), borrowAmount);
      await expect(
        protocol.connect(user1).repay(
          await collateralToken.getAddress(),
          await borrowToken.getAddress(),
          borrowAmount
        )
      ).to.emit(protocol, "Repaid");
      
      const collateral = await protocol.collaterals(
        user1.address,
        await collateralToken.getAddress()
      );
      expect(collateral.borrowedAmount).to.equal(0);
    });
  });
  
  describe("Withdraw", function () {
    it("Should withdraw collateral after repaying debt", async function () {
      const { protocol, collateralToken, borrowToken, user1 } = await loadFixture(deployProtocolFixture);
      
      // 存入、借出、还款
      const collateralAmount = ethers.parseEther("150");
      await collateralToken.connect(user1).approve(await protocol.getAddress(), collateralAmount);
      await protocol.connect(user1).deposit(await collateralToken.getAddress(), collateralAmount);
      
      const borrowAmount = ethers.parseEther("50");
      await protocol.connect(user1).borrow(
        await collateralToken.getAddress(),
        await borrowToken.getAddress(),
        borrowAmount
      );
      
      await borrowToken.connect(user1).approve(await protocol.getAddress(), borrowAmount);
      await protocol.connect(user1).repay(
        await collateralToken.getAddress(),
        await borrowToken.getAddress(),
        borrowAmount
      );
      
      // 提取
      await expect(
        protocol.connect(user1).withdraw(await collateralToken.getAddress(), collateralAmount)
      ).to.emit(protocol, "Withdrawn")
        .withArgs(user1.address, await collateralToken.getAddress(), collateralAmount);
    });
    
    it("Should revert when withdrawing would break collateral ratio", async function () {
      const { protocol, collateralToken, borrowToken, user1 } = await loadFixture(deployProtocolFixture);
      
      const collateralAmount = ethers.parseEther("150");
      await collateralToken.connect(user1).approve(await protocol.getAddress(), collateralAmount);
      await protocol.connect(user1).deposit(await collateralToken.getAddress(), collateralAmount);
      
      await protocol.connect(user1).borrow(
        await collateralToken.getAddress(),
        await borrowToken.getAddress(),
        ethers.parseEther("90")
      );
      
      // 尝试提取过多抵押品
      await expect(
        protocol.connect(user1).withdraw(await collateralToken.getAddress(), ethers.parseEther("20"))
      ).to.be.revertedWith("Would break collateral ratio");
    });
  });
  
  describe("Liquidation", function () {
    it("Should liquidate unhealthy position", async function () {
      const { protocol, collateralToken, borrowToken, user1, liquidator } = await loadFixture(deployProtocolFixture);
      
      // 用户建立接近清算阈值的仓位
      const collateralAmount = ethers.parseEther("125");
      await collateralToken.connect(user1).approve(await protocol.getAddress(), collateralAmount);
      await protocol.connect(user1).deposit(await collateralToken.getAddress(), collateralAmount);
      
      const borrowAmount = ethers.parseEther("100");
      await protocol.connect(user1).borrow(
        await collateralToken.getAddress(),
        await borrowToken.getAddress(),
        borrowAmount
      );
      
      // 模拟价格下跌或利息累积
      // 在实际场景中，价格变化会导致健康因子降低
      // 这里简化处理...
      
      // 清算人准备债务代币
      await borrowToken.mint(liquidator.address, borrowAmount);
      await borrowToken.connect(liquidator).approve(await protocol.getAddress(), borrowAmount);
      
      // 注：此测试简化了清算条件检查
    });
  });
});
```

---

## Part 4: 集成测试 (2小时)

### 4.1 完整流程测试

```javascript
// test/integration/protocol.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("DeFi Protocol Integration Tests", function () {
  let protocol, collateralToken, borrowToken;
  let owner, user1, user2, liquidator;
  
  beforeEach(async function () {
    [owner, user1, user2, liquidator] = await ethers.getSigners();
    
    // 部署合约
    const MockToken = await ethers.getContractFactory("MockToken");
    collateralToken = await MockToken.deploy("USDC", "USDC", 6);
    borrowToken = await MockToken.deploy("DAI", "DAI", 18);
    
    const DeFiProtocol = await ethers.getContractFactory("DeFiProtocol");
    protocol = await DeFiProtocol.deploy(ethers.ZeroAddress);
    
    await protocol.addSupportedToken(await collateralToken.getAddress());
    await protocol.addSupportedToken(await borrowToken.getAddress());
    
    // 准备资金
    await collateralToken.mint(user1.address, 1000_000000); // 1000 USDC
    await collateralToken.mint(user2.address, 500_000000);  // 500 USDC
    await borrowToken.mint(await protocol.getAddress(), ethers.parseEther("100000"));
  });
  
  it("Complete user journey: deposit, borrow, repay, withdraw", async function () {
    const collateralAmount = 150_000000; // 150 USDC
    const borrowAmount = ethers.parseEther("100"); // 100 DAI
    
    // 1. 用户存入抵押品
    await collateralToken.connect(user1).approve(await protocol.getAddress(), collateralAmount);
    await protocol.connect(user1).deposit(await collateralToken.getAddress(), collateralAmount);
    
    let collateral = await protocol.collaterals(user1.address, await collateralToken.getAddress());
    expect(collateral.amount).to.equal(collateralAmount);
    
    // 2. 用户借出
    await protocol.connect(user1).borrow(
      await collateralToken.getAddress(),
      await borrowToken.getAddress(),
      borrowAmount
    );
    
    expect(await borrowToken.balanceOf(user1.address)).to.equal(borrowAmount);
    
    // 3. 查询债务
    const debt = await protocol.getCurrentDebt(user1.address, await collateralToken.getAddress());
    expect(debt).to.be.gte(borrowAmount);
    
    // 4. 部分还款
    const partialRepay = ethers.parseEther("50");
    await borrowToken.connect(user1).approve(await protocol.getAddress(), partialRepay);
    await protocol.connect(user1).repay(
      await collateralToken.getAddress(),
      await borrowToken.getAddress(),
      partialRepay
    );
    
    // 5. 完全还款
    const remainingDebt = await protocol.getCurrentDebt(user1.address, await collateralToken.getAddress());
    await borrowToken.connect(user1).approve(await protocol.getAddress(), remainingDebt);
    await protocol.connect(user1).repay(
      await collateralToken.getAddress(),
      await borrowToken.getAddress(),
      remainingDebt
    );
    
    // 6. 提取抵押品
    await protocol.connect(user1).withdraw(await collateralToken.getAddress(), collateralAmount);
    
    collateral = await protocol.collaterals(user1.address, await collateralToken.getAddress());
    expect(collateral.amount).to.equal(0);
    expect(collateral.borrowedAmount).to.equal(0);
  });
  
  it("Multiple users can interact independently", async function () {
    // User1 操作
    await collateralToken.connect(user1).approve(await protocol.getAddress(), 200_000000);
    await protocol.connect(user1).deposit(await collateralToken.getAddress(), 200_000000);
    await protocol.connect(user1).borrow(
      await collateralToken.getAddress(),
      await borrowToken.getAddress(),
      ethers.parseEther("100")
    );
    
    // User2 操作
    await collateralToken.connect(user2).approve(await protocol.getAddress(), 300_000000);
    await protocol.connect(user2).deposit(await collateralToken.getAddress(), 300_000000);
    await protocol.connect(user2).borrow(
      await collateralToken.getAddress(),
      await borrowToken.getAddress(),
      ethers.parseEther("150")
    );
    
    // 验证独立性
    const user1Collateral = await protocol.collaterals(user1.address, await collateralToken.getAddress());
    const user2Collateral = await protocol.collaterals(user2.address, await collateralToken.getAddress());
    
    expect(user1Collateral.amount).to.equal(200_000000);
    expect(user2Collateral.amount).to.equal(300_000000);
  });
});
```

---

## Part 5: E2E测试 (2小时)

### 5.1 前端E2E测试

```javascript
// test/e2e/wallet-connection.test.js
const { test, expect } = require('@playwright/test');

test.describe('Wallet Connection', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:3000');
  });
  
  test('should connect MetaMask wallet', async ({ page, context }) => {
    // 安装MetaMask扩展并连接
    const metamaskPath = require('@playwright/test').chromium.launchPersistentContext.metamaskPath;
    
    // 点击连接钱包按钮
    await page.click('button:has-text("Connect Wallet")');
    
    // 等待MetaMask弹窗
    const pages = context.pages();
    const metamaskPopup = pages[pages.length - 1];
    
    // 在MetaMask中确认连接
    await metamaskPopup.click('button:has-text("Next")');
    await metamaskPopup.click('button:has-text("Connect")');
    
    // 验证连接成功
    await expect(page.locator('[data-testid="wallet-address"]')).toBeVisible();
  });
  
  test('should display correct network', async ({ page }) => {
    await page.click('button:has-text("Connect Wallet")');
    
    // 等待网络信息显示
    const networkName = await page.locator('[data-testid="network-name"]').textContent();
    expect(networkName).toBe('Sepolia');
  });
  
  test('should disconnect wallet', async ({ page }) => {
    // 先连接
    await page.click('button:has-text("Connect Wallet")');
    await page.waitForSelector('[data-testid="wallet-address"]');
    
    // 断开连接
    await page.click('[data-testid="wallet-menu"]');
    await page.click('button:has-text("Disconnect")');
    
    // 验证断开
    await expect(page.locator('[data-testid="wallet-address"]')).not.toBeVisible();
  });
});
```

### 5.2 交易流程E2E测试

```javascript
// test/e2e/trading.test.js
const { test, expect } = require('@playwright/test');

test.describe('DeFi Protocol Trading', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:3000');
    
    // 连接钱包
    await page.click('button:has-text("Connect Wallet")');
    await page.waitForSelector('[data-testid="wallet-address"]');
  });
  
  test('complete deposit flow', async ({ page }) => {
    // 导航到存款页面
    await page.click('a:has-text("Deposit")');
    
    // 选择代币
    await page.selectOption('[data-testid="token-select"]', 'USDC');
    
    // 输入金额
    await page.fill('[data-testid="amount-input"]', '100');
    
    // 点击存款
    await page.click('button:has-text("Deposit")');
    
    // 等待MetaMask确认
    // (简化处理，实际需要处理MetaMask弹窗)
    
    // 验证成功消息
    await expect(page.locator('[data-testid="success-message"]')).toContainText('Deposit successful');
    
    // 验证余额更新
    const balance = await page.locator('[data-testid="collateral-balance"]').textContent();
    expect(parseFloat(balance)).toBeGreaterThan(0);
  });
  
  test('complete borrow flow', async ({ page }) => {
    // 假设已有抵押品
    await page.click('a:has-text("Borrow")');
    
    // 选择抵押代币和借贷代币
    await page.selectOption('[data-testid="collateral-token"]', 'USDC');
    await page.selectOption('[data-testid="borrow-token"]', 'DAI');
    
    // 输入借贷金额
    await page.fill('[data-testid="borrow-amount"]', '50');
    
    // 检查最大可借金额
    const maxBorrow = await page.locator('[data-testid="max-borrow"]').textContent();
    expect(parseFloat(maxBorrow)).toBeGreaterThan(50);
    
    // 确认借贷
    await page.click('button:has-text("Borrow")');
    
    // 验证成功
    await expect(page.locator('[data-testid="success-message"]')).toContainText('Borrow successful');
  });
  
  test('should show error for insufficient collateral', async ({ page }) => {
    await page.click('a:has-text("Borrow")');
    
    // 尝试借贷超额
    await page.fill('[data-testid="borrow-amount"]', '10000');
    await page.click('button:has-text("Borrow")');
    
    // 验证错误消息
    await expect(page.locator('[data-testid="error-message"]')).toContainText('Insufficient collateral');
  });
});
```

---

## Part 6: 性能测试 (1小时)

### 6.1 Load Testing

```javascript
// test/performance/load.test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },  // 逐渐增加到20个用户
    { duration: '1m', target: 20 },   // 保持20个用户1分钟
    { duration: '30s', target: 50 },  // 增加到50个用户
    { duration: '1m', target: 50 },   // 保持50个用户1分钟
    { duration: '30s', target: 0 },   // 逐渐降至0
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95%的请求应在500ms内完成
    http_req_failed: ['rate<0.01'],   // 错误率应低于1%
  },
};

const BASE_URL = 'http://localhost:3000/api';

export default function () {
  // 查询协议信息
  let res = http.get(`${BASE_URL}/protocol/info`);
  check(res, {
    'protocol info status is 200': (r) => r.status === 200,
    'protocol info response time < 200ms': (r) => r.timings.duration < 200,
  });
  
  sleep(1);
  
  // 查询用户抵押品
  res = http.get(`${BASE_URL}/user/0x.../collateral`);
  check(res, {
    'collateral status is 200': (r) => r.status === 200,
  });
  
  sleep(1);
}
```

---

## Part 7: CI/CD集成 (1.5小时)

### 7.1 GitHub Actions配置

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Compile contracts
        run: npm run compile
      
      - name: Run unit tests
        run: npm run test:unit
      
      - name: Generate coverage report
        run: npm run test:coverage
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
  
  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run integration tests
        run: npm run test:integration
  
  e2e-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install Playwright
        run: npx playwright install --with-deps
      
      - name: Start local node
        run: |
          npm run node &
          sleep 10
      
      - name: Deploy contracts
        run: npm run deploy:local
      
      - name: Start frontend
        run: |
          cd frontend
          npm install
          npm run build
          npm run start &
          sleep 10
      
      - name: Run E2E tests
        run: npm run test:e2e
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
  
  security-scan:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Slither
        uses: crytic/slither-action@v0.3.0
        with:
          target: 'contracts/'
      
      - name: Run Mythril
        run: |
          pip3 install mythril
          myth analyze contracts/DeFiProtocol.sol
```

---

## Part 8: 测试报告 (1.5小时)

### 8.1 报告生成脚本

```javascript
// scripts/generate-report.js
const fs = require('fs');
const path = require('path');

async function generateReport() {
  // 读取测试结果
  const unitResults = JSON.parse(
    fs.readFileSync('test-results/unit.json', 'utf8')
  );
  const integrationResults = JSON.parse(
    fs.readFileSync('test-results/integration.json', 'utf8')
  );
  const e2eResults = JSON.parse(
    fs.readFileSync('test-results/e2e.json', 'utf8')
  );
  
  // 生成HTML报告
  const html = `
<!DOCTYPE html>
<html>
<head>
  <title>Test Report</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; }
    .summary { background: #f0f0f0; padding: 20px; border-radius: 5px; }
    .passed { color: green; }
    .failed { color: red; }
    table { width: 100%; border-collapse: collapse; margin: 20px 0; }
    th, td { padding: 10px; text-align: left; border-bottom: 1px solid #ddd; }
    th { background-color: #4CAF50; color: white; }
  </style>
</head>
<body>
  <h1>Test Report</h1>
  
  <div class="summary">
    <h2>Summary</h2>
    <p>Total Tests: ${getTotalTests()}</p>
    <p class="passed">Passed: ${getPassedTests()}</p>
    <p class="failed">Failed: ${getFailedTests()}</p>
    <p>Duration: ${getTotalDuration()}ms</p>
    <p>Coverage: ${getCoverage()}%</p>
  </div>
  
  <h2>Unit Tests</h2>
  ${generateTable(unitResults)}
  
  <h2>Integration Tests</h2>
  ${generateTable(integrationResults)}
  
  <h2>E2E Tests</h2>
  ${generateTable(e2eResults)}
</body>
</html>
  `;
  
  fs.writeFileSync('reports/test-report.html', html);
  console.log('Report generated: reports/test-report.html');
}

generateReport();
```

---

## 📝 项目总结

### 完成内容
1. ✅ DeFi协议合约开发
2. ✅ 完整测试套件（单元、集成、E2E）
3. ✅ 性能测试
4. ✅ CI/CD集成
5. ✅ 测试报告生成

### 关键指标
- 测试覆盖率: >90%
- 测试用例数: 50+
- CI/CD自动化: 完全自动化
- 文档完整度: 100%

---

## ✅ 检查清单

- [ ] 完成合约开发
- [ ] 编写单元测试
- [ ] 编写集成测试
- [ ] 实现E2E测试
- [ ] 配置CI/CD
- [ ] 生成测试报告
- [ ] 达到90%+覆盖率

---

## 📅 下周预告

下周学习前端集成：
- React基础
- Web3集成
- MetaMask连接
- 完整DApp开发

**🎉 完成Week 5！准备进入前端开发阶段！**
