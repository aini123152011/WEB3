# Week 4 - Day 2: Hardhat测试框架深入 - 下

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 学习目标

- ✅ 掌握时间操纵技巧
- ✅ 学习网络快照与回滚
- ✅ 理解Gas优化测试
- ✅ 掌握覆盖率测试
- ✅ 学习模拟与Stub技术
- ✅ 理解并行测试优化

---

## Part 1: 时间操纵技巧 (2小时)

### 1.1 基础时间控制

```javascript
const { time } = require("@nomicfoundation/hardhat-network-helpers");

describe("Time Manipulation", function () {
  it("Should increase time", async function () {
    const { staking } = await loadFixture(deployStakingFixture);
    
    // 获取当前时间
    const currentTime = await time.latest();
    console.log("Current time:", currentTime);
    
    // 增加1天
    await time.increase(24 * 60 * 60);
    
    const newTime = await time.latest();
    expect(newTime).to.be.above(currentTime);
  });
  
  it("Should set specific time", async function () {
    // 设置到特定时间戳
    const targetTime = 1672531200; // 2023-01-01 00:00:00 UTC
    await time.increaseTo(targetTime);
    
    expect(await time.latest()).to.equal(targetTime);
  });
  
  it("Should mine blocks", async function () {
    const currentBlock = await ethers.provider.getBlockNumber();
    
    // 挖10个区块
    await time.mine(10);
    
    const newBlock = await ethers.provider.getBlockNumber();
    expect(newBlock).to.equal(currentBlock + 10);
  });
  
  it("Should mine blocks with interval", async function () {
    // 每10秒挖一个区块，总共挖5个
    await time.mine(5, { interval: 10 });
    
    const latestTime = await time.latest();
    console.log("Time after mining:", latestTime);
  });
});
```

### 1.2 时间旅行测试场景

```javascript
describe("Time-based Testing", function () {
  // 测试质押奖励
  it("Should calculate staking rewards over time", async function () {
    const { staking, rewardToken, user } = await loadFixture(deployStakingFixture);
    
    // 质押1000代币
    await staking.connect(user).stake(1000);
    
    // 快进7天
    await time.increase(7 * 24 * 60 * 60);
    
    // 计算奖励
    const reward = await staking.calculateReward(user.address);
    expect(reward).to.be.above(0);
    
    // 再快进7天
    await time.increase(7 * 24 * 60 * 60);
    
    const reward2 = await staking.calculateReward(user.address);
    expect(reward2).to.be.above(reward);
  });
  
  // 测试锁定期
  it("Should enforce lock period", async function () {
    const { vesting, beneficiary } = await loadFixture(deployVestingFixture);
    
    // 尝试立即提取（应该失败）
    await expect(
      vesting.connect(beneficiary).release()
    ).to.be.revertedWith("Still locked");
    
    // 快进到锁定期结束
    const lockEnd = await vesting.lockEnd();
    await time.increaseTo(lockEnd);
    
    // 现在应该可以提取
    await expect(
      vesting.connect(beneficiary).release()
    ).to.not.be.reverted;
  });
  
  // 测试线性释放
  it("Should release tokens linearly", async function () {
    const { vesting, beneficiary, totalAmount } = await loadFixture(deployVestingFixture);
    
    const startTime = await vesting.startTime();
    const duration = await vesting.duration();
    
    // 快进到25%时间点
    await time.increaseTo(startTime + duration / 4n);
    
    await vesting.connect(beneficiary).release();
    const released25 = await vesting.released();
    expect(released25).to.be.closeTo(totalAmount / 4n, totalAmount / 100n); // 允许1%误差
    
    // 快进到50%时间点
    await time.increaseTo(startTime + duration / 2n);
    
    await vesting.connect(beneficiary).release();
    const released50 = await vesting.released();
    expect(released50).to.be.closeTo(totalAmount / 2n, totalAmount / 100n);
    
    // 快进到100%时间点
    await time.increaseTo(startTime + duration);
    
    await vesting.connect(beneficiary).release();
    const released100 = await vesting.released();
    expect(released100).to.equal(totalAmount);
  });
});
```

### 1.3 区块时间戳测试

