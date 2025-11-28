# Week 1 - Day 6-7: 周末综合项目 - 投票DApp

**学习日期**: ___________  
**预计用时**: 10-12小时（分两天完成）  
**难度等级**: ⭐⭐⭐⭐ (综合实战)

## 📚 项目概述

### 项目目标
构建一个完整的去中心化投票应用（Voting DApp），整合本周所学的所有知识：
- ✅ 智能合约开发（Solidity）
- ✅ Hardhat开发框架
- ✅ 合约部署与交互
- ✅ 前端页面集成
- ✅ Git版本控制

### 项目功能
1. 创建投票提案
2. 用户投票
3. 查看投票结果
4. 提案截止时间控制
5. 所有权管理

### 技术栈
```
后端（区块链）:
- Solidity ^0.8.20
- Hardhat
- Ethers.js v6

前端:
- HTML5/CSS3
- JavaScript
- Ethers.js (前端)
```

---

## 🏗️ Day 6: 智能合约开发 (5-6小时)

### Part 1: 项目初始化 (30分钟)

**步骤1: 创建项目**

```powershell
# 创建项目目录
cd E:\Seadragon\WEB3
mkdir voting-dapp
cd voting-dapp

# 初始化npm
npm init -y

# 安装Hardhat
npm install --save-dev hardhat

# 初始化Hardhat项目
npx hardhat
# 选择: Create a JavaScript project
```

**步骤2: 项目结构**

```
voting-dapp/
├── contracts/
│   ├── Voting.sol           # 主合约
│   └── VotingFactory.sol    # 工厂合约（可选）
├── scripts/
│   ├── deploy.js            # 部署脚本
│   └── interact.js          # 交互脚本
├── test/
│   ├── Voting.test.js       # 测试文件
│   └── integration.test.js  # 集成测试
├── frontend/
│   ├── index.html           # 前端页面
│   ├── style.css            # 样式文件
│   └── app.js               # 前端交互
├── hardhat.config.js
├── package.json
├── .gitignore
└── README.md
```

**步骤3: 配置Hardhat**

```javascript
// hardhat.config.js
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
      chainId: 31337
    },
    localhost: {
      url: "http://127.0.0.1:8545"
    }
  },
  paths: {
    sources: "./contracts",
    tests: "./test",
    cache: "./cache",
    artifacts: "./artifacts"
  }
};
```

**步骤4: 初始化Git**

```powershell
# 初始化Git
git init

# 创建.gitignore
echo "node_modules/
cache/
artifacts/
.env
.DS_Store
coverage/
typechain-types/
" > .gitignore

# 第一次提交
git add .
git commit -m "Initial commit: Project setup"
```

---

### Part 2: 智能合约开发 (2.5小时)

**创建Voting.sol合约：**

