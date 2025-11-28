# Week 5 - Day 1: Web3自动化测试基础 - 上

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 理解Web3自动化测试概念
- ✅ 掌握Ethers.js测试框架
- ✅ 学习合约交互测试
- ✅ 理解事件监听测试
- ✅ 掌握异步测试技巧
- ✅ 学习错误处理测试

---

## Part 1: Web3测试环境搭建 (1.5小时)

### 1.1 安装依赖

```bash
npm install --save-dev ethers hardhat @nomicfoundation/hardhat-ethers
npm install --save-dev @nomicfoundation/hardhat-chai-matchers chai
npm install --save-dev @types/mocha @types/chai
```

### 1.2 项目配置

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
  solidity: {
    version: "0.8.20",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200
      }
    }
  },
  networks: {
    hardhat: {
      chainId: 31337,
      mining: {
        auto: true,
        interval: 0
      }
    },
    localhost: {
      url: "http://127.0.0.1:8545"
    },
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : []
    }
  },
  mocha: {
    timeout: 40000
  }
};
```

---

## Part 2: 基础合约交互测试 (2小时)

### 2.1 简单合约测试

```javascript
// test/Token.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Token Contract", function () {
  let Token, token, owner, addr1, addr2;
  
  // 在每个测试前部署合约
  beforeEach(async function () {
    [owner, addr1, addr2] = await ethers.getSigners();
    
    Token = await ethers.getContractFactory("Token");
    token = await Token.deploy("MyToken", "MTK", ethers.parseEther("1000000"));
    await token.waitForDeployment();
  });
  
  describe("Deployment", function () {
    it("Should set the right owner", async function () {
      expect(await token.owner()).to.equal(owner.address);
    });
    
    it("Should assign the total supply to the owner", async function () {
      const ownerBalance = await token.balanceOf(owner.address);
      expect(await token.totalSupply()).to.equal(ownerBalance);
    });
    
    it("Should have correct name and symbol", async function () {
      expect(await token.name()).to.equal("MyToken");
      expect(await token.symbol()).to.equal("MTK");
    });
  });
  
  describe("Transactions", function () {
    it("Should transfer tokens between accounts", async function () {
      // Transfer 50 tokens from owner to addr1
      await token.transfer(addr1.address, 50);
      const addr1Balance = await token.balanceOf(addr1.address);
      expect(addr1Balance).to.equal(50);
      
      // Transfer 50 tokens from addr1 to addr2
      await token.connect(addr1).transfer(addr2.address, 50);
      const addr2Balance = await token.balanceOf(addr2.address);
      expect(addr2Balance).to.equal(50);
    });
    
    it("Should fail if sender doesn't have enough tokens", async function () {
      const initialOwnerBalance = await token.balanceOf(owner.address);
      
      // Try to send 1 token from addr1 (0 tokens) to owner
      await expect(
        token.connect(addr1).transfer(owner.address, 1)
      ).to.be.revertedWith("Insufficient balance");
      
      // Owner balance shouldn't have changed
      expect(await token.balanceOf(owner.address)).to.equal(
        initialOwnerBalance
      );
    });
    
    it("Should update balances after transfers", async function () {
      const initialOwnerBalance = await token.balanceOf(owner.address);
      
      // Transfer 100 tokens from owner to addr1
      await token.transfer(addr1.address, 100);
      
      // Transfer another 50 tokens from owner to addr2
      await token.transfer(addr2.address, 50);
      
      // Check balances
      const finalOwnerBalance = await token.balanceOf(owner.address);
      expect(finalOwnerBalance).to.equal(initialOwnerBalance - 150n);
      
      const addr1Balance = await token.balanceOf(addr1.address);
      expect(addr1Balance).to.equal(100);
      
      const addr2Balance = await token.balanceOf(addr2.address);
      expect(addr2Balance).to.equal(50);
    });
  });
});
```

### 2.2 复杂合约交互

```javascript
describe("DEX Contract", function () {
  let DEX, dex, TokenA, tokenA, TokenB, tokenB;
  let owner, trader1, trader2;
  
  beforeEach(async function () {
    [owner, trader1, trader2] = await ethers.getSigners();
    
    // Deploy tokens
    TokenA = await ethers.getContractFactory("Token");
    tokenA = await TokenA.deploy("TokenA", "TKA", ethers.parseEther("1000000"));
    await tokenA.waitForDeployment();
    
    TokenB = await ethers.getContractFactory("Token");
    tokenB = await TokenB.deploy("TokenB", "TKB", ethers.parseEther("1000000"));
    await tokenB.waitForDeployment();
    
    // Deploy DEX
    DEX = await ethers.getContractFactory("DEX");
    dex = await DEX.deploy(
      await tokenA.getAddress(),
      await tokenB.getAddress()
    );
    await dex.waitForDeployment();
    
    // Distribute tokens
    await tokenA.transfer(trader1.address, ethers.parseEther("10000"));
    await tokenB.transfer(trader1.address, ethers.parseEther("10000"));
  });
  
  describe("Liquidity Management", function () {
    it("Should add liquidity correctly", async function () {
      const amountA = ethers.parseEther("1000");
      const amountB = ethers.parseEther("1000");
      
      await tokenA.approve(await dex.getAddress(), amountA);
      await tokenB.approve(await dex.getAddress(), amountB);
      
      await dex.addLiquidity(amountA, amountB);
      
      expect(await dex.reserveA()).to.equal(amountA);
      expect(await dex.reserveB()).to.equal(amountB);
    });
    
    it("Should remove liquidity correctly", async function () {
      // Add liquidity first
      const amountA = ethers.parseEther("1000");
      const amountB = ethers.parseEther("1000");
      
      await tokenA.approve(await dex.getAddress(), amountA);
      await tokenB.approve(await dex.getAddress(), amountB);
      await dex.addLiquidity(amountA, amountB);
      
      const liquidity = await dex.balanceOf(owner.address);
      
      // Remove liquidity
      const [returnedA, returnedB] = await dex.removeLiquidity(liquidity / 2n);
      
      expect(returnedA).to.be.closeTo(amountA / 2n, ethers.parseEther("1"));
      expect(returnedB).to.be.closeTo(amountB / 2n, ethers.parseEther("1"));
    });
  });
  
  describe("Token Swap", function () {
    beforeEach(async function () {
      // Add initial liquidity
      await tokenA.approve(await dex.getAddress(), ethers.parseEther("10000"));
      await tokenB.approve(await dex.getAddress(), ethers.parseEther("10000"));
      await dex.addLiquidity(
        ethers.parseEther("10000"),
        ethers.parseEther("10000")
      );
    });
    
    it("Should swap tokens A for B", async function () {
      await tokenA.connect(trader1).approve(
        await dex.getAddress(),
        ethers.parseEther("100")
      );
      
      const balanceBBefore = await tokenB.balanceOf(trader1.address);
      
      await dex.connect(trader1).swap(
        ethers.parseEther("100"),
        0,
        true // A to B
      );
      
      const balanceBAfter = await tokenB.balanceOf(trader1.address);
      expect(balanceBAfter).to.be.gt(balanceBBefore);
    });
    
    it("Should calculate correct output amount", async function () {
      const amountIn = ethers.parseEther("100");
      const expectedOut = await dex.getAmountOut(amountIn, true);
      
      await tokenA.connect(trader1).approve(
        await dex.getAddress(),
        amountIn
      );
      
      const balanceBBefore = await tokenB.balanceOf(trader1.address);
      
      await dex.connect(trader1).swap(amountIn, 0, true);
      
      const balanceBAfter = await tokenB.balanceOf(trader1.address);
      const actualOut = balanceBAfter - balanceBBefore;
      
      expect(actualOut).to.be.closeTo(expectedOut, ethers.parseEther("0.1"));
    });
  });
});
```

---

## Part 3: 事件测试 (1.5小时)

### 3.1 基础事件测试

```javascript
describe("Event Testing", function () {
  let token, owner, addr1;
  
  beforeEach(async function () {
    [owner, addr1] = await ethers.getSigners();
    const Token = await ethers.getContractFactory("Token");
    token = await Token.deploy("Test", "TST", ethers.parseEther("1000000"));
    await token.waitForDeployment();
  });
  
  it("Should emit Transfer event on transfer", async function () {
    await expect(token.transfer(addr1.address, 100))
      .to.emit(token, "Transfer")
      .withArgs(owner.address, addr1.address, 100);
  });
  
  it("Should emit Approval event on approve", async function () {
    await expect(token.approve(addr1.address, 1000))
      .to.emit(token, "Approval")
      .withArgs(owner.address, addr1.address, 1000);
  });
  
  it("Should emit multiple events in order", async function () {
    const tx = token.batchTransfer(
      [addr1.address, addr2.address],
      [100, 200]
    );
    
    await expect(tx)
      .to.emit(token, "Transfer")
      .withArgs(owner.address, addr1.address, 100)
      .to.emit(token, "Transfer")
      .withArgs(owner.address, addr2.address, 200);
  });
});
```

### 3.2 事件监听

```javascript
describe("Event Listeners", function () {
  it("Should listen to events", async function () {
    const [owner, addr1] = await ethers.getSigners();
    const Token = await ethers.getContractFactory("Token");
    const token = await Token.deploy("Test", "TST", ethers.parseEther("1000"));
    await token.waitForDeployment();
    
    // Set up event listener
    const transferPromise = new Promise((resolve) => {
      token.once("Transfer", (from, to, value) => {
        resolve({ from, to, value });
      });
    });
    
    // Trigger event
    await token.transfer(addr1.address, 100);
    
    // Wait for event
    const event = await transferPromise;
    expect(event.from).to.equal(owner.address);
    expect(event.to).to.equal(addr1.address);
    expect(event.value).to.equal(100);
  });
  
  it("Should filter events by parameters", async function () {
    const [owner, addr1, addr2] = await ethers.getSigners();
    const Token = await ethers.getContractFactory("Token");
    const token = await Token.deploy("Test", "TST", ethers.parseEther("1000"));
    await token.waitForDeployment();
    
    // Create filter for transfers to addr1
    const filter = token.filters.Transfer(null, addr1.address);
    
    // Make multiple transfers
    await token.transfer(addr1.address, 100);
    await token.transfer(addr2.address, 200);
    await token.transfer(addr1.address, 300);
    
    // Query filtered events
    const events = await token.queryFilter(filter);
    
    expect(events).to.have.lengthOf(2);
    expect(events[0].args.value).to.equal(100);
    expect(events[1].args.value).to.equal(300);
  });
});
```

---

## Part 4: 异步测试技巧 (1.5小时)

### 4.1 Promise处理

```javascript
describe("Async Testing", function () {
  it("Should handle multiple async operations", async function () {
    const [owner, addr1, addr2] = await ethers.getSigners();
    const Token = await ethers.getContractFactory("Token");
    const token = await Token.deploy("Test", "TST", ethers.parseEther("1000"));
    await token.waitForDeployment();
    
    // Execute multiple operations in parallel
    await Promise.all([
      token.transfer(addr1.address, 100),
      token.transfer(addr2.address, 200)
    ]);
    
    // Verify results
    const [balance1, balance2] = await Promise.all([
      token.balanceOf(addr1.address),
      token.balanceOf(addr2.address)
    ]);
    
    expect(balance1).to.equal(100);
    expect(balance2).to.equal(200);
  });
  
  it("Should handle sequential async operations", async function () {
    const Token = await ethers.getContractFactory("Token");
    const token = await Token.deploy("Test", "TST", ethers.parseEther("1000"));
    await token.waitForDeployment();
    
    const [owner, addr1] = await ethers.getSigners();
    
    // Approve
    await token.approve(addr1.address, 1000);
    
    // TransferFrom
    await token.connect(addr1).transferFrom(
      owner.address,
      addr1.address,
      500
    );
    
    expect(await token.balanceOf(addr1.address)).to.equal(500);
  });
});
```

### 4.2 等待交易确认

```javascript
describe("Transaction Confirmation", function () {
  it("Should wait for transaction confirmation", async function () {
    const [owner, addr1] = await ethers.getSigners();
    const Token = await ethers.getContractFactory("Token");
    const token = await Token.deploy("Test", "TST", ethers.parseEther("1000"));
    await token.waitForDeployment();
    
    // Send transaction
    const tx = await token.transfer(addr1.address, 100);
    
    // Wait for confirmation
    const receipt = await tx.wait();
    
    // Check receipt
    expect(receipt.status).to.equal(1); // Success
    expect(receipt.from).to.equal(owner.address);
    
    // Verify balance
    expect(await token.balanceOf(addr1.address)).to.equal(100);
  });
  
  it("Should wait for multiple confirmations", async function () {
    const [owner, addr1] = await ethers.getSigners();
    const Token = await ethers.getContractFactory("Token");
    const token = await Token.deploy("Test", "TST", ethers.parseEther("1000"));
    await token.waitForDeployment();
    
    const tx = await token.transfer(addr1.address, 100);
    
    // Wait for 2 confirmations
    const receipt = await tx.wait(2);
    
    expect(receipt.confirmations).to.be.gte(2);
  });
});
```

---

## 📝 今日作业

### 作业1: ERC20完整测试

为ERC20代币编写完整测试套件：
1. 部署测试
2. 转账测试
3. 授权测试
4. 事件测试
5. 错误处理测试

### 作业2: NFT合约测试

为NFT合约编写测试：
1. 铸造测试
2. 转账测试
3. 元数据测试
4. 批量操作测试

### 作业3: DEX交互测试

测试DEX完整流程：
1. 添加流动性
2. 代币交换
3. 移除流动性
4. 价格计算

---

## ✅ 检查清单

- [ ] 理解异步测试
- [ ] 掌握事件测试
- [ ] 会处理Promise
- [ ] 完成所有作业
- [ ] 测试通过率100%

---

## 🎯 常见问题FAQ

### Q1: 如何测试require失败？

```javascript
await expect(
  contract.function()
).to.be.revertedWith("Error message");
```

### Q2: 如何获取事件参数？

```javascript
const tx = await contract.function();
const receipt = await tx.wait();
const event = receipt.events[0];
const args = event.args;
```

### Q3: 如何处理BigNumber？

```javascript
const value = ethers.parseEther("1.0");
expect(balance).to.equal(value);
```

### Q4: 测试超时怎么办？

```javascript
// hardhat.config.js
mocha: {
  timeout: 100000
}
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

---

## 📅 明日预告

明天学习Web3自动化测试基础 - 下：
- 网络模拟测试
- 时间旅行测试
- Gas估算测试
- 多签钱包测试

**🎉 完成Day 1！明天见！**