```javascript
describe("Block Timestamp Testing", function () {
  it("Should test time-sensitive operations", async function () {
    const { auction } = await loadFixture(deployAuctionFixture);
    
    // 获取拍卖开始时间
    const startTime = await auction.startTime();
    const duration = await auction.duration();
    
    // 在开始前尝试出价（应失败）
    await time.increaseTo(startTime - 1n);
    await expect(
      auction.bid({ value: ethers.parseEther("1") })
    ).to.be.revertedWith("Auction not started");
    
    // 在拍卖期间出价
    await time.increaseTo(startTime + 100n);
    await expect(
      auction.bid({ value: ethers.parseEther("1") })
    ).to.not.be.reverted;
    
    // 在结束后尝试出价（应失败）
    await time.increaseTo(startTime + duration + 1n);
    await expect(
      auction.bid({ value: ethers.parseEther("2") })
    ).to.be.revertedWith("Auction ended");
  });
  
  it("Should handle multiple time-based conditions", async function () {
    const { lottery } = await loadFixture(deployLotteryFixture);
    
    // 阶段1: 购买期
    const buyStart = await lottery.buyStart();
    await time.increaseTo(buyStart);
    await lottery.buyTicket({ value: ethers.parseEther("0.1") });
    
    // 阶段2: 抽奖期
    const drawTime = await lottery.drawTime();
    await time.increaseTo(drawTime);
    await lottery.draw();
    
    // 阶段3: 领奖期
    const claimStart = await lottery.claimStart();
    await time.increaseTo(claimStart);
    await lottery.claim();
  });
});
```

---

## Part 2: 网络快照与回滚 (1.5小时)

### 2.1 基础快照功能

```javascript
describe("Snapshot and Revert", function () {
  let snapshotId;
  
  beforeEach(async function () {
    // 每个测试前创建快照
    snapshotId = await ethers.provider.send("evm_snapshot", []);
  });
  
  afterEach(async function () {
    // 每个测试后恢复快照
    await ethers.provider.send("evm_revert", [snapshotId]);
  });
  
  it("Should modify state", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    await token.transfer(addr1.address, 1000);
    expect(await token.balanceOf(addr1.address)).to.equal(1000);
  });
  
  it("Should have clean state", async function () {
    const { token, addr1 } = await loadFixture(deployTokenFixture);
    
    // 由于快照恢复，余额应为0
    expect(await token.balanceOf(addr1.address)).to.equal(0);
  });
});
```

### 2.2 高级快照技巧

```javascript
describe("Advanced Snapshot", function () {
  it("Should use nested snapshots", async function () {
    const { token, owner, addr1, addr2 } = await loadFixture(deployTokenFixture);
    
    // 快照1
    const snap1 = await ethers.provider.send("evm_snapshot", []);
    await token.transfer(addr1.address, 1000);
    expect(await token.balanceOf(addr1.address)).to.equal(1000);
    
    // 快照2（在快照1基础上）
    const snap2 = await ethers.provider.send("evm_snapshot", []);
    await token.transfer(addr2.address, 500);
    expect(await token.balanceOf(addr2.address)).to.equal(500);
    
    // 恢复到快照2
    await ethers.provider.send("evm_revert", [snap2]);
    expect(await token.balanceOf(addr1.address)).to.equal(1000);
    expect(await token.balanceOf(addr2.address)).to.equal(0);
    
    // 恢复到快照1
    await ethers.provider.send("evm_revert", [snap1]);
    expect(await token.balanceOf(addr1.address)).to.equal(0);
  });
  
  it("Should snapshot complex state", async function () {
    const { dex, tokenA, tokenB } = await loadFixture(deployDexFixture);
    
    const snap = await ethers.provider.send("evm_snapshot", []);
    
    // 执行复杂操作
    await tokenA.approve(await dex.getAddress(), 1000);
    await tokenB.approve(await dex.getAddress(), 1000);
    await dex.addLiquidity(1000, 1000);
    await dex.swap(100, 0);
    
    // 验证状态改变
    const reserves = await dex.getReserves();
    expect(reserves[0]).to.be.above(0);
    
    // 恢复
    await ethers.provider.send("evm_revert", [snap]);
    
    // 验证状态恢复
    const reservesAfter = await dex.getReserves();
    expect(reservesAfter[0]).to.equal(0);
  });
});
```

---

## Part 3: Gas优化测试 (1.5小时)

### 3.1 Gas消耗测试