```solidity
// contracts/Voting.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title Voting
 * @dev 投票合约，支持创建提案和投票
 */
contract Voting {
    // ============ 数据结构 ============
    
    /**
     * @dev 提案结构
     */
    struct Proposal {
        uint256 id;              // 提案ID
        string description;      // 提案描述
        uint256 voteCount;       // 得票数
        uint256 deadline;        // 截止时间
        address proposer;        // 提案人
        bool executed;           // 是否已执行
        mapping(address => bool) voters;  // 投票记录
    }
    
    // ============ 状态变量 ============
    
    address public owner;                    // 合约所有者
    uint256 public proposalCount;            // 提案总数
    mapping(uint256 => Proposal) public proposals;  // 提案映射
    
    // 投票权限控制
    mapping(address => bool) public hasVotingRight;
    bool public openToAll;  // 是否对所有人开放
    
    // ============ 事件 ============
    
    event ProposalCreated(
        uint256 indexed proposalId,
        string description,
        uint256 deadline,
        address indexed proposer
    );
    
    event Voted(
        uint256 indexed proposalId,
        address indexed voter
    );
    
    event ProposalExecuted(
        uint256 indexed proposalId
    );
    
    event VotingRightGranted(
        address indexed voter
    );
    
    event VotingRightRevoked(
        address indexed voter
    );
    
    // ============ 修饰器 ============
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not the owner");
        _;
    }
    
    modifier proposalExists(uint256 _proposalId) {
        require(_proposalId > 0 && _proposalId <= proposalCount, "Proposal does not exist");
        _;
    }
    
    modifier hasNotVoted(uint256 _proposalId) {
        require(!proposals[_proposalId].voters[msg.sender], "Already voted");
        _;
    }
    
    modifier beforeDeadline(uint256 _proposalId) {
        require(block.timestamp < proposals[_proposalId].deadline, "Voting has ended");
        _;
    }
    
    modifier canVote() {
        require(openToAll || hasVotingRight[msg.sender], "No voting right");
        _;
    }
    
    // ============ 构造函数 ============
    
    /**
     * @dev 构造函数
     * @param _openToAll 是否对所有人开放投票
     */
    constructor(bool _openToAll) {
        owner = msg.sender;
        openToAll = _openToAll;
        hasVotingRight[msg.sender] = true;
    }
    
    // ============ 管理函数 ============
    
    /**
     * @dev 授予投票权
     * @param _voter 投票者地址
     */
    function grantVotingRight(address _voter) external onlyOwner {
        require(_voter != address(0), "Invalid address");
        require(!hasVotingRight[_voter], "Already has voting right");
        
        hasVotingRight[_voter] = true;
        emit VotingRightGranted(_voter);
    }
    
    /**
     * @dev 撤销投票权
     * @param _voter 投票者地址
     */
    function revokeVotingRight(address _voter) external onlyOwner {
        require(hasVotingRight[_voter], "Does not have voting right");
        
        hasVotingRight[_voter] = false;
        emit VotingRightRevoked(_voter);
    }
    
    /**
     * @dev 设置是否对所有人开放
     * @param _openToAll 是否开放
     */
    function setOpenToAll(bool _openToAll) external onlyOwner {
        openToAll = _openToAll;
    }
    
    // ============ 提案函数 ============
    
    /**
     * @dev 创建提案
     * @param _description 提案描述
     * @param _durationInDays 投票持续天数
     */
    function createProposal(
        string memory _description,
        uint256 _durationInDays
    ) external canVote returns (uint256) {
        require(bytes(_description).length > 0, "Description cannot be empty");
        require(_durationInDays > 0, "Duration must be positive");
        
        proposalCount++;
        uint256 deadline = block.timestamp + (_durationInDays * 1 days);
        
        Proposal storage newProposal = proposals[proposalCount];
        newProposal.id = proposalCount;
        newProposal.description = _description;
        newProposal.deadline = deadline;
        newProposal.proposer = msg.sender;
        
        emit ProposalCreated(proposalCount, _description, deadline, msg.sender);
        
        return proposalCount;
    }
    
    /**
     * @dev 投票
     * @param _proposalId 提案ID
     */
    function vote(uint256 _proposalId)
        external
        proposalExists(_proposalId)
        hasNotVoted(_proposalId)
        beforeDeadline(_proposalId)
        canVote
    {
        Proposal storage proposal = proposals[_proposalId];
        
        proposal.voters[msg.sender] = true;
        proposal.voteCount++;
        
        emit Voted(_proposalId, msg.sender);
    }
    
    /**
     * @dev 执行提案（占位符，实际项目中可实现具体逻辑）
     * @param _proposalId 提案ID
     */
    function executeProposal(uint256 _proposalId)
        external
        onlyOwner
        proposalExists(_proposalId)
    {
        Proposal storage proposal = proposals[_proposalId];
        require(block.timestamp >= proposal.deadline, "Voting not ended");
        require(!proposal.executed, "Already executed");
        
        proposal.executed = true;
        emit ProposalExecuted(_proposalId);
    }
    
    // ============ 查询函数 ============
    
    /**
     * @dev 获取提案信息
     * @param _proposalId 提案ID
     */
    function getProposal(uint256 _proposalId)
        external
        view
        proposalExists(_proposalId)
        returns (
            uint256 id,
            string memory description,
            uint256 voteCount,
            uint256 deadline,
            address proposer,
            bool executed
        )
    {
        Proposal storage proposal = proposals[_proposalId];
        return (
            proposal.id,
            proposal.description,
            proposal.voteCount,
            proposal.deadline,
            proposal.proposer,
            proposal.executed
        );
    }
    
    /**
     * @dev 检查是否已投票
     * @param _proposalId 提案ID
     * @param _voter 投票者地址
     */
    function hasVoted(uint256 _proposalId, address _voter)
        external
        view
        proposalExists(_proposalId)
        returns (bool)
    {
        return proposals[_proposalId].voters[_voter];
    }
    
    /**
     * @dev 获取所有提案ID
     */
    function getAllProposalIds() external view returns (uint256[] memory) {
        uint256[] memory ids = new uint256[](proposalCount);
        for (uint256 i = 1; i <= proposalCount; i++) {
            ids[i - 1] = i;
        }
        return ids;
    }
    
    /**
     * @dev 检查提案是否活跃
     * @param _proposalId 提案ID
     */
    function isProposalActive(uint256 _proposalId)
        external
        view
        proposalExists(_proposalId)
        returns (bool)
    {
        return block.timestamp < proposals[_proposalId].deadline;
    }
}
```

**提交代码：**

```powershell
git add contracts/Voting.sol
git commit -m "feat: Add Voting smart contract"
```

---

### Part 3: 编写测试 (1.5小时)

**创建测试文件：**

