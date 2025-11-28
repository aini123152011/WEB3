# Week 1 - Day 5: Hardhat框架入门

**学习日期**: ___________
**预计用时**: 5-6小时  
**难度等级**: ⭐⭐⭐ (进阶)

## 📚 今日学习目标

- ✅ 理解Hardhat开发框架核心概念
- ✅ 安装并配置Hardhat项目
- ✅ 编写第一个智能合约
- ✅ 使用Hardhat编译和部署合约
- ✅ 使用Hardhat Console与合约交互
- ✅ 理解Hardhat Network本地区块链
- ✅ 配置多网络支持

---

## 🎯 Part 1: Hardhat概述 (30分钟)

### 1.1 什么是Hardhat？

**Hardhat简介：**

```
Hardhat = 专业的以太坊开发框架

核心功能：
✓ 编译智能合约
✓ 部署到多种网络
✓ 内置本地区块链
✓ 编写和运行测试
✓ 调试智能合约
✓ 插件生态系统

为什么选择Hardhat？
- 最先进的Solidity工具链
- 出色的错误提示
- 强大的测试环境
- TypeScript原生支持
- 活跃的社区
```

### 1.2 Hardhat vs 其他框架

**框架对比：**

```
Hardhat vs Truffle:
Hardhat优势：
- 更快的编译速度
- 更详细的错误信息
- 内置console.log调试
- TypeScript原生支持
- 主网分叉功能强大

Hardhat vs Foundry:
Hardhat优势：
- JavaScript/TypeScript生态
- 更易上手
- 丰富的插件
Foundry优势：
- 更快的执行速度
- Solidity编写测试
- 更好的Fuzz测试
```

### 1.3 Hardhat架构

**核心组件：**

```
Hardhat架构：

hardhat.config.js       # 配置文件
  ├── Solidity编译器
  ├── 网络配置
  └── 插件配置

Hardhat Runtime Environment (HRE)
  ├── ethers            # 与区块链交互
  ├── network           # 网络信息
  ├── artifacts         # 编译产物
  └── config            # 配置访问

Hardhat Network         # 内置本地链
  ├── JSON-RPC服务
  ├── 自动挖矿
  └── 主网分叉

Tasks系统
  ├── 内置tasks
  └── 自定义tasks
```

---

## 🔧 Part 2: 安装与初始化 (45分钟)

### 2.1 创建Hardhat项目

**步骤1: 创建项目目录**

```powershell
# 创建项目目录
cd E:\Seadragon\WEB3
mkdir my-first-hardhat
cd my-first-hardhat

# 初始化npm项目
npm init -y
```

**步骤2: 安装Hardhat**

```powershell
# 安装Hardhat
npm install --save-dev hardhat

# 验证安装
npx hardhat --version
# 输出: 2.19.0 (或更高版本)
```

**步骤3: 初始化Hardhat项目**

```powershell
# 运行Hardhat初始化
npx hardhat

# 选择项目类型（选择第一个）：
? What do you want to do? …
❯ Create a JavaScript project
  Create a TypeScript project
  Create an empty hardhat.config.js
  Quit

# 后续提示：
? Hardhat project root: › (当前目录)  # 回车
? Do you want to add a .gitignore? › (Y/n)  # Y
? Do you want to install this sample project's dependencies with npm? › (Y/n)  # Y
```

**生成的项目结构：**

```
my-first-hardhat/
├── contracts/              # 智能合约目录
│   └── Lock.sol           # 示例合约
├── scripts/               # 部署脚本
│   └── deploy.js
├── test/                  # 测试文件
│   └── Lock.js
├── hardhat.config.js      # Hardhat配置
├── package.json
├── .gitignore
└── README.md
```

### 2.2 理解项目文件

**hardhat.config.js解析：**

```javascript
require("@nomicfoundation/hardhat-toolbox");

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  solidity: "0.8.19",  // Solidity编译器版本
  
  // 网络配置
  networks: {
    hardhat: {
      // Hardhat Network配置
    },
    localhost: {
      url: "http://127.0.0.1:8545"
    }
  }
};
```

**package.json依赖：**

```json
{
  "devDependencies": {
    "@nomicfoundation/hardhat-toolbox": "^4.0.0",
    "hardhat": "^2.19.0",
    
    // Toolbox包含以下依赖：
    // - @nomicfoundation/hardhat-ethers
    // - @nomicfoundation/hardhat-chai-matchers
    // - ethers
    // - chai
    // - hardhat-gas-reporter
    // - solidity-coverage
  }
}
```