```javascript
describe("Gas Optimization Testing", function () {
  it("Should measure gas usage", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    const tx = await token.transfer(addr1.address, 100);
    const receipt = await tx.wait();
    
    console.log("Gas used:", receipt.gasUsed.toString());
    console.log("Gas price:", receipt.gasPrice.toString());
    console.log("Total cost:", (receipt.gasUsed * receipt.gasPrice).toString());
  });
  
  it("Should compare gas costs", async function () {
    const { tokenV1, tokenV2, owner, addr1 } = await loadFixture(deployBothVersionsFixture);
    
    // V1版本
    const tx1 = await tokenV1.transfer(addr1.address, 100);
    const receipt1 = await tx1.wait();
    const gasV1 = receipt1.gasUsed;
    
    // V2版本（优化后）
    const tx2 = await tokenV2.transfer(addr1.address, 100);
    const receipt2 = await tx2.wait();
    const gasV2 = receipt2.gasUsed;
    
    console.log("V1 gas:", gasV1.toString());
    console.log("V2 gas:", gasV2.toString());
    console.log("Saved:", (gasV1 - gasV2).toString());
    
    // 验证V2更节省
    expect(gasV2).to.be.below(gasV1);
  });
  
  it("Should test batch operations gas", async function () {
    const { token, owner, recipients } = await loadFixture(deployTokenFixture);
    
    // 单独转账
    let totalGasSingle = 0n;
    for (let i = 0; i < recipients.length; i++) {
      const tx = await token.transfer(recipients[i].address, 100);
      const receipt = await tx.wait();
      totalGasSingle += receipt.gasUsed;
    }
    
    // 批量转账
    await ethers.provider.send("evm_revert", [await ethers.provider.send("evm_snapshot", [])]);
    
    const tx = await token.batchTransfer(
      recipients.map(r => r.address),
      recipients.map(() => 100)
    );
    const receipt = await tx.wait();
    const totalGasBatch = receipt.gasUsed;
    
    console.log("Single transfers gas:", totalGasSingle.toString());
    console.log("Batch transfer gas:", totalGasBatch.toString());
    console.log("Saved:", (totalGasSingle - totalGasBatch).toString());
    
    expect(totalGasBatch).to.be.below(totalGasSingle);
  });
});
```

### 3.2 Storage优化测试

```javascript
describe("Storage Optimization", function () {
  it("Should test storage packing", async function () {
    // 未优化版本
    const UnoptimizedStorage = await ethers.getContractFactory("UnoptimizedStorage");
    const unoptimized = await UnoptimizedStorage.deploy();
    await unoptimized.waitForDeployment();
    
    const tx1 = await unoptimized.setValues(123, true, 456);
    const receipt1 = await tx1.wait();
    
    // 优化版本（变量打包）
    const OptimizedStorage = await ethers.getContractFactory("OptimizedStorage");
    const optimized = await OptimizedStorage.deploy();
    await optimized.waitForDeployment();
    
    const tx2 = await optimized.setValues(123, true, 456);
    const receipt2 = await tx2.wait();
    
    console.log("Unoptimized gas:", receipt1.gasUsed.toString());
    console.log("Optimized gas:", receipt2.gasUsed.toString());
    
    expect(receipt2.gasUsed).to.be.below(receipt1.gasUsed);
  });
  
  it("Should test memory vs storage", async function () {
    const { arrays } = await loadFixture(deployArraysFixture);
    
    // Storage操作
    const tx1 = await arrays.processArrayStorage([1, 2, 3, 4, 5]);
    const receipt1 = await tx1.wait();
    
    // Memory操作
    const tx2 = await arrays.processArrayMemory([1, 2, 3, 4, 5]);
    const receipt2 = await tx2.wait();
    
    console.log("Storage gas:", receipt1.gasUsed.toString());
    console.log("Memory gas:", receipt2.gasUsed.toString());
    
    expect(receipt2.gasUsed).to.be.below(receipt1.gasUsed);
  });
});
```

---

## Part 4: 覆盖率测试 (1小时)

### 4.1 配置覆盖率测试

```bash
# 安装solidity-coverage
npm install --save-dev solidity-coverage
```

```javascript
// hardhat.config.js
require("solidity-coverage");

module.exports = {
  solidity: "0.8.20",
  networks: {
    hardhat: {
      // coverage需要的配置
    }
  }
};
```

### 4.2 运行覆盖率测试

```bash
# 运行覆盖率测试
npx hardhat coverage

# 指定测试文件
npx hardhat coverage --testfiles "test/Token.test.js"

# 生成HTML报告
npx hardhat coverage --show-stack-traces
```

### 4.3 分析覆盖率报告