```javascript
// test/Voting.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { time } = require("@nomicfoundation/hardhat-network-helpers");

describe("Voting Contract", function () {
  let voting;
  let owner;
  let voter1;
  let voter2;
  let voter3;
  
  beforeEach(async function () {
    // 获取签名者
    [owner, voter1, voter2, voter3] = await ethers.getSigners();
    
    // 部署合约（对所有人开放）
    const Voting = await ethers.getContractFactory("Voting");
    voting = await Voting.deploy(true);
    await voting.waitForDeployment();
  });
  
  describe("Deployment", function () {
    it("Should set the right owner", async function () {
      expect(await voting.owner()).to.equal(owner.address);
    });
    
    it("Should be open to all", async function () {
      expect(await voting.openToAll()).to.be.true;
    });
    
    it("Should grant voting right to owner", async function () {
      expect(await voting.hasVotingRight(owner.address)).to.be.true;
    });
  });
  
  describe("Proposal Creation", function () {
    it("Should create a proposal", async function () {
      const description = "Proposal 1";
      const duration = 7; // 7 days
      
      await expect(voting.createProposal(description, duration))
        .to.emit(voting, "ProposalCreated")
        .withArgs(1, description, await time.latest() + duration * 24 * 60 * 60, owner.address);
      
      expect(await voting.proposalCount()).to.equal(1);
    });
    
    it("Should fail with empty description", async function () {
      await expect(voting.createProposal("", 7))
        .to.be.revertedWith("Description cannot be empty");
    });
    
    it("Should fail with zero duration", async function () {
      await expect(voting.createProposal("Proposal", 0))
        .to.be.revertedWith("Duration must be positive");
    });
  });
  
  describe("Voting", function () {
    beforeEach(async function () {
      // 创建提案
      await voting.createProposal("Test Proposal", 7);
    });
    
    it("Should allow voting", async function () {
      await expect(voting.connect(voter1).vote(1))
        .to.emit(voting, "Voted")
        .withArgs(1, voter1.address);
      
      const proposal = await voting.getProposal(1);
      expect(proposal.voteCount).to.equal(1);
    });
    
    it("Should prevent double voting", async function () {
      await voting.connect(voter1).vote(1);
      
      await expect(voting.connect(voter1).vote(1))
        .to.be.revertedWith("Already voted");
    });
    
    it("Should track who voted", async function () {
      await voting.connect(voter1).vote(1);
      
      expect(await voting.hasVoted(1, voter1.address)).to.be.true;
      expect(await voting.hasVoted(1, voter2.address)).to.be.false;
    });
    
    it("Should prevent voting after deadline", async function () {
      // 增加时间到截止日期后
      await time.increase(8 * 24 * 60 * 60); // 8 days
      
      await expect(voting.connect(voter1).vote(1))
        .to.be.revertedWith("Voting has ended");
    });
    
    it("Should fail for non-existent proposal", async function () {
      await expect(voting.vote(999))
        .to.be.revertedWith("Proposal does not exist");
    });
  });
  
  describe("Voting Rights Management", function () {
    let restrictedVoting;
    
    beforeEach(async function () {
      // 部署受限制的投票合约
      const Voting = await ethers.getContractFactory("Voting");
      restrictedVoting = await Voting.deploy(false);
      await restrictedVoting.waitForDeployment();
    });
    
    it("Should grant voting right", async function () {
      await expect(restrictedVoting.grantVotingRight(voter1.address))
        .to.emit(restrictedVoting, "VotingRightGranted")
        .withArgs(voter1.address);
      
      expect(await restrictedVoting.hasVotingRight(voter1.address)).to.be.true;
    });
    
    it("Should revoke voting right", async function () {
      await restrictedVoting.grantVotingRight(voter1.address);
      
      await expect(restrictedVoting.revokeVotingRight(voter1.address))
        .to.emit(restrictedVoting, "VotingRightRevoked")
        .withArgs(voter1.address);
      
      expect(await restrictedVoting.hasVotingRight(voter1.address)).to.be.false;
    });
    
    it("Should prevent voting without right", async function () {
      await restrictedVoting.createProposal("Test", 7);
      
      await expect(restrictedVoting.connect(voter1).vote(1))
        .to.be.revertedWith("No voting right");
    });
    
    it("Only owner can grant rights", async function () {
      await expect(restrictedVoting.connect(voter1).grantVotingRight(voter2.address))
        .to.be.revertedWith("Not the owner");
    });
  });
  
  describe("Query Functions", function () {
    beforeEach(async function () {
      await voting.createProposal("Proposal 1", 7);
      await voting.createProposal("Proposal 2", 14);
      await voting.connect(voter1).vote(1);
    });
    
    it("Should get proposal details", async function () {
      const proposal = await voting.getProposal(1);
      
      expect(proposal.id).to.equal(1);
      expect(proposal.description).to.equal("Proposal 1");
      expect(proposal.voteCount).to.equal(1);
      expect(proposal.proposer).to.equal(owner.address);
      expect(proposal.executed).to.be.false;
    });
    
    it("Should get all proposal IDs", async function () {
      const ids = await voting.getAllProposalIds();
      
      expect(ids.length).to.equal(2);
      expect(ids[0]).to.equal(1);
      expect(ids[1]).to.equal(2);
    });
    
    it("Should check if proposal is active", async function () {
      expect(await voting.isProposalActive(1)).to.be.true;
      
      await time.increase(8 * 24 * 60 * 60);
      expect(await voting.isProposalActive(1)).to.be.false;
    });
  });
  
  describe("Proposal Execution", function () {
    beforeEach(async function () {
      await voting.createProposal("Test", 7);
      await voting.connect(voter1).vote(1);
    });
    
    it("Should execute proposal after deadline", async function () {
      await time.increase(8 * 24 * 60 * 60);
      
      await expect(voting.executeProposal(1))
        .to.emit(voting, "ProposalExecuted")
        .withArgs(1);
      
      const proposal = await voting.getProposal(1);
      expect(proposal.executed).to.be.true;
    });
    
    it("Should prevent execution before deadline", async function () {
      await expect(voting.executeProposal(1))
        .to.be.revertedWith("Voting not ended");
    });
    
    it("Should prevent double execution", async function () {
      await time.increase(8 * 24 * 60 * 60);
      await voting.executeProposal(1);
      
      await expect(voting.executeProposal(1))
        .to.be.revertedWith("Already executed");
    });
    
    it("Only owner can execute", async function () {
      await time.increase(8 * 24 * 60 * 60);
      
      await expect(voting.connect(voter1).executeProposal(1))
        .to.be.revertedWith("Not the owner");
    });
  });
});
```