### 2.3 配置详解

**完整配置示例：**

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  // Solidity编译器配置
  solidity: {
    version: "0.8.20",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200
      }
    }
  },
  
  // 网络配置
  networks: {
    // Hardhat Network (默认)
    hardhat: {
      chainId: 31337,
      mining: {
        auto: true,
        interval: 0
      }
    },
    
    // 本地节点
    localhost: {
      url: "http://127.0.0.1:8545",
      chainId: 31337
    },
    
    // Sepolia测试网
    sepolia: {
      url: "https://eth-sepolia.g.alchemy.com/v2/YOUR-API-KEY",
      accounts: ["YOUR-PRIVATE-KEY"],
      chainId: 11155111
    }
  },
  
  // 路径配置
  paths: {
    sources: "./contracts",
    tests: "./test",
    cache: "./cache",
    artifacts: "./artifacts"
  },
  
  // Gas报告
  gasReporter: {
    enabled: true,
    currency: "USD"
  },
  
  // Etherscan验证
  etherscan: {
    apiKey: "YOUR-ETHERSCAN-API-KEY"
  }
};
```

---

## 📝 Part 3: 第一个智能合约 (1小时)

### 3.1 创建简单合约

**创建Counter.sol：**

```solidity
// contracts/Counter.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title Counter
 * @dev 简单的计数器合约
 */
contract Counter {
    // 状态变量
    uint256 private count;
    address public owner;
    
    // 事件
    event CountChanged(uint256 newCount);
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    
    // 修饰器
    modifier onlyOwner() {
        require(msg.sender == owner, "Not the owner");
        _;
    }
    
    /**
     * @dev 构造函数
     * @param _initialCount 初始计数值
     */
    constructor(uint256 _initialCount) {
        count = _initialCount;
        owner = msg.sender;
    }
    
    /**
     * @dev 增加计数
     */
    function increment() public {
        count += 1;
        emit CountChanged(count);
    }
    
    /**
     * @dev 减少计数
     */
    function decrement() public {
        require(count > 0, "Count is already 0");
        count -= 1;
        emit CountChanged(count);
    }
    
    /**
     * @dev 获取当前计数
     * @return 当前计数值
     */
    function getCount() public view returns (uint256) {
        return count;
    }
    
    /**
     * @dev 重置计数
     */
    function reset() public onlyOwner {
        count = 0;
        emit CountChanged(count);
    }
    
    /**
     * @dev 转移所有权
     * @param newOwner 新所有者地址
     */
    function transferOwnership(address newOwner) public onlyOwner {
        require(newOwner != address(0), "Invalid address");
        address previousOwner = owner;
        owner = newOwner;
        emit OwnershipTransferred(previousOwner, newOwner);
    }
}
```

### 3.2 编译合约

**编译命令：**

```powershell
# 编译所有合约
npx hardhat compile

# 输出：
Compiled 1 Solidity file successfully (evm target: paris).

# 查看编译产物
ls artifacts/contracts/Counter.sol/
# Counter.json  Counter.dbg.json
```

**理解编译产物：**

```javascript
// artifacts/contracts/Counter.sol/Counter.json
{
  "contractName": "Counter",
  "abi": [
    // ABI定义（应用二进制接口）
    {
      "inputs": [{"type": "uint256", "name": "_initialCount"}],
      "stateMutability": "nonpayable",
      "type": "constructor"
    },
    {
      "name": "increment",
      "type": "function",
      "inputs": [],
      "outputs": [],
      "stateMutability": "nonpayable"
    }
    // ...更多函数定义
  ],
  "bytecode": "0x608060405234801561001057600080fd5b5060405161...",  // 部署字节码
  "deployedBytecode": "0x608060405234801561001057600080fd5b50600436..."   // 运行时字节码
}
```

**重新编译：**

```powershell
# 清除缓存并重新编译
npx hardhat clean
npx hardhat compile

# 强制重新编译
npx hardhat compile --force
```

---

## 🚀 Part 4: 部署合约 (1小时)

### 4.1 编写部署脚本

**创建deploy.js：**

```javascript
// scripts/deploy-counter.js
const hre = require("hardhat");

