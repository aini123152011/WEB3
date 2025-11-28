# Week 4 - Day 1: Hardhat测试框架深入 - 上

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 掌握Hardhat测试架构
- ✅ 学习Chai断言库
- ✅ 理解测试夹具(Fixtures)
- ✅ 掌握时间操纵技巧
- ✅ 学习事件测试方法
- ✅ 理解快照与回滚

---

## Part 1: Hardhat测试基础架构 (1.5小时)

### 1.1 测试环境配置

#### 安装依赖

```bash
npm install --save-dev @nomicfoundation/hardhat-chai-matchers
npm install --save-dev @nomicfoundation/hardhat-network-helpers
npm install --save-dev chai
npm install --save-dev @nomiclabs/hardhat-ethers
npm install --save-dev ethers
```

#### hardhat.config.js配置

```javascript
require("@nomicfoundation/hardhat-toolbox");

/** @type import('hardhat/config').HardhatUserConfig */
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
      },
      accounts: {
        count: 20,
        accountsBalance: "10000000000000000000000" // 10000 ETH
      }
    }
  },
  mocha: {
    timeout: 40000
  }
};
```

---

### 1.2 测试文件结构

#### 基础测试模板

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");

describe("TokenContract", function () {
  // 测试夹具
  async function deployTokenFixture() {
    const [owner, addr1, addr2] = await ethers.getSigners();
    
    const Token = await ethers.getContractFactory("Token");
    const token = await Token.deploy("MyToken", "MTK", 1000000);
    await token.waitForDeployment();
    
    return { token, owner, addr1, addr2 };
  }
  
  describe("Deployment", function () {
    it("Should set the right owner", async function () {
      const { token, owner } = await loadFixture(deployTokenFixture);
      expect(await token.owner()).to.equal(owner.address);
    });
    
    it("Should assign total supply to owner", async function () {
      const { token, owner } = await loadFixture(deployTokenFixture);
      const ownerBalance = await token.balanceOf(owner.address);
      expect(await token.totalSupply()).to.equal(ownerBalance);
    });
  });
  
  describe("Transactions", function () {
    it("Should transfer tokens between accounts", async function () {
      const { token, owner, addr1, addr2 } = await loadFixture(deployTokenFixture);
      
      await expect(token.transfer(addr1.address, 50))
        .to.changeTokenBalances(token, [owner, addr1], [-50, 50]);
        
      await expect(token.connect(addr1).transfer(addr2.address, 50))
        .to.changeTokenBalances(token, [addr1, addr2], [-50, 50]);
    });
    
    it("Should fail if sender doesn't have enough tokens", async function () {
      const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
      const initialOwnerBalance = await token.balanceOf(owner.address);
      
      await expect(
        token.connect(addr1).transfer(owner.address, 1)
      ).to.be.revertedWith("Not enough tokens");
    });
  });
});
```

---

## Part 2: Chai断言完全指南 (2小时)

### 2.1 基础断言

```javascript
describe("Basic Assertions", function () {
  it("Should test equality", async function () {
    const { token } = await loadFixture(deployTokenFixture);
    
    // 相等断言
    expect(await token.name()).to.equal("MyToken");
    expect(await token.totalSupply()).to.equal(1000000);
    
    // 深度相等（对象/数组）
    const array1 = [1, 2, 3];
    const array2 = [1, 2, 3];
    expect(array1).to.deep.equal(array2);
  });
  
  it("Should test types", async function () {
    const { token, owner } = await loadFixture(deployTokenFixture);
    
    expect(owner.address).to.be.a("string");
    expect(await token.totalSupply()).to.be.a("bigint");
  });
  
  it("Should test boolean values", async function () {
    const { token, owner } = await loadFixture(deployTokenFixture);
    
    expect(await token.paused()).to.be.false;
    expect(await token.hasRole(ADMIN_ROLE, owner.address)).to.be.true;
  });
  
  it("Should test inclusion", async function () {
    const roles = ["ADMIN", "MINTER", "BURNER"];
    
    expect(roles).to.include("ADMIN");
    expect(roles).to.have.members(["ADMIN", "MINTER", "BURNER"]);
    expect(roles).to.have.lengthOf(3);
  });
});
```

### 2.2 BigNumber断言

```javascript
describe("BigNumber Assertions", function () {
  it("Should compare BigNumbers", async function () {
    const { token, owner } = await loadFixture(deployTokenFixture);
    
    const balance = await token.balanceOf(owner.address);
    const totalSupply = await token.totalSupply();
    
    // 数值比较
    expect(balance).to.equal(totalSupply);
    expect(balance).to.be.above(0);
    expect(balance).to.be.at.least(1000000);
    expect(balance).to.be.below(2000000);
    expect(balance).to.be.at.most(1000000);
  });
  
  it("Should handle precision", async function () {
    const { token } = await loadFixture(deployTokenFixture);
    
    const decimals = await token.decimals();
    const oneToken = ethers.parseUnits("1", decimals);
    
    expect(oneToken).to.equal(ethers.parseUnits("1.0", decimals));
  });
});
```

### 2.3 交易断言

```javascript
describe("Transaction Assertions", function () {
  it("Should test transaction success", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    // 检查交易不回滚
    await expect(
      token.transfer(addr1.address, 100)
    ).to.not.be.reverted;
  });
  
  it("Should test revert messages", async function () {
    const { token, addr1 } = await loadFixture(deployTokenFixture);
    
    // 精确消息
    await expect(
      token.connect(addr1).transfer(owner.address, 1)
    ).to.be.revertedWith("Not enough tokens");
    
    // 包含消息
    await expect(
      token.connect(addr1).transfer(owner.address, 1)
    ).to.be.revertedWith(/not enough/i);
  });
  
  it("Should test custom errors", async function () {
    const { token, addr1 } = await loadFixture(deployTokenFixture);
    
    await expect(
      token.connect(addr1).burn(1000)
    ).to.be.revertedWithCustomError(token, "InsufficientBalance")
      .withArgs(addr1.address, 0, 1000);
  });
  
  it("Should test panic codes", async function () {
    const { math } = await loadFixture(deployMathFixture);
    
    // 除零错误 (panic code 0x12)
    await expect(
      math.divide(10, 0)
    ).to.be.revertedWithPanic(0x12);
    
    // 溢出错误 (panic code 0x11)
    await expect(
      math.add(ethers.MaxUint256, 1)
    ).to.be.revertedWithPanic(0x11);
  });
});
```

### 2.4 余额变化断言

```javascript
describe("Balance Change Assertions", function () {
  it("Should test ETH balance changes", async function () {
    const [owner, addr1] = await ethers.getSigners();
    
    // 单个账户
    await expect(() =>
      owner.sendTransaction({
        to: addr1.address,
        value: ethers.parseEther("1.0")
      })
    ).to.changeEtherBalance(owner, ethers.parseEther("-1.0"));
    
    // 多个账户
    await expect(
      owner.sendTransaction({
        to: addr1.address,
        value: ethers.parseEther("1.0")
      })
    ).to.changeEtherBalances(
      [owner, addr1],
      [ethers.parseEther("-1.0"), ethers.parseEther("1.0")]
    );
  });
  
  it("Should test token balance changes", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    await expect(
      token.transfer(addr1.address, 100)
    ).to.changeTokenBalances(
      token,
      [owner, addr1],
      [-100, 100]
    );
  });
  
  it("Should test balance changes with transactions", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    const tx = await token.transfer(addr1.address, 100);
    
    await expect(tx).to.changeTokenBalance(token, owner, -100);
    await expect(tx).to.changeTokenBalance(token, addr1, 100);
  });
});
```

---

## Part 3: 事件测试 (1.5小时)

### 3.1 基础事件测试

```javascript
describe("Event Testing", function () {
  it("Should emit Transfer event", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    await expect(token.transfer(addr1.address, 100))
      .to.emit(token, "Transfer")
      .withArgs(owner.address, addr1.address, 100);
  });
  
  it("Should emit multiple events", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    const tx = token.approve(addr1.address, 100);
    
    await expect(tx)
      .to.emit(token, "Approval")
      .withArgs(owner.address, addr1.address, 100);
  });
  
  it("Should not emit events", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    // 转账0不应触发事件
    await expect(
      token.transfer(addr1.address, 0)
    ).to.not.emit(token, "Transfer");
  });
});
```

### 3.2 高级事件测试

```javascript
describe("Advanced Event Testing", function () {
  it("Should test indexed parameters", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    const tx = await token.transfer(addr1.address, 100);
    const receipt = await tx.wait();
    
    const event = receipt.logs.find(
      log => log.fragment && log.fragment.name === "Transfer"
    );
    
    expect(event.args.from).to.equal(owner.address);
    expect(event.args.to).to.equal(addr1.address);
    expect(event.args.value).to.equal(100);
  });
  
  it("Should test multiple events in order", async function () {
    const { nft, owner, addr1 } = await loadFixture(deployNFTFixture);
    
    const tx = await nft.mintAndTransfer(addr1.address, 1);
    
    // 按顺序检查多个事件
    await expect(tx)
      .to.emit(nft, "Mint")
      .withArgs(owner.address, 1)
      .to.emit(nft, "Transfer")
      .withArgs(owner.address, addr1.address, 1);
  });
  
  it("Should filter and parse events", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    // 发送多笔交易
    await token.transfer(addr1.address, 100);
    await token.transfer(addr1.address, 200);
    await token.transfer(addr1.address, 300);
    
    // 过滤事件
    const filter = token.filters.Transfer(owner.address, addr1.address);
    const events = await token.queryFilter(filter);
    
    expect(events).to.have.lengthOf(3);
    expect(events[0].args.value).to.equal(100);
    expect(events[1].args.value).to.equal(200);
    expect(events[2].args.value).to.equal(300);
  });
});
```

### 3.3 事件监听器

```javascript
describe("Event Listeners", function () {
  it("Should listen to events", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    let eventEmitted = false;
    let eventArgs;
    
    token.on("Transfer", (from, to, value) => {
      eventEmitted = true;
      eventArgs = { from, to, value };
    });
    
    await token.transfer(addr1.address, 100);
    
    // 等待事件
    await new Promise((resolve) => setTimeout(resolve, 100));
    
    expect(eventEmitted).to.be.true;
    expect(eventArgs.from).to.equal(owner.address);
    expect(eventArgs.to).to.equal(addr1.address);
    expect(eventArgs.value).to.equal(100);
    
    token.removeAllListeners("Transfer");
  });
  
  it("Should test event with once", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    const promise = new Promise((resolve) => {
      token.once("Transfer", (from, to, value) => {
        resolve({ from, to, value });
      });
    });
    
    await token.transfer(addr1.address, 100);
    
    const eventArgs = await promise;
    expect(eventArgs.value).to.equal(100);
  });
});
```

---

## Part 4: 测试夹具(Fixtures) (1小时)

### 4.1 基础夹具

```javascript
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");