```javascript
describe("Coverage Testing", function () {
  it("Should cover all branches", async function () {
    const { conditionalContract } = await loadFixture(deployFixture);
    
    // 测试所有分支
    await conditionalContract.execute(true);  // if分支
    await conditionalContract.execute(false); // else分支
    
    // 测试边界条件
    await conditionalContract.setValue(0);     // 最小值
    await conditionalContract.setValue(100);   // 最大值
    await expect(
      conditionalContract.setValue(101)
    ).to.be.reverted;                          // 超出范围
  });
  
  it("Should cover error paths", async function () {
    const { token, addr1 } = await loadFixture(deployTokenFixture);
    
    // 正常路径
    await token.transfer(addr1.address, 100);
    
    // 错误路径1: 余额不足
    await expect(
      token.connect(addr1).transfer(owner.address, 1000)
    ).to.be.reverted;
    
    // 错误路径2: 无效地址
    await expect(
      token.transfer(ethers.ZeroAddress, 100)
    ).to.be.reverted;
    
    // 错误路径3: 零金额
    await expect(
      token.transfer(addr1.address, 0)
    ).to.be.reverted;
  });
});
```

---

## Part 5: 模拟与Stub (1小时)

### 5.1 合约Mock

```javascript
describe("Contract Mocking", function () {
  it("Should mock external contract", async function () {
    // 部署Mock合约
    const MockOracle = await ethers.getContractFactory("MockOracle");
    const mockOracle = await MockOracle.deploy();
    await mockOracle.waitForDeployment();
    
    // 设置Mock返回值
    await mockOracle.setPrice(ethers.parseEther("2000"));
    
    // 使用Mock
    const { priceConsumer } = await loadFixture(deployPriceConsumerFixture);
    await priceConsumer.setOracle(await mockOracle.getAddress());
    
    const price = await priceConsumer.getPrice();
    expect(price).to.equal(ethers.parseEther("2000"));
  });
  
  it("Should stub contract behavior", async function () {
    const { contract, mockDependency } = await loadFixture(deployWithMockFixture);
    
    // 配置不同的返回值
    await mockDependency.setReturnValue(true);
    expect(await contract.doSomething()).to.be.true;
    
    await mockDependency.setReturnValue(false);
    expect(await contract.doSomething()).to.be.false;
  });
});
```

### 5.2 函数拦截

```solidity
// MockERC20.sol
contract MockERC20 is ERC20 {
    bool public shouldFail;
    
    function setShouldFail(bool _shouldFail) external {
        shouldFail = _shouldFail;
    }
    
    function transfer(address to, uint256 amount) public override returns (bool) {
        if (shouldFail) {
            revert("Mock transfer failed");
        }
        return super.transfer(to, amount);
    }
}
```

```javascript
describe("Function Interception", function () {
  it("Should test with failing mock", async function () {
    const MockToken = await ethers.getContractFactory("MockERC20");
    const mockToken = await MockToken.deploy("Mock", "MCK");
    await mockToken.waitForDeployment();
    
    // 正常情况
    await mockToken.transfer(addr1.address, 100);
    
    // 设置失败
    await mockToken.setShouldFail(true);
    await expect(
      mockToken.transfer(addr1.address, 100)
    ).to.be.revertedWith("Mock transfer failed");
  });
});
```

---

## 📝 今日作业

### 作业1: 时间敏感合约测试

创建一个时间锁合约并测试：
1. 不同时间点的行为
2. 边界时间点测试
3. 时间跳跃测试
4. 多阶段测试

### 作业2: Gas优化测试

对比两个版本的合约：
1. 测量Gas消耗
2. 识别优化点
3. 实现优化
4. 验证改进效果

### 作业3: 覆盖率提升

为现有合约提升覆盖率：
1. 运行覆盖率测试
2. 识别未覆盖代码
3. 编写测试用例
4. 达到95%+覆盖率

---

## ✅ 检查清单

- [ ] 掌握时间操纵API
- [ ] 理解快照机制
- [ ] 会测试Gas消耗
- [ ] 达到高覆盖率
- [ ] 完成所有作业

---

## 🎯 常见问题FAQ

### Q1: 如何快进时间？

```javascript
await time.increase(24 * 60 * 60); // 1天
await time.increaseTo(timestamp);  // 到特定时间
```

### Q2: 如何恢复快照？

```javascript
const id = await ethers.provider.send("evm_snapshot", []);
// ...操作...
await ethers.provider.send("evm_revert", [id]);
```

### Q3: 如何测量Gas？

```javascript
const tx = await contract.function();
const receipt = await tx.wait();
console.log(receipt.gasUsed);
```

### Q4: 覆盖率低怎么办？

1. 检查未覆盖分支
2. 添加边界测试
3. 测试错误路径
4. 测试特殊情况

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

明天学习Foundry入门 - 上：
- Foundry安装配置
- Forge测试框架
- Solidity测试编写
- Fuzz测试基础

**预习内容**: 安装Foundry工具链

**🎉 完成Day 2！明天见！**