async function main() {
  console.log("开始部署Counter合约...");
  
  // 获取合约工厂
  const Counter = await hre.ethers.getContractFactory("Counter");
  
  // 设置初始计数
  const initialCount = 10;
  
  // 部署合约
  console.log(`部署参数：initialCount = ${initialCount}`);
  const counter = await Counter.deploy(initialCount);
  
  // 等待部署完成
  await counter.waitForDeployment();
  
  // 获取合约地址
  const address = await counter.getAddress();
  
  console.log("✅ Counter合约已部署");
  console.log(`📍 地址: ${address}`);
  console.log(`👤 所有者: ${await counter.owner()}`);
  console.log(`🔢 初始计数: ${await counter.getCount()}`);
  
  // 测试功能
  console.log("\n测试合约功能...");
  
  // 增加计数
  console.log("调用increment()...");
  const tx1 = await counter.increment();
  await tx1.wait();
  console.log(`当前计数: ${await counter.getCount()}`);
  
  // 再次增加
  console.log("再次调用increment()...");
  const tx2 = await counter.increment();
  await tx2.wait();
  console.log(`当前计数: ${await counter.getCount()}`);
  
  console.log("\n🎉 部署和测试完成！");
}

// 错误处理
main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

### 4.2 部署到本地网络

**启动Hardhat Network：**

```powershell
# 终端1: 启动本地节点
npx hardhat node

# 输出：
Started HTTP and WebSocket JSON-RPC server at http://127.0.0.1:8545/

Accounts
========
Account #0: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 (10000 ETH)
Private Key: 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

Account #1: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 (10000 ETH)
# ... 更多账户
```

**部署合约：**

```powershell
# 终端2: 部署到本地网络
npx hardhat run scripts/deploy-counter.js --network localhost

# 输出：
开始部署Counter合约...
部署参数：initialCount = 10
✅ Counter合约已部署
📍 地址: 0x5FbDB2315678afecb367f032d93F642f64180aa3
👤 所有者: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
🔢 初始计数: 10

测试合约功能...
调用increment()...
当前计数: 11
再次调用increment()...
当前计数: 12

🎉 部署和测试完成！
```

### 4.3 直接部署（无需启动节点）

```powershell
# 直接在Hardhat Network运行
npx hardhat run scripts/deploy-counter.js

# 每次运行都是全新的区块链环境
# 合约地址可能相同（因为是确定性的）
```

---

## 🎮 Part 5: Hardhat Console交互 (45分钟)

### 5.1 启动Console

**打开Hardhat Console：**

```powershell
# 启动console（连接到本地节点）
npx hardhat console --network localhost

# 进入交互式环境
Welcome to Node.js v20.10.0.
Type ".help" for more information.
>
```

### 5.2 基础操作

**获取账户：**

```javascript
// 获取所有账户
const accounts = await ethers.getSigners();
console.log("Accounts:", accounts.length);

// 获取第一个账户
const [owner] = accounts;
console.log("Owner address:", owner.address);

// 查看余额
const balance = await ethers.provider.getBalance(owner.address);
console.log("Balance:", ethers.formatEther(balance), "ETH");

// 获取账户2
const [, account2] = accounts;
console.log("Account 2:", account2.address);
```

### 5.3 与已部署合约交互

**连接到合约：**

```javascript
// 获取合约工厂
const Counter = await ethers.getContractFactory("Counter");

// 连接到已部署的合约
const contractAddress = "0x5FbDB2315678afecb367f032d93F642f64180aa3";
const counter = Counter.attach(contractAddress);

// 或使用Contract接口
const counterABI = [
  "function getCount() view returns (uint256)",
  "function increment()",
  "function decrement()",
  "function reset()",
  "function owner() view returns (address)"
];
const counter = new ethers.Contract(contractAddress, counterABI, owner);
```

**调用只读函数：**

```javascript
// 查看计数（不消耗gas）
const count = await counter.getCount();
console.log("Current count:", count.toString());

// 查看所有者
const ownerAddress = await counter.owner();
console.log("Owner:", ownerAddress);
```

**发送交易：**

```javascript
// 增加计数
const tx = await counter.increment();
console.log("Transaction hash:", tx.hash);

// 等待交易确认
const receipt = await tx.wait();
console.log("Confirmed in block:", receipt.blockNumber);

// 查看新的计数
const newCount = await counter.getCount();
console.log("New count:", newCount.toString());

// 连续操作
await (await counter.increment()).wait();
await (await counter.increment()).wait();
console.log("Count after 2 increments:", (await counter.getCount()).toString());
```