**运行测试：**

```powershell
# 运行所有测试
npx hardhat test

# 查看测试覆盖率
npx hardhat coverage

# 运行特定测试
npx hardhat test --grep "Voting"
```

**提交代码：**

```powershell
git add test/
git commit -m "test: Add comprehensive tests for Voting contract"
```

---

### Part 4: 部署脚本 (30分钟)

**创建部署脚本：**

```javascript
// scripts/deploy.js
const hre = require("hardhat");
const fs = require("fs");

async function main() {
  console.log("=" .repeat(50));
  console.log("🚀 开始部署Voting合约");
  console.log("=" .repeat(50));
  
  // 获取部署账户
  const [deployer] = await hre.ethers.getSigners();
  console.log("\n📝 部署账户:", deployer.address);
  
  const balance = await hre.ethers.provider.getBalance(deployer.address);
  console.log("💰 账户余额:", hre.ethers.formatEther(balance), "ETH");
  
  // 部署合约
  console.log("\n⏳ 正在部署合约...");
  const openToAll = true;  // 对所有人开放投票
  
  const Voting = await hre.ethers.getContractFactory("Voting");
  const voting = await Voting.deploy(openToAll);
  
  await voting.waitForDeployment();
  const address = await voting.getAddress();
  
  console.log("✅ 合约部署成功！");
  console.log("📍 合约地址:", address);
  console.log("👤 所有者:", await voting.owner());
  console.log("🌍 开放模式:", await voting.openToAll() ? "是" : "否");
  
  // 创建示例提案
  console.log("\n📋 创建示例提案...");
  const proposals = [
    { description: "是否升级合约到v2版本？", duration: 7 },
    { description: "是否增加新功能X？", duration: 14 },
    { description: "是否调整gas费用？", duration: 3 }
  ];
  
  for (let i = 0; i < proposals.length; i++) {
    const tx = await voting.createProposal(
      proposals[i].description,
      proposals[i].duration
    );
    await tx.wait();
    console.log(`✓ 提案 ${i + 1}: ${proposals[i].description}`);
  }
  
  // 保存部署信息
  const deploymentInfo = {
    network: hre.network.name,
    contractAddress: address,
    owner: deployer.address,
    openToAll: openToAll,
    deployedAt: new Date().toISOString(),
    blockNumber: await hre.ethers.provider.getBlockNumber(),
    initialProposals: proposals.length
  };
  
  const deploymentPath = "./deployment.json";
  fs.writeFileSync(
    deploymentPath,
    JSON.stringify(deploymentInfo, null, 2)
  );
  
  console.log("\n💾 部署信息已保存到:", deploymentPath);
  
  // 保存合约ABI和地址给前端使用
  const contractInfo = {
    address: address,
    abi: JSON.parse(
      fs.readFileSync("./artifacts/contracts/Voting.sol/Voting.json", "utf8")
    ).abi
  };
  
  if (!fs.existsSync("./frontend")) {
    fs.mkdirSync("./frontend");
  }
  
  fs.writeFileSync(
    "./frontend/contract.json",
    JSON.stringify(contractInfo, null, 2)
  );
  
  console.log("📄 合约ABI和地址已保存到: ./frontend/contract.json");
  
  console.log("\n" + "=".repeat(50));
  console.log("🎉 部署完成！");
  console.log("=".repeat(50));
  
  // 打印下一步指令
  console.log("\n📌 下一步：");
  console.log("1. 启动前端: 打开 frontend/index.html");
  console.log("2. 或使用console交互:");
  console.log(`   npx hardhat console --network ${hre.network.name}`);
  console.log(`   const voting = await ethers.getContractAt("Voting", "${address}")`);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error("❌ 部署失败:", error);
    process.exit(1);
  });
```

**交互脚本：**

```javascript
// scripts/interact.js
const hre = require("hardhat");
const fs = require("fs");

async function main() {
  // 读取部署信息
  const deployment = JSON.parse(fs.readFileSync("./deployment.json", "utf8"));
  const contractAddress = deployment.contractAddress;
  
  console.log("连接到合约:", contractAddress);
  
  // 连接到合约
  const voting = await hre.ethers.getContractAt("Voting", contractAddress);
  const [owner, voter1, voter2] = await hre.ethers.getSigners();
  
  // 查看提案数量
  const count = await voting.proposalCount();
  console.log("\n当前提案数:", count.toString());
  
  // 获取所有提案
  const ids = await voting.getAllProposalIds();
  console.log("\n所有提案:");
  
  for (const id of ids) {
    const proposal = await voting.getProposal(id);
    const isActive = await voting.isProposalActive(id);
    const deadline = new Date(Number(proposal.deadline) * 1000);
    
    console.log(`\n提案 #${id}:`);
    console.log(`  描述: ${proposal.description}`);
    console.log(`  票数: ${proposal.voteCount}`);
    console.log(`  截止: ${deadline.toLocaleString()}`);
    console.log(`  状态: ${isActive ? "进行中" : "已结束"}`);
    console.log(`  提案人: ${proposal.proposer}`);
  }
  
  // voter1 投票给提案1
  console.log("\n\n测试投票功能...");
  console.log(`${voter1.address} 投票给提案1`);
  const tx1 = await voting.connect(voter1).vote(1);
  await tx1.wait();
  console.log("✓ 投票成功");
  
  // voter2 投票给提案1
  console.log(`${voter2.address} 投票给提案1`);
  const tx2 = await voting.connect(voter2).vote(1);
  await tx2.wait();
  console.log("✓ 投票成功");
  
  // 查看更新后的提案
  const updated = await voting.getProposal(1);
  console.log(`\n提案1当前票数: ${updated.voteCount}`);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

