# Week 5 - Day 2: Web3自动化测试基础 - 下

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 掌握网络Fork测试
- ✅ 学习时间操纵测试
- ✅ 理解Gas估算优化
- ✅ 掌握多签钱包测试
- ✅ 学习代理模式测试
- ✅ 理解安全测试方法

---

## Part 1: 网络Fork测试 (2小时)

### 1.1 Fork主网环境

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Mainnet Fork Tests", function () {
  let provider;
  let uniswapRouter;
  let usdc, dai;
  
  before(async function () {
    // Fork主网
    await network.provider.request({
      method: "hardhat_reset",
      params: [{
        forking: {
          jsonRpcUrl: process.env.MAINNET_RPC_URL,
          blockNumber: 18000000 // 固定区块
        }
      }]
    });
    
    // 连接已部署的合约
    uniswapRouter = await ethers.getContractAt(
      "IUniswapV2Router02",
      "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D"
    );
    
    usdc = await ethers.getContractAt(
      "IERC20",
      "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
    );
    
    dai = await ethers.getContractAt(
      "IERC20",
      "0x6B175474E89094C44Da98b954EedeAC495271d0F"
    );
  });
  
  it("Should swap USDC for DAI on Uniswap", async function () {
    const [signer] = await ethers.getSigners();
    
    // 模拟拥有USDC
    const usdcWhale = "0x55FE002aefF02F77364de339a1292923A15844B8";
    await network.provider.request({
      method: "hardhat_impersonateAccount",
      params: [usdcWhale]
    });
    
    const whaleSigner = await ethers.getSigner(usdcWhale);
    
    // 转一些USDC给测试账户
    await usdc.connect(whaleSigner).transfer(
      signer.address,
      ethers.parseUnits("1000", 6) // USDC has 6 decimals
    );
    
    // 授权Router
    await usdc.connect(signer).approve(
      await uniswapRouter.getAddress(),
      ethers.parseUnits("1000", 6)
    );
    
    // 执行交换
    const path = [await usdc.getAddress(), await dai.getAddress()];
    const deadline = Math.floor(Date.now() / 1000) + 60 * 20;
    
    const daiBalanceBefore = await dai.balanceOf(signer.address);
    
    await uniswapRouter.connect(signer).swapExactTokensForTokens(
      ethers.parseUnits("1000", 6),
      0,
      path,
      signer.address,
      deadline
    );
    
    const daiBalanceAfter = await dai.balanceOf(signer.address);
    expect(daiBalanceAfter).to.be.gt(daiBalanceBefore);
    
    // 停止模拟
    await network.provider.request({
      method: "hardhat_stopImpersonatingAccount",
      params: [usdcWhale]
    });
  });
  
  it("Should interact with Aave protocol", async function () {
    const aavePool = await ethers.getContractAt(
      "IPool",
      "0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2"
    );
    
    const [signer] = await ethers.getSigners();
    
    // 存入USDC到Aave
    const depositAmount = ethers.parseUnits("100", 6);
    
    // 模拟拥有USDC
    const usdcWhale = "0x55FE002aefF02F77364de339a1292923A15844B8";
    await network.provider.request({
      method: "hardhat_impersonateAccount",
      params: [usdcWhale]
    });
    
    const whaleSigner = await ethers.getSigner(usdcWhale);
    await usdc.connect(whaleSigner).transfer(signer.address, depositAmount);
    
    // 授权Aave
    await usdc.connect(signer).approve(
      await aavePool.getAddress(),
      depositAmount
    );
    
    // 存入
    await aavePool.connect(signer).supply(
      await usdc.getAddress(),
      depositAmount,
      signer.address,
      0
    );
    
    // 验证存入成功
    const aToken = await aavePool.getReserveData(await usdc.getAddress());
    const aUSDC = await ethers.getContractAt("IERC20", aToken.aTokenAddress);
    
    expect(await aUSDC.balanceOf(signer.address)).to.be.gte(depositAmount);
  });
});
```

### 1.2 多链Fork测试

```javascript
describe("Multi-Chain Fork Tests", function () {
  it("Should compare prices across chains", async function () {
    // Fork Ethereum主网
    await network.provider.request({
      method: "hardhat_reset",
      params: [{
        forking: {
          jsonRpcUrl: process.env.MAINNET_RPC_URL,
          blockNumber: 18000000
        }
      }]
    });
    
    const uniswapV2Router = await ethers.getContractAt(
      "IUniswapV2Router02",
      "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D"
    );
    
    const weth = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2";
    const usdc = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48";
    
    const amountsOut = await uniswapV2Router.getAmountsOut(
      ethers.parseEther("1"),
      [weth, usdc]
    );
    
    const ethPriceOnMainnet = amountsOut[1];
    console.log("ETH price on Mainnet:", ethers.formatUnits(ethPriceOnMainnet, 6));
    
    // 可以继续Fork其他链进行对比
  });
});
```

---

## Part 2: 时间操纵测试 (1.5小时)

### 2.1 时间跳跃

```javascript
describe("Time Manipulation", function () {
  let timelock;
  
  beforeEach(async function () {
    const Timelock = await ethers.getContractFactory("Timelock");
    timelock = await Timelock.deploy(7 * 24 * 60 * 60); // 7天锁定期
    await timelock.waitForDeployment();
  });
  
  it("Should lock funds for specified period", async function () {
    const [owner] = await ethers.getSigners();
    const amount = ethers.parseEther("1.0");
    
    // 存入资金
    await timelock.deposit({ value: amount });
    
    // 立即尝试提取（应该失败）
    await expect(
      timelock.withdraw()
    ).to.be.revertedWith("Funds are locked");
    
    // 快进6天（还未到期）
    await ethers.provider.send("evm_increaseTime", [6 * 24 * 60 * 60]);
    await ethers.provider.send("evm_mine", []);
    
    await expect(
      timelock.withdraw()
    ).to.be.revertedWith("Funds are locked");
    
    // 再快进2天（超过7天）
    await ethers.provider.send("evm_increaseTime", [2 * 24 * 60 * 60]);
    await ethers.provider.send("evm_mine", []);
    
    // 现在应该可以提取
    await expect(timelock.withdraw()).to.not.be.reverted;
  });
  
  it("Should handle vesting schedule", async function () {
    const Vesting = await ethers.getContractFactory("TokenVesting");
    const token = await (await ethers.getContractFactory("Token"))
      .deploy("Test", "TST", ethers.parseEther("1000000"));
    await token.waitForDeployment();
    
    const [owner, beneficiary] = await ethers.getSigners();
    
    const startTime = (await ethers.provider.getBlock("latest")).timestamp;
    const duration = 365 * 24 * 60 * 60; // 1年
    const totalAmount = ethers.parseEther("10000");
    
    const vesting = await Vesting.deploy(
      await token.getAddress(),
      beneficiary.address,
      startTime,
      duration
    );
    await vesting.waitForDeployment();
    
    // 转入代币
    await token.transfer(await vesting.getAddress(), totalAmount);
    
    // 快进到25%时间点
    await ethers.provider.send("evm_increaseTime", [duration / 4]);
    await ethers.provider.send("evm_mine", []);
    
    await vesting.release();
    let released = await vesting.released();
    expect(released).to.be.closeTo(
      totalAmount / 4n,
      ethers.parseEther("100")
    );
    
    // 快进到50%时间点
    await ethers.provider.send("evm_increaseTime", [duration / 4]);
    await ethers.provider.send("evm_mine", []);
    
    await vesting.release();
    released = await vesting.released();
    expect(released).to.be.closeTo(
      totalAmount / 2n,
      ethers.parseEther("100")
    );
    
    // 快进到100%时间点
    await ethers.provider.send("evm_increaseTime", [duration / 2]);
    await ethers.provider.send("evm_mine", []);
    
    await vesting.release();
    released = await vesting.released();
    expect(released).to.equal(totalAmount);
  });
});
```

### 2.2 区块时间戳测试

```javascript
describe("Block Timestamp Tests", function () {
  it("Should test auction timing", async function () {
    const Auction = await ethers.getContractFactory("Auction");
    const duration = 7 * 24 * 60 * 60; // 7天
    
    const auction = await Auction.deploy(duration);
    await auction.waitForDeployment();
    
    const [owner, bidder1, bidder2] = await ethers.getSigners();
    
    // 出价
    await auction.connect(bidder1).bid({ value: ethers.parseEther("1.0") });
    await auction.connect(bidder2).bid({ value: ethers.parseEther("2.0") });
    
    // 拍卖期间不能结束
    await expect(
      auction.finalize()
    ).to.be.revertedWith("Auction not ended");
    
    // 快进到拍卖结束
    await ethers.provider.send("evm_increaseTime", [duration + 1]);
    await ethers.provider.send("evm_mine", []);
    
    // 现在可以结束
    await expect(auction.finalize()).to.not.be.reverted;
    
    // 拍卖结束后不能再出价
    await expect(
      auction.connect(bidder1).bid({ value: ethers.parseEther("3.0") })
    ).to.be.revertedWith("Auction ended");
  });
});
```

---

## Part 3: Gas优化测试 (1.5小时)

### 3.1 Gas消耗对比

```javascript
describe("Gas Optimization Tests", function () {
  let token;
  
  beforeEach(async function () {
    const Token = await ethers.getContractFactory("Token");
    token = await Token.deploy("Test", "TST", ethers.parseEther("1000000"));
    await token.waitForDeployment();
  });
  
  it("Should measure gas for single transfer", async function () {
    const [owner, addr1] = await ethers.getSigners();
    
    const tx = await token.transfer(addr1.address, 100);
    const receipt = await tx.wait();
    
    console.log("Single transfer gas:", receipt.gasUsed.toString());
    expect(receipt.gasUsed).to.be.lt(100000);
  });
  
  it("Should compare batch vs individual transfers", async function () {
    const [owner] = await ethers.getSigners();
    const recipients = [];
    const amounts = [];
    
    for (let i = 0; i < 10; i++) {
      recipients.push(ethers.Wallet.createRandom().address);
      amounts.push(100);
    }
    
    // 测量单独转账
    let totalGasIndividual = 0n;
    for (let i = 0; i < recipients.length; i++) {
      const tx = await token.transfer(recipients[i], amounts[i]);
      const receipt = await tx.wait();
      totalGasIndividual += receipt.gasUsed;
    }
    
    console.log("Individual transfers gas:", totalGasIndividual.toString());
    
    // 部署新合约测试批量转账
    const Token2 = await ethers.getContractFactory("Token");
    const token2 = await Token2.deploy("Test2", "TST2", ethers.parseEther("1000000"));
    await token2.waitForDeployment();
    
    const tx = await token2.batchTransfer(recipients, amounts);
    const receipt = await tx.wait();
    const gasBatch = receipt.gasUsed;
    
    console.log("Batch transfer gas:", gasBatch.toString());
    console.log("Gas saved:", (totalGasIndividual - gasBatch).toString());
    
    expect(gasBatch).to.be.lt(totalGasIndividual);
  });
  
  it("Should test storage optimization", async function () {
    const StorageTest = await ethers.getContractFactory("StorageOptimization");
    const storage = await StorageTest.deploy();
    await storage.waitForDeployment();
    
    // 测试未优化版本
    const tx1 = await storage.setValuesUnoptimized(123, true, 456);
    const receipt1 = await tx1.wait();
    
    // 测试优化版本（变量打包）
    const tx2 = await storage.setValuesOptimized(123, true, 456);
    const receipt2 = await tx2.wait();
    
    console.log("Unoptimized gas:", receipt1.gasUsed.toString());
    console.log("Optimized gas:", receipt2.gasUsed.toString());
    console.log("Saved:", (receipt1.gasUsed - receipt2.gasUsed).toString());
    
    expect(receipt2.gasUsed).to.be.lt(receipt1.gasUsed);
  });
});
```

### 3.2 Gas估算

```javascript
describe("Gas Estimation", function () {
  it("Should estimate gas before transaction", async function () {
    const Token = await ethers.getContractFactory("Token");
    const token = await Token.deploy("Test", "TST", ethers.parseEther("1000"));
    await token.waitForDeployment();
    
    const [owner, addr1] = await ethers.getSigners();
    
    // 估算Gas
    const estimatedGas = await token.transfer.estimateGas(addr1.address, 100);
    console.log("Estimated gas:", estimatedGas.toString());
    
    // 执行交易
    const tx = await token.transfer(addr1.address, 100);
    const receipt = await tx.wait();
    
    console.log("Actual gas:", receipt.gasUsed.toString());
    
    // 实际使用应该接近估算
    expect(receipt.gasUsed).to.be.closeTo(estimatedGas, estimatedGas / 10n);
  });
});
```

---

## Part 4: 多签钱包测试 (1.5小时)

### 4.1 多签基础测试

```javascript
describe("MultiSig Wallet Tests", function () {
  let multisig;
  let owners;
  
  beforeEach(async function () {
    owners = await ethers.getSigners();
    const ownerAddresses = owners.slice(0, 3).map(o => o.address);
    
    const MultiSig = await ethers.getContractFactory("MultiSigWallet");
    multisig = await MultiSig.deploy(ownerAddresses, 2); // 2/3多签
    await multisig.waitForDeployment();
    
    // 存入一些ETH
    await owners[0].sendTransaction({
      to: await multisig.getAddress(),
      value: ethers.parseEther("10.0")
    });
  });
  
  it("Should require multiple signatures", async function () {
    const [owner1, owner2, owner3, recipient] = owners;
    const amount = ethers.parseEther("1.0");
    
    // 提交交易
    const tx = await multisig.connect(owner1).submitTransaction(
      recipient.address,
      amount,
      "0x"
    );
    const receipt = await tx.wait();
    
    // 获取交易ID
    const event = receipt.logs.find(
      log => log.fragment && log.fragment.name === "SubmitTransaction"
    );
    const txId = event.args.txId;
    
    // 第一个签名（提交者自动签名）
    expect(await multisig.isConfirmed(txId, owner1.address)).to.be.true;
    
    // 尝试执行（应该失败，需要2个签名）
    await expect(
      multisig.connect(owner1).executeTransaction(txId)
    ).to.be.revertedWith("Cannot execute tx");
    
    // 第二个签名
    await multisig.connect(owner2).confirmTransaction(txId);
    
    // 现在可以执行
    const balanceBefore = await ethers.provider.getBalance(recipient.address);
    await multisig.connect(owner1).executeTransaction(txId);
    const balanceAfter = await ethers.provider.getBalance(recipient.address);
    
    expect(balanceAfter - balanceBefore).to.equal(amount);
  });
  
  it("Should allow revoking confirmation", async function () {
    const [owner1, owner2, , recipient] = owners;
    const amount = ethers.parseEther("1.0");
    
    // 提交交易
    const tx = await multisig.connect(owner1).submitTransaction(
      recipient.address,
      amount,
      "0x"
    );
    const receipt = await tx.wait();
    const event = receipt.logs.find(
      log => log.fragment && log.fragment.name === "SubmitTransaction"
    );
    const txId = event.args.txId;
    
    // Owner2签名
    await multisig.connect(owner2).confirmTransaction(txId);
    
    // Owner2撤销签名
    await multisig.connect(owner2).revokeConfirmation(txId);
    
    // 现在不能执行
    await expect(
      multisig.connect(owner1).executeTransaction(txId)
    ).to.be.revertedWith("Cannot execute tx");
  });
  
  it("Should handle contract calls", async function () {
    const [owner1, owner2] = owners;
    
    // 部署目标合约
    const Token = await ethers.getContractFactory("Token");
    const token = await Token.deploy("Test", "TST", ethers.parseEther("1000"));
    await token.waitForDeployment();
    
    // 给多签钱包一些代币
    await token.transfer(await multisig.getAddress(), 1000);
    
    // 编码transfer调用
    const data = token.interface.encodeFunctionData("transfer", [
      owner1.address,
      500
    ]);
    
    // 提交交易
    const tx = await multisig.connect(owner1).submitTransaction(
      await token.getAddress(),
      0,
      data
    );
    const receipt = await tx.wait();
    const event = receipt.logs.find(
      log => log.fragment && log.fragment.name === "SubmitTransaction"
    );
    const txId = event.args.txId;
    
    // 签名并执行
    await multisig.connect(owner2).confirmTransaction(txId);
    await multisig.connect(owner1).executeTransaction(txId);
    
    // 验证转账成功
    expect(await token.balanceOf(owner1.address)).to.equal(500);
  });
});
```

---

## Part 5: 代理模式测试 (1小时)

### 5.1 可升级合约测试

```javascript
describe("Upgradeable Contract Tests", function () {
  it("Should upgrade implementation", async function () {
    const [owner] = await ethers.getSigners();
    
    // 部署V1
    const TokenV1 = await ethers.getContractFactory("TokenV1");
    const tokenV1 = await TokenV1.deploy();
    await tokenV1.waitForDeployment();
    
    // 部署代理
    const Proxy = await ethers.getContractFactory("TransparentProxy");
    const proxy = await Proxy.deploy(
      await tokenV1.getAddress(),
      owner.address,
      "0x"
    );
    await proxy.waitForDeployment();
    
    // 通过代理调用V1
    const tokenProxy = TokenV1.attach(await proxy.getAddress());
    await tokenProxy.initialize("TokenV1", "TV1", ethers.parseEther("1000"));
    
    expect(await tokenProxy.name()).to.equal("TokenV1");
    expect(await tokenProxy.version()).to.equal(1);
    
    // 部署V2
    const TokenV2 = await ethers.getContractFactory("TokenV2");
    const tokenV2 = await TokenV2.deploy();
    await tokenV2.waitForDeployment();
    
    // 升级到V2
    await proxy.upgradeTo(await tokenV2.getAddress());
    
    // 通过代理调用V2
    const tokenProxyV2 = TokenV2.attach(await proxy.getAddress());
    
    // 数据应该保留
    expect(await tokenProxyV2.name()).to.equal("TokenV1");
    expect(await tokenProxyV2.totalSupply()).to.equal(ethers.parseEther("1000"));
    
    // V2新功能可用
    expect(await tokenProxyV2.version()).to.equal(2);
    await tokenProxyV2.newFunction();
  });
});
```

---

## 📝 今日作业

### 作业1: Fork测试实战

Fork主网测试DeFi协议：
1. Uniswap交换
2. Aave存贷
3. Compound借贷
4. 价格对比

### 作业2: 时间敏感合约

测试时间锁合约：
1. 锁定期测试
2. 线性释放测试
3. 拍卖时间测试

### 作业3: Gas优化

优化合约Gas消耗：
1. 批量操作
2. 存储优化
3. 循环优化
4. 对比测试

---

## ✅ 检查清单

- [ ] 掌握Fork测试
- [ ] 理解时间操纵
- [ ] 会Gas优化
- [ ] 完成多签测试
- [ ] 完成所有作业

---

## 📅 明日预告

明天学习Ethers.js深入 - 上：
- Provider深入
- Signer管理
- Contract交互
- 工具函数

**🎉 完成Day 2！明天见！**