**监听事件：**

```javascript
// 监听CountChanged事件
counter.on("CountChanged", (newCount) => {
  console.log("Count changed to:", newCount.toString());
});

// 触发事件
await (await counter.increment()).wait();
// 输出: Count changed to: 13

// 查询历史事件
const filter = counter.filters.CountChanged();
const events = await counter.queryFilter(filter);
events.forEach((event) => {
  console.log("Event:", event.args.newCount.toString());
});

// 停止监听
counter.removeAllListeners("CountChanged");
```

### 5.4 高级操作

**切换账户：**

```javascript
// 使用不同账户调用
const [, account2] = await ethers.getSigners();
const counterAsAccount2 = counter.connect(account2);

// account2调用increment
await (await counterAsAccount2.increment()).wait();

// 尝试reset（会失败，因为不是owner）
try {
  await counterAsAccount2.reset();
} catch (error) {
  console.log("Error:", error.message);
  // 输出: Not the owner
}
```

**估算Gas：**

```javascript
// 估算increment的gas
const gasEstimate = await counter.increment.estimateGas();
console.log("Estimated gas:", gasEstimate.toString());

// 带参数的估算
const gasPrice = await ethers.provider.getFeeData();
console.log("Gas price:", ethers.formatUnits(gasPrice.gasPrice, "gwei"), "Gwei");
```

**查看交易详情：**

```javascript
// 获取交易receipt
const txHash = "0x...";
const receipt = await ethers.provider.getTransactionReceipt(txHash);
console.log("Gas used:", receipt.gasUsed.toString());
console.log("Status:", receipt.status); // 1=成功, 0=失败

// 获取交易详情
const tx = await ethers.provider.getTransaction(txHash);
console.log("From:", tx.from);
console.log("To:", tx.to);
console.log("Value:", ethers.formatEther(tx.value));
```

---

## 🧪 Part 6: Hardhat Network详解 (30分钟)

### 6.1 Hardhat Network特性

**自动挖矿模式：**

```javascript
// hardhat.config.js
module.exports = {
  networks: {
    hardhat: {
      mining: {
        auto: true,      // 自动挖矿
        interval: 0      // 立即挖矿
      }
    }
  }
};
```

**手动挖矿模式：**

```javascript
// hardhat.config.js
module.exports = {
  networks: {
    hardhat: {
      mining: {
        auto: false,     // 手动挖矿
        interval: 5000   // 或每5秒挖一个块
      }
    }
  }
};

// 在console中手动挖矿
await network.provider.send("evm_mine");

// 挖多个块
await network.provider.send("hardhat_mine", ["0x64"]); // 挖100个块
```

### 6.2 时间操作

**增加时间：**

```javascript
// 增加1小时
await ethers.provider.send("evm_increaseTime", [3600]);
await ethers.provider.send("evm_mine"); // 挖矿生效

// 设置下一个块的时间戳
const timestamp = Math.floor(Date.now() / 1000) + 86400; // 明天
await ethers.provider.send("evm_setNextBlockTimestamp", [timestamp]);
await ethers.provider.send("evm_mine");

// 获取当前块时间
const block = await ethers.provider.getBlock("latest");
console.log("Block timestamp:", block.timestamp);
```

### 6.3 账户操作

**模拟账户：**

```javascript
// 模拟任意地址发送交易
await hre.network.provider.request({
  method: "hardhat_impersonateAccount",
  params: ["0x742d35Cc6634C0532925a3b844Bc38F2285cabb1"]
});

const impersonated = await ethers.getSigner("0x742d35Cc6634C0532925a3b844Bc38F2285cabb1");

// 使用模拟账户
await counter.connect(impersonated).increment();

// 停止模拟
await hre.network.provider.request({
  method: "hardhat_stopImpersonatingAccount",
  params: ["0x742d35Cc6634C0532925a3b844Bc38F2285cabb1"]
});
```

**设置账户余额：**

```javascript
// 设置账户余额
await hre.network.provider.send("hardhat_setBalance", [
  "0x742d35Cc6634C0532925a3b844Bc38F2285cabb1",
  "0x56BC75E2D63100000" // 100 ETH in hex
]);

// 验证
const balance = await ethers.provider.getBalance("0x742d35Cc6634C0532925a3b844Bc38F2285cabb1");
console.log("Balance:", ethers.formatEther(balance));
```