**部署合约：**

```powershell
# 启动本地节点（终端1）
npx hardhat node

# 部署合约（终端2）
npx hardhat run scripts/deploy.js --network localhost

# 交互测试
npx hardhat run scripts/interact.js --network localhost
```

**提交代码：**

```powershell
git add scripts/
git commit -m "feat: Add deployment and interaction scripts"
```

---

## 🎨 Day 7: 前端开发 (5-6小时)

### Part 1: HTML结构 (1小时)

```html
<!-- frontend/index.html -->
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>投票DApp - Voting DApp</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <!-- 头部 -->
    <header>
        <div class="container">
            <h1>🗳️ 投票 DApp</h1>
            <div class="wallet-info">
                <button id="connectWallet" class="btn btn-primary">连接钱包</button>
                <div id="accountInfo" class="account-info" style="display: none;">
                    <span id="accountAddress"></span>
                    <span id="accountBalance"></span>
                </div>
            </div>
        </div>
    </header>

    <!-- 主内容 -->
    <main class="container">
        <!-- 创建提案部分 -->
        <section class="create-proposal-section">
            <h2>创建新提案</h2>
            <div class="form-group">
                <label for="proposalDescription">提案描述：</label>
                <textarea
                    id="proposalDescription"
                    placeholder="输入提案描述..."
                    rows="3"
                ></textarea>
            </div>
            <div class="form-group">
                <label for="proposalDuration">投票持续时间（天）：</label>
                <input
                    type="number"
                    id="proposalDuration"
                    placeholder="7"
                    min="1"
                    value="7"
                >
            </div>
            <button id="createProposal" class="btn btn-success">创建提案</button>
        </section>

        <!-- 提案列表 -->
        <section class="proposals-section">
            <div class="section-header">
                <h2>所有提案</h2>
                <button id="refreshProposals" class="btn btn-secondary">刷新</button>
            </div>
            
            <div id="proposalsList" class="proposals-list">
                <!-- 动态加载提案 -->
                <div class="loading">加载中...</div>
            </div>
        </section>
    </main>

    <!-- 提示消息 -->
    <div id="toast" class="toast"></div>

    <!-- 引入ethers.js -->
    <script src="https://cdn.ethers.io/lib/ethers-5.7.2.umd.min.js"></script>
    <!-- 引入应用脚本 -->
    <script src="app.js"></script>
</body>
</html>
```

### Part 2: CSS样式 (1小时)