describe("Fixtures", function () {
  // 简单夹具
  async function deployBasicFixture() {
    const [owner, user1, user2] = await ethers.getSigners();
    return { owner, user1, user2 };
  }
  
  // 复杂夹具
  async function deployCompleteFixture() {
    const [owner, user1, user2] = await ethers.getSigners();
    
    // 部署代币
    const Token = await ethers.getContractFactory("Token");
    const token = await Token.deploy("MyToken", "MTK", 1000000);
    await token.waitForDeployment();
    
    // 部署NFT
    const NFT = await ethers.getContractFactory("NFT");
    const nft = await NFT.deploy("MyNFT", "MNFT");
    await nft.waitForDeployment();
    
    // 分配代币
    await token.transfer(user1.address, 10000);
    await token.transfer(user2.address, 10000);
    
    return { token, nft, owner, user1, user2 };
  }
  
  it("Should use fixture", async function () {
    const { token, owner } = await loadFixture(deployCompleteFixture);
    expect(await token.owner()).to.equal(owner.address);
  });
});
```

### 4.2 嵌套夹具

```javascript
describe("Nested Fixtures", function () {
  async function deployTokensFixture() {
    const Token = await ethers.getContractFactory("Token");
    const tokenA = await Token.deploy("TokenA", "TKA", 1000000);
    const tokenB = await Token.deploy("TokenB", "TKB", 1000000);
    
    await tokenA.waitForDeployment();
    await tokenB.waitForDeployment();
    
    return { tokenA, tokenB };
  }
  
  async function deploySwapFixture() {
    const { tokenA, tokenB } = await loadFixture(deployTokensFixture);
    
    const Swap = await ethers.getContractFactory("Swap");
    const swap = await Swap.deploy(
      await tokenA.getAddress(),
      await tokenB.getAddress()
    );
    await swap.waitForDeployment();
    
    return { tokenA, tokenB, swap };
  }
  
  it("Should use nested fixture", async function () {
    const { swap, tokenA, tokenB } = await loadFixture(deploySwapFixture);
    
    expect(await swap.tokenA()).to.equal(await tokenA.getAddress());
    expect(await swap.tokenB()).to.equal(await tokenB.getAddress());
  });
});
```

### 4.3 参数化夹具

```javascript
describe("Parameterized Fixtures", function () {
  function createDeployFixture(name, symbol, supply) {
    return async function () {
      const Token = await ethers.getContractFactory("Token");
      const token = await Token.deploy(name, symbol, supply);
      await token.waitForDeployment();
      return { token };
    };
  }
  
  it("Should use parameterized fixture", async function () {
    const fixture = createDeployFixture("MyToken", "MTK", 1000000);
    const { token } = await loadFixture(fixture);
    
    expect(await token.name()).to.equal("MyToken");
    expect(await token.symbol()).to.equal("MTK");
    expect(await token.totalSupply()).to.equal(1000000);
  });
});
```

---

## 📝 今日作业

### 作业1: ERC20代币完整测试

创建一个ERC20代币并编写完整测试套件：
1. 部署测试
2. 转账测试（成功/失败）
3. 授权测试
4. 事件测试
5. 边界条件测试

### 作业2: NFT铸造测试

创建NFT合约并测试：
1. 铸造功能
2. 转账功能
3. 元数据功能
4. 所有事件
5. 访问控制

### 作业3: 复杂交互测试

创建多合约交互场景：
1. 代币兑换合约
2. 测试完整兑换流程
3. 测试边界条件
4. 测试异常情况

---

## ✅ 检查清单

- [ ] 理解Chai断言语法
- [ ] 掌握事件测试方法
- [ ] 会使用测试夹具
- [ ] 完成所有作业
- [ ] 测试覆盖率>80%

---

## 🎯 常见问题FAQ

### Q1: 如何测试require失败？

```javascript
await expect(
  contract.function()
).to.be.revertedWith("Error message");
```

### Q2: 如何测试自定义错误？

```javascript
await expect(
  contract.function()
).to.be.revertedWithCustomError(contract, "ErrorName")
  .withArgs(arg1, arg2);
```

### Q3: 如何测试多个账户余额变化？

```javascript
await expect(tx).to.changeTokenBalances(
  token,
  [account1, account2],
  [change1, change2]
);
```

### Q4: loadFixture有什么优势？

- 自动快照和回滚
- 提高测试速度
- 确保测试隔离
- 避免重复代码

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

明天学习Hardhat测试框架深入 - 下：
- 时间操纵技巧
- 网络快照与回滚
- Gas优化测试
- 并行测试技巧

**预习内容**: 了解Hardhat Network的时间控制功能

**🎉 完成Day 1！明天见！**