### 6.4 快照与恢复

**使用快照：**

```javascript
// 创建快照
const snapshotId = await network.provider.send("evm_snapshot");
console.log("Snapshot ID:", snapshotId);

// 执行一些操作
await counter.increment();
console.log("Count:", await counter.getCount());

// 恢复到快照
await network.provider.send("evm_revert", [snapshotId]);
console.log("Count after revert:", await counter.getCount());
```

---

## 🛠️ Part 7: Hardhat Tasks (30分钟)

### 7.1 使用内置Tasks

**查看所有tasks：**

```powershell
npx hardhat
```

**常用内置tasks：**

```powershell
# 编译
npx hardhat compile

# 清理
npx hardhat clean

# 运行测试
npx hardhat test

# 查看账户
npx hardhat accounts

# 查看账户余额
npx hardhat balances

# 检查合约大小
npx hardhat size-contracts
```

### 7.2 创建自定义Task

**简单task：**

```javascript
// hardhat.config.js
const { task } = require("hardhat/config");

task("accounts", "Prints the list of accounts", async (taskArgs, hre) => {
  const accounts = await hre.ethers.getSigners();
  
  for (const account of accounts) {
    const balance = await hre.ethers.provider.getBalance(account.address);
    console.log(`${account.address} - ${hre.ethers.formatEther(balance)} ETH`);
  }
});

// 运行
// npx hardhat accounts
```

**带参数的task：**

```javascript
// hardhat.config.js
task("balance", "Prints an account's balance")
  .addParam("account", "The account's address")
  .setAction(async (taskArgs, hre) => {
    const balance = await hre.ethers.provider.getBalance(taskArgs.account);
    console.log(hre.ethers.formatEther(balance), "ETH");
  });

// 运行
// npx hardhat balance --account 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```

**复杂task - 部署和验证：**

```javascript
// hardhat.config.js
task("deploy-counter", "Deploy Counter contract")
  .addParam("initial", "Initial count value", "0")
  .setAction(async (taskArgs, hre) => {
    console.log("Deploying Counter...");
    
    const Counter = await hre.ethers.getContractFactory("Counter");
    const counter = await Counter.deploy(taskArgs.initial);
    await counter.waitForDeployment();
    
    const address = await counter.getAddress();
    console.log("Counter deployed to:", address);
    
    // 保存地址到文件
    const fs = require("fs");
    const data = {
      address: address,
      initialCount: taskArgs.initial,
      deployedAt: new Date().toISOString()
    };
    fs.writeFileSync("deployment.json", JSON.stringify(data, null, 2));
    
    return address;
  });

// 运行
// npx hardhat deploy-counter --initial 100
```

---

## 📝 今日作业

### 作业1: 创建Token合约

```solidity
// contracts/MyToken.sol
// 创建一个简单的ERC20-like代币合约

/**
 * 要求：
 * 1. 代币名称: MyToken
 * 2. 代币符号: MTK
 * 3. 总供应量: 1,000,000
 * 4. 功能:
 *    - balanceOf(address): 查询余额
 *    - transfer(address to, uint amount): 转账
 *    - mint(address to, uint amount): 铸造（仅owner）
 *    - burn(uint amount): 销毁
 * 5. 事件:
 *    - Transfer(address from, address to, uint amount)
 *    - Mint(address to, uint amount)
 *    - Burn(address from, uint amount)
 */

// TODO: 实现合约
```

### 作业2: 部署脚本

```javascript
// scripts/deploy-token.js

/**
 * 创建部署脚本，要求：
 * 1. 部署MyToken合约
 * 2. 铸造初始代币给deployer
 * 3. 转移一些代币给其他账户
 * 4. 打印所有账户余额
 * 5. 保存部署信息到JSON文件
 */

// TODO: 实现部署脚本
```

### 作业3: Console交互练习

```markdown
# Console交互任务

## 任务列表
[ ] 1. 连接到已部署的Counter合约
[ ] 2. 查询当前计数值
[ ] 3. 调用increment() 5次
[ ] 4. 监听CountChanged事件
[ ] 5. 使用不同账户调用合约
[ ] 6. 尝试调用reset()（非owner账户）
[ ] 7. 查询所有历史事件
[ ] 8. 创建快照，执行操作，然后恢复

记录你的操作过程和结果：
```