```css
/* frontend/style.css */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

:root {
    --primary-color: #4f46e5;
    --success-color: #10b981;
    --danger-color: #ef4444;
    --warning-color: #f59e0b;
    --secondary-color: #6b7280;
    --bg-color: #f3f4f6;
    --card-bg: #ffffff;
    --text-color: #1f2937;
    --border-color: #e5e7eb;
}

body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
    background-color: var(--bg-color);
    color: var(--text-color);
    line-height: 1.6;
}

.container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 20px;
}

/* 头部样式 */
header {
    background: linear-gradient(135deg, var(--primary-color), #7c3aed);
    color: white;
    padding: 20px 0;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

header .container {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

header h1 {
    font-size: 28px;
    font-weight: 700;
}

/* 钱包信息 */
.wallet-info {
    display: flex;
    align-items: center;
    gap: 15px;
}

.account-info {
    background: rgba(255,255,255,0.2);
    padding: 10px 15px;
    border-radius: 8px;
    display: flex;
    flex-direction: column;
    align-items: flex-end;
}

.account-info span {
    font-size: 14px;
}

#accountAddress {
    font-weight: 600;
}

#accountBalance {
    opacity: 0.9;
    font-size: 12px;
}

/* 按钮样式 */
.btn {
    padding: 10px 20px;
    border: none;
    border-radius: 8px;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
}

.btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 8px rgba(0,0,0,0.2);
}

.btn-primary {
    background-color: white;
    color: var(--primary-color);
}

.btn-success {
    background-color: var(--success-color);
    color: white;
}

.btn-secondary {
    background-color: var(--secondary-color);
    color: white;
}

.btn-vote {
    background-color: var(--primary-color);
    color: white;
    padding: 8px 16px;
}

.btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
    transform: none;
}

/* 主内容 */
main {
    padding: 40px 20px;
}

/* 创建提案部分 */
.create-proposal-section {
    background: var(--card-bg);
    padding: 30px;
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    margin-bottom: 40px;
}

.create-proposal-section h2 {
    margin-bottom: 20px;
    color: var(--text-color);
}

.form-group {
    margin-bottom: 20px;
}

.form-group label {
    display: block;
    margin-bottom: 8px;
    font-weight: 600;
    color: var(--text-color);
}

.form-group textarea,
.form-group input {
    width: 100%;
    padding: 12px;
    border: 2px solid var(--border-color);
    border-radius: 8px;
    font-size: 14px;
    font-family: inherit;
    transition: border-color 0.3s ease;
}

.form-group textarea:focus,
.form-group input:focus {
    outline: none;
    border-color: var(--primary-color);
}

/* 提案列表 */
.proposals-section {
    background: var(--card-bg);
    padding: 30px;
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.section-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
}

.proposals-list {
    display: grid;
    gap: 20px;
}

/* 提案卡片 */
.proposal-card {
    background: var(--bg-color);
    border: 2px solid var(--border-color);
    border-radius: 12px;
    padding: 20px;
    transition: all 0.3s ease;
}

.proposal-card:hover {
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
    transform: translateY(-2px);
}

.proposal-header {
    display: flex;
    justify-content: space-between;
    align-items: start;
    margin-bottom: 15px;
}

.proposal-id {
    background: var(--primary-color);
    color: white;
    padding: 4px 12px;
    border-radius: 20px;
    font-size: 12px;
    font-weight: 600;
}

.proposal-status {
    padding: 4px 12px;
    border-radius: 20px;
    font-size: 12px;
    font-weight: 600;
}

.status-active {
    background-color: var(--success-color);
    color: white;
}

.status-ended {
    background-color: var(--secondary-color);
    color: white;
}

.proposal-description {
    font-size: 16px;
    margin-bottom: 15px;
    line-height: 1.6;
}

.proposal-info {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
    gap: 15px;
    margin-bottom: 15px;
    padding: 15px;
    background: var(--card-bg);
    border-radius: 8px;
}

.info-item {
    display: flex;
    flex-direction: column;
}

.info-label {
    font-size: 12px;
    color: var(--secondary-color);
    margin-bottom: 4px;
}

.info-value {
    font-weight: 600;
    color: var(--text-color);
}

.proposal-footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.vote-count {
    font-size: 18px;
    font-weight: 700;
    color: var(--primary-color);
}

/* Toast消息 */
.toast {
    position: fixed;
    bottom: 20px;
    right: 20px;
    background: var(--text-color);
    color: white;
    padding: 15px 20px;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.3);
    transform: translateX(400px);
    transition: transform 0.3s ease;
    z-index: 1000;
}

.toast.show {
    transform: translateX(0);
}

.toast.success {
    background: var(--success-color);
}

.toast.error {
    background: var(--danger-color);
}

/* 加载状态 */
.loading {
    text-align: center;
    padding: 40px;
    color: var(--secondary-color);
}

/* 响应式设计 */
@media (max-width: 768px) {
    header .container {
        flex-direction: column;
        gap: 15px;
    }
    
    .section-header {
        flex-direction: column;
        align-items: flex-start;
        gap: 10px;
    }
    
    .proposal-header {
        flex-direction: column;
        gap: 10px;
    }
    
    .proposal-footer {
        flex-direction: column;
        gap: 10px;
        align-items: flex-start;
    }
}
```

### Part 3: JavaScript交互 (3小时)

```javascript
// frontend/app.js
// 合约配置
let contract;
let provider;
let signer;
let userAddress;

// 从contract.json加载合约信息
let contractAddress;
let contractABI;

// 初始化
async function init() {
    try {
        // 加载合约配置
        const response = await fetch('contract.json');
        const contractInfo = await response.json();
        contractAddress = contractInfo.address;
        contractABI = contractInfo.abi;
        
        console.log("合约地址:", contractAddress);
        
        // 绑定事件
        document.getElementById('connectWallet').addEventListener('click', connectWallet);
        document.getElementById('createProposal').addEventListener('click', createProposal);
        document.getElementById('refreshProposals').addEventListener('click', loadProposals);
        
        // 检查是否已连接
        if (typeof window.ethereum !== 'undefined') {
            const accounts = await window.ethereum.request({ method: 'eth_accounts' });
            if (accounts.length > 0) {
                await connectWallet();
            }
        }
    } catch (error) {
        console.error("初始化失败:", error);
        showToast("初始化失败: " + error.message, 'error');
    }
}

// 连接钱包
async function connectWallet() {
    if (typeof window.ethereum === 'undefined') {
        showToast('请安装MetaMask!', 'error');
        return;
    }
    
    try {
        // 请求账户访问
        await window.ethereum.request({ method: 'eth_requestAccounts' });
        
        // 创建provider和signer
        provider = new ethers.providers.Web3Provider(window.ethereum);
        signer = provider.getSigner();
        userAddress = await signer.getAddress();
        
        // 创建合约实例
        contract = new ethers.Contract(contractAddress, contractABI, signer);
        
        // 更新UI
        document.getElementById('connectWallet').style.display = 'none';
        document.getElementById('accountInfo').style.display = 'flex';
        document.getElementById('accountAddress').textContent = 
            userAddress.slice(0, 6) + '...' + userAddress.slice(-4);
        
        // 获取余额
        const balance = await provider.getBalance(userAddress);
        document.getElementById('accountBalance').textContent = 
            parseFloat(ethers.utils.formatEther(balance)).toFixed(4) + ' ETH';
        
        showToast('钱包连接成功!', 'success');
        
        // 加载提案
        await loadProposals();
        
        // 监听账户变化
        window.ethereum.on('accountsChanged', (accounts) => {
            if (accounts.length === 0) {
                location.reload();
            } else {
                location.reload();
            }
        });
        
        // 监听网络变化
        window.ethereum.on('chainChanged', () => {
            location.reload();
        });
        
    } catch (error) {
        console.error("连接钱包失败:", error);
        showToast('连接钱包失败: ' + error.message, 'error');
    }
}

// 创建提案
async function createProposal() {
    if (!contract) {
        showToast('请先连接钱包!', 'error');
        return;
    }
    
    const description = document.getElementById('proposalDescription').value.trim();
    const duration = parseInt(document.getElementById('proposalDuration').value);
    
    if (!description) {
        showToast('请输入提案描述!', 'error');
        return;
    }
    
    if (!duration || duration <= 0) {
        showToast('请输入有效的持续时间!', 'error');
        return;
    }
    
    try {
        showToast('正在创建提案...', 'info');
        
        const tx = await contract.createProposal(description, duration);
        showToast('交易已提交，等待确认...', 'info');
        
        await tx.wait();
        showToast('提案创建成功!', 'success');
        
        // 清空表单
        document.getElementById('proposalDescription').value = '';
        document.getElementById('proposalDuration').value = '7';
        
        // 刷新提案列表
        await loadProposals();
        
    } catch (error) {
        console.error("创建提案失败:", error);
        showToast('创建提案失败: ' + error.message, 'error');
    }
}

// 加载提案
async function loadProposals() {
    if (!contract) {
        return;
    }
    
    const proposalsList = document.getElementById('proposalsList');
    proposalsList.innerHTML = '<div class="loading">加载中...</div>';
    
    try {
        const proposalCount = await contract.proposalCount();
        
        if (proposalCount.toNumber() === 0) {
            proposalsList.innerHTML = '<div class="loading">暂无提案</div>';
            return;
        }
        
        const proposals = [];
        
        for (let i = 1; i <= proposalCount.toNumber(); i++) {
            const proposal = await contract.getProposal(i);
            const isActive = await contract.isProposalActive(i);
            const hasVoted = userAddress ? 
                await contract.hasVoted(i, userAddress) : false;
            
            proposals.push({
                id: proposal.id.toNumber(),
                description: proposal.description,
                voteCount: proposal.voteCount.toNumber(),
                deadline: new Date(proposal.deadline.toNumber() * 1000),
                proposer: proposal.proposer,
                executed: proposal.executed,
                isActive: isActive,
                hasVoted: hasVoted
            });
        }
        
        // 按ID降序排列（最新的在前）
        proposals.sort((a, b) => b.id - a.id);
        
        // 渲染提案
        proposalsList.innerHTML = proposals.map(proposal => 
            renderProposal(proposal)
        ).join('');
        
        // 绑定投票按钮事件
        proposals.forEach(proposal => {
            const voteBtn = document.getElementById(`vote-${proposal.id}`);
            if (voteBtn) {
                voteBtn.addEventListener('click', () => vote(proposal.id));
            }
        });
        
    } catch (error) {
        console.error("加载提案失败:", error);
        proposalsList.innerHTML = '<div class="loading">加载失败</div>';
        showToast('加载提案失败: ' + error.message, 'error');
    }
}

// 渲染单个提案
function renderProposal(proposal) {
    const statusClass = proposal.isActive ? 'status-active' : 'status-ended';
    const statusText = proposal.isActive ? '进行中' : '已结束';
    
    const canVote = proposal.isActive && !proposal.hasVoted && userAddress;
    
    return `
        <div class="proposal-card">
            <div class="proposal-header">
                <span class="proposal-id">#${proposal.id}</span>
                <span class="proposal-status ${statusClass}">${statusText}</span>
            </div>
            
            <div class="proposal-description">
                ${escapeHtml(proposal.description)}
            </div>
            
            <div class="proposal-info">
                <div class="info-item">
                    <span class="info-label">票数</span>
                    <span class="info-value">${proposal.voteCount} 票</span>
                </div>
                <div class="info-item">
                    <span class="info-label">截止时间</span>
                    <span class="info-value">${formatDate(proposal.deadline)}</span>
                </div>
                <div class="info-item">
                    <span class="info-label">提案人</span>
                    <span class="info-value">${formatAddress(proposal.proposer)}</span>
                </div>
            </div>
            
            <div class="proposal-footer">
                <div class="vote-count">
                    🗳️ ${proposal.voteCount} 票
                </div>
                ${canVote ? 
                    `<button id="vote-${proposal.id}" class="btn btn-vote">投票</button>` :
                    proposal.hasVoted ? 
                        `<button class="btn btn-vote" disabled>已投票</button>` :
                        `<button class="btn btn-vote" disabled>${statusText}</button>`
                }
            </div>
        </div>
    `;
}

// 投票
async function vote(proposalId) {
    if (!contract) {
        showToast('请先连接钱包!', 'error');
        return;
    }
    
    try {
        showToast('正在投票...', 'info');
        
        const tx = await contract.vote(proposalId);
        showToast('交易已提交，等待确认...', 'info');
        
        await tx.wait();
        showToast('投票成功!', 'success');
        
        // 刷新提案列表
        await loadProposals();
        
    } catch (error) {
        console.error("投票失败:", error);
        let errorMessage = '投票失败';
        
        if (error.message.includes('Already voted')) {
            errorMessage = '您已经投过票了';
        } else if (error.message.includes('Voting has ended')) {
            errorMessage = '投票已结束';
        } else if (error.message.includes('No voting right')) {
            errorMessage = '您没有投票权限';
        }
        
        showToast(errorMessage, 'error');
    }
}

// 工具函数
function showToast(message, type = 'info') {
    const toast = document.getElementById('toast');
    toast.textContent = message;
    toast.className = `toast ${type} show`;
    
    setTimeout(() => {
        toast.classList.remove('show');
    }, 3000);
}

function formatAddress(address) {
    return address.slice(0, 6) + '...' + address.slice(-4);
}

function formatDate(date) {
    return date.toLocaleString('zh-CN', {
        year: 'numeric',
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit'
    });
}

function escapeHtml(text) {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}

// 页面加载时初始化
window.addEventListener('load', init);
```