### 作业4: 自定义Task

```javascript
// hardhat.config.js

/**
 * 创建以下自定义tasks：
 * 
 * 1. check-balance
 *    - 检查指定地址的ETH余额
 *    - 参数: --address
 * 
 * 2. transfer
 *    - 从账户0转账ETH到指定地址
 *    - 参数: --to, --amount
 * 
 * 3. contract-info
 *    - 显示指定合约的信息
 *    - 参数: --address
 *    - 显示: 代码大小、余额等
 */

// TODO: 实现这些tasks
```

---

## ✅ 今日检查清单

### 环境搭建
- [ ] 安装Hardhat
- [ ] 初始化Hardhat项目
- [ ] 理解项目结构

### 合约开发
- [ ] 编写Counter合约
- [ ] 成功编译合约
- [ ] 理解编译产物

### 部署
- [ ] 编写部署脚本
- [ ] 启动本地节点
- [ ] 成功部署合约

### 交互
- [ ] 使用Hardhat Console
- [ ] 调用合约函数
- [ ] 监听事件

### 高级功能
- [ ] 使用快照和恢复
- [ ] 操作时间和区块
- [ ] 创建自定义task

---

## 🆘 常见问题FAQ

### Q1: 合约编译失败？

```powershell
# 检查Solidity版本
npx hardhat compile

# 常见错误：
# 1. pragma版本不匹配
#    解决：修改pragma或config中的version

# 2. 语法错误
#    解决：仔细阅读错误信息

# 3. 依赖问题
#    解决：重新安装依赖
npm install
```

### Q2: 部署后找不到合约？

```javascript
// 确保等待部署完成
const counter = await Counter.deploy(10);
await counter.waitForDeployment();  // 重要！

// 正确获取地址
const address = await counter.getAddress();
console.log("Deployed to:", address);
```

### Q3: Console中连接合约失败？

```javascript
// 方法1: 使用ContractFactory
const Counter = await ethers.getContractFactory("Counter");
const counter = Counter.attach(address);

// 方法2: 使用Contract
const counter = new ethers.Contract(
  address,
  ["function getCount() view returns (uint256)"],
  await ethers.getSigner()
);

// 确保合约地址正确
console.log("Connecting to:", address);
```

### Q4: 交易一直pending？

```javascript
// 检查是否等待确认
const tx = await counter.increment();
await tx.wait();  // 等待交易被挖矿

// 检查是否连接到正确的网络
const network = await ethers.provider.getNetwork();
console.log("Connected to:", network.name);
```

---

## 📚 扩展阅读

### Hardhat官方文档
- [Getting Started](https://hardhat.org/getting-started/)
- [Hardhat Network](https://hardhat.org/hardhat-network/)
- [Hardhat Runtime Environment](https://hardhat.org/advanced/hardhat-runtime-environment.html)

### 最佳实践
- [Hardhat Project Structure](https://hardhat.org/tutorial/creating-a-new-hardhat-project)
- [Testing Contracts](https://hardhat.org/tutorial/testing-contracts)
- [Deploying Contracts](https://hardhat.org/tutorial/deploying-to-a-live-network)

---

## 📅 明日预告: 周末综合项目

明天和后天是周末综合项目时间！我们将：
- 完整实现一个DApp项目
- 包含智能合约、部署、测试
- 前端页面与合约交互
- Git版本管理
- 完整项目文档

**今晚准备**：
- 确保Hardhat环境配置正确
- 完成今天的所有作业
- 复习前4天的知识
- 准备迎接第一个完整项目！

---

## ✍️ 我的学习记录

**完成日期**: ___________
**实际耗时**: _____ 小时

### ✅ 完成情况
- [ ] Hardhat安装配置
- [ ] 编写并编译合约
- [ ] 部署合约到本地网络
- [ ] Console交互
- [ ] 自定义Task

### 💡 今日收获
1. 最有价值的知识:
2. 最复杂的部分:
3. 成功部署的合约地址:

### 📝 合约开发
- 编写合约数: _________
- 部署成功次数: _________
- 遇到的错误: _________

### 🤔 疑问与思考
- 问题1:
- 问题2:

**🎉 完成Day 5！准备周末综合项目！**