**提交代码：**

```powershell
git add frontend/
git commit -m "feat: Add frontend with HTML, CSS and JavaScript"
```

---

## 📝 完整测试流程

### 测试步骤

**1. 启动本地网络**
```powershell
# 终端1
npx hardhat node
```

**2. 部署合约**
```powershell
# 终端2
npx hardhat run scripts/deploy.js --network localhost
```

**3. 配置MetaMask**
```
1. 打开MetaMask
2. 添加网络:
   - 网络名称: Hardhat Local
   - RPC URL: http://127.0.0.1:8545
   - 链ID: 31337
   - 货币符号: ETH
3. 导入Hardhat账户:
   - 使用Hardhat node显示的私钥
```

**4. 打开前端**
```powershell
# 使用任意HTTP服务器
# 方法1: Python
python -m http.server 8000

# 方法2: Node.js (安装http-server)
npx http-server frontend -p 8000

# 访问: http://localhost:8000
```

**5. 测试功能**
```
[ ] 连接MetaMask
[ ] 创建提案
[ ] 查看提案列表
[ ] 投票
[ ] 切换账户测试
[ ] 测试投票截止
```

---

## 📋 项目文档

**创建README.md：**

```markdown
# 投票 DApp

一个基于以太坊的去中心化投票应用。

## 功能特性

- ✅ 创建投票提案
- ✅ 用户投票
- ✅ 实时查看投票结果
- ✅ 投票截止时间控制
- ✅ 投票权限管理

## 技术栈

- Solidity ^0.8.20
- Hardhat
- Ethers.js
- HTML/CSS/JavaScript

## 安装和运行

### 1. 安装依赖
\`\`\`bash
npm install
\`\`\`

### 2. 启动本地节点
\`\`\`bash
npx hardhat node
\`\`\`

### 3. 部署合约
\`\`\`bash
npx hardhat run scripts/deploy.js --network localhost
\`\`\`

### 4. 配置MetaMask
- 添加Hardhat网络 (http://127.0.0.1:8545, 链ID: 31337)
- 导入Hardhat测试账户

### 5. 启动前端
\`\`\`bash
npx http-server frontend -p 8000
\`\`\`

访问: http://localhost:8000

## 测试

\`\`\`bash
# 运行所有测试
npx hardhat test

# 查看覆盖率
npx hardhat coverage
\`\`\`

## 合约地址

部署后的合约地址保存在 `deployment.json` 文件中。

## 项目结构

\`\`\`
voting-dapp/
├── contracts/          # 智能合约
├── scripts/           # 部署脚本
├── test/              # 测试文件
├── frontend/          # 前端文件
└── hardhat.config.js  # Hardhat配置
\`\`\`

## License

MIT
```

**最终提交：**

```powershell
git add README.md
git commit -m "docs: Add comprehensive README"

# 添加标签
git tag -a v1.0.0 -m "Version 1.0.0: Complete Voting DApp"
```

---

## ✅ 项目检查清单

### 智能合约
- [ ] Voting.sol 编写完成
- [ ] 合约编译通过
- [ ] 测试覆盖率 > 80%
- [ ] 部署脚本正常工作

### 前端
- [ ] HTML结构完整
- [ ] CSS样式美观
- [ ] JavaScript交互正常
- [ ] MetaMask集成成功
- [ ] 所有功能可用

### 文档
- [ ] README.md 完整
- [ ] 代码注释充分
- [ ] 部署说明清晰

### Git管理
- [ ] .gitignore 配置正确
- [ ] 提交信息清晰
- [ ] 代码已打标签

---

## 🎯 扩展挑战（可选）

1. **添加提案类别功能**
   - 给提案添加分类
   - 按类别筛选提案

2. **投票权重系统**
   - 根据持币数量计算投票权重
   - 实现加权投票

3. **提案评论功能**
   - 允许用户评论提案
   - 存储评论到IPFS

4. **投票结果可视化**
   - 使用Chart.js绘制图表
   - 显示投票趋势

5. **移动端优化**
   - 响应式设计优化
   - PWA支持

---

## 📝 学习总结

**完成日期**: ___________
**实际耗时**: _____ 小时

### ✅ 完成情况
- [ ] 智能合约开发
- [ ] 合约测试
- [ ] 部署脚本
- [ ] 前端开发
- [ ] 完整测试
- [ ] 项目文档

### 💡 项目亮点
1. 实现的核心功能:
2. 解决的技术难点:
3. 代码质量:

### 🎓 学到的知识
1. Solidity 开发:
2. Hardhat 使用:
3. 前端集成:
4. 调试技巧:

### 🤔 遇到的问题
- 问题1及解决方案:
- 问题2及解决方案:

### 📈 下周计划
- [ ] 学习Solidity高级特性
- [ ] 深入测试框架
- [ ] 了解gas优化
- [ ] 准备Week 2学习

**🎉 恭喜完成Week 1！准备进入Week 2学习Solidity编程！**
