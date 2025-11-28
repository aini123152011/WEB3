# Week 2 - Day 6-7: 众筹合约综合项目

**学习日期**: ___________
**预计用时**: 12-14小时  
**难度等级**: ⭐⭐⭐⭐⭐ (综合项目)

## 📚 项目目标

实现一个功能完整的去中心化众筹平台，包含：
- ✅ 众筹项目创建和管理
- ✅ 多级里程碑支持
- ✅ 投资者权益保护
- ✅ ERC20代币奖励
- ✅ NFT纪念徽章
- ✅ 前端交互界面

---

## 🎯 Day 6: 智能合约开发 (6-7小时)

### Part 1: 项目分析和架构设计 (1小时)

#### 1.1 功能需求

```
核心功能：
1. 项目创建
   - 设置众筹目标
   - 设置截止时间
   - 定义里程碑
   - 设置奖励方案

2. 投资管理
   - 用户投资
   - 投资记录
   - 退款机制
   - 投资证明NFT

3. 项目管理
   - 状态跟踪
   - 里程碑验证
   - 资金提取
   - 项目更新

4. 治理机制
   - 投资者投票
   - 里程碑审批
   - 项目取消

5. 激励系统
   - 代币奖励
   - NFT徽章
   - 分红机制
```

#### 1.2 合约架构

```
CrowdfundingPlatform（主合约）
├── Project（项目管理）
├── Investment（投资管理）
├── Milestone（里程碑）
└── Reward（奖励系统）

RewardToken（ERC20代币）
└── 平台代币奖励

BadgeNFT（ERC721）
└── 投资纪念徽章
```

---

### Part 2: 核心合约实现 (3小时)

#### 2.1 奖励代币合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * 众筹平台奖励代币
 */
contract RewardToken is ERC20, Ownable {
    constructor() ERC20("Crowdfund Reward Token", "CRT") Ownable(msg.sender) {
        // 初始供应1000万
        _mint(msg.sender, 10_000_000 * 10 ** decimals());
    }
    
    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }
}
```

#### 2.2 徽章NFT合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * 投资纪念徽章NFT
 */
contract BadgeNFT is ERC721, Ownable {
    uint256 private _nextTokenId;
    
    struct BadgeInfo {
        uint256 projectId;
        uint256 investmentAmount;
        uint256 timestamp;
        string tier; // Bronze, Silver, Gold, Platinum
    }
    
    mapping(uint256 => BadgeInfo) public badges;
    
    constructor() ERC721("Crowdfunding Badge", "CFB") Ownable(msg.sender) {}
    
    function mint(
        address to,
        uint256 projectId,
        uint256 investmentAmount,
        string memory tier
    ) external onlyOwner returns (uint256) {
        uint256 tokenId = _nextTokenId++;
        _safeMint(to, tokenId);
        
        badges[tokenId] = BadgeInfo({
            projectId: projectId,
            investmentAmount: investmentAmount,
            timestamp: block.timestamp,
            tier: tier
        });
        
        return tokenId;
    }
    
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        _requireOwned(tokenId);
        
        BadgeInfo memory badge = badges[tokenId];
        
        return string(abi.encodePacked(
            "https://api.crowdfund.example/badge/",
            _toString(tokenId),
            "?project=", _toString(badge.projectId),
            "&tier=", badge.tier
        ));
    }
    
    function _toString(uint256 value) internal pure returns (string memory) {
        if (value == 0) return "0";
        
        uint256 temp = value;
        uint256 digits;
        while (temp != 0) {
            digits++;
            temp /= 10;
        }
        
        bytes memory buffer = new bytes(digits);
        while (value != 0) {
            digits -= 1;
            buffer[digits] = bytes1(uint8(48 + uint256(value % 10)));
            value /= 10;
        }
        
        return string(buffer);
    }
}
```

#### 2.3 众筹主合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * 众筹平台主合约
 */
contract CrowdfundingPlatform is ReentrancyGuard, Ownable {
    RewardToken public rewardToken;
    BadgeNFT public badgeNFT;
    
    uint256 private _nextProjectId;
    uint256 public platformFeeRate = 300; // 3%
    
    enum ProjectState {
        Fundraising,    // 募资中
        Failed,         // 募资失败
        Successful,     // 募资成功
        InProgress,     // 执行中
        Completed,      // 已完成
        Cancelled       // 已取消
    }
    
    struct Milestone {
        string description;
        uint256 amount;
        uint256 deadline;
        bool approved;
        bool completed;
        uint256 approvalVotes;
        uint256 totalVoters;
    }
    
    struct Project {
        uint256 id;
        address creator;
        string title;
        string description;
        uint256 goal;
        uint256 raised;
        uint256 deadline;
        ProjectState state;
        Milestone[] milestones;
        uint256 currentMilestone;
    }
    
    struct Investment {
        address investor;
        uint256 amount;
        uint256 timestamp;
        bool refunded;
    }
    
    mapping(uint256 => Project) public projects;
    mapping(uint256 => Investment[]) public projectInvestments;
    mapping(uint256 => mapping(address => uint256)) public investorAmounts;
    mapping(uint256 => mapping(uint256 => mapping(address => bool))) public milestoneVotes;
    
    event ProjectCreated(uint256 indexed projectId, address indexed creator, string title, uint256 goal);
    event InvestmentMade(uint256 indexed projectId, address indexed investor, uint256 amount);
    event MilestoneApproved(uint256 indexed projectId, uint256 milestoneIndex);
    event FundsWithdrawn(uint256 indexed projectId, uint256 amount);
    event RefundIssued(uint256 indexed projectId, address indexed investor, uint256 amount);
    event ProjectStateChanged(uint256 indexed projectId, ProjectState newState);
    
    constructor(address _rewardToken, address _badgeNFT) Ownable(msg.sender) {
        rewardToken = RewardToken(_rewardToken);
        badgeNFT = BadgeNFT(_badgeNFT);
    }
    
    // 创建项目
    function createProject(
        string memory title,
        string memory description,
        uint256 goal,
        uint256 duration,
        string[] memory milestoneDescriptions,
        uint256[] memory milestoneAmounts,
        uint256[] memory milestoneDeadlines
    ) external returns (uint256) {
        require(goal > 0, "Goal must be positive");
        require(duration > 0, "Duration must be positive");
        require(
            milestoneDescriptions.length == milestoneAmounts.length &&
            milestoneAmounts.length == milestoneDeadlines.length,
            "Milestone arrays length mismatch"
        );
        
        uint256 projectId = _nextProjectId++;
        Project storage project = projects[projectId];
        
        project.id = projectId;
        project.creator = msg.sender;
        project.title = title;
        project.description = description;
        project.goal = goal;
        project.deadline = block.timestamp + duration;
        project.state = ProjectState.Fundraising;
        
        // 添加里程碑
        uint256 totalMilestoneAmount = 0;
        for (uint256 i = 0; i < milestoneDescriptions.length; i++) {
            project.milestones.push(Milestone({
                description: milestoneDescriptions[i],
                amount: milestoneAmounts[i],
                deadline: milestoneDeadlines[i],
                approved: false,
                completed: false,
                approvalVotes: 0,
                totalVoters: 0
            }));
            totalMilestoneAmount += milestoneAmounts[i];
        }
        
        require(totalMilestoneAmount == goal, "Total milestone amount must equal goal");
        
        emit ProjectCreated(projectId, msg.sender, title, goal);
        return projectId;
    }
    
    // 投资项目
    function invest(uint256 projectId) external payable nonReentrant {
        Project storage project = projects[projectId];
        require(project.state == ProjectState.Fundraising, "Not in fundraising state");
        require(block.timestamp < project.deadline, "Fundraising ended");
        require(msg.value > 0, "Investment must be positive");
        
        project.raised += msg.value;
        investorAmounts[projectId][msg.sender] += msg.value;
        
        projectInvestments[projectId].push(Investment({
            investor: msg.sender,
            amount: msg.value,
            timestamp: block.timestamp,
            refunded: false
        }));
        
        emit InvestmentMade(projectId, msg.sender, msg.value);
        
        // 发放奖励代币（1:100比例）
        uint256 rewardAmount = msg.value * 100;
        rewardToken.mint(msg.sender, rewardAmount);
        
        // 铸造徽章NFT
        string memory tier = _determineBadgeTier(msg.value);
        badgeNFT.mint(msg.sender, projectId, msg.value, tier);
        
        // 检查是否达到目标
        if (project.raised >= project.goal) {
            project.state = ProjectState.Successful;
            emit ProjectStateChanged(projectId, ProjectState.Successful);
        }
    }
    
    // 确定徽章等级
    function _determineBadgeTier(uint256 amount) private pure returns (string memory) {
        if (amount >= 10 ether) return "Platinum";
        if (amount >= 5 ether) return "Gold";
        if (amount >= 1 ether) return "Silver";
        return "Bronze";
    }
    
    // 检查项目状态
    function checkProjectStatus(uint256 projectId) external {
        Project storage project = projects[projectId];
        
        if (project.state == ProjectState.Fundraising && 
            block.timestamp >= project.deadline) {
            if (project.raised >= project.goal) {
                project.state = ProjectState.Successful;
            } else {
                project.state = ProjectState.Failed;
            }
            emit ProjectStateChanged(projectId, project.state);
        }
    }
    
    // 投资者投票批准里程碑
    function approveMilestone(uint256 projectId, uint256 milestoneIndex) external {
        Project storage project = projects[projectId];
        require(project.state == ProjectState.Successful || 
                project.state == ProjectState.InProgress, 
                "Invalid project state");
        require(milestoneIndex < project.milestones.length, "Invalid milestone");
        require(investorAmounts[projectId][msg.sender] > 0, "Not an investor");
        require(!milestoneVotes[projectId][milestoneIndex][msg.sender], "Already voted");
        
        Milestone storage milestone = project.milestones[milestoneIndex];
        require(!milestone.approved, "Already approved");
        
        milestoneVotes[projectId][milestoneIndex][msg.sender] = true;
        milestone.approvalVotes += investorAmounts[projectId][msg.sender];
        milestone.totalVoters++;
        
        // 检查是否达到批准阈值（超过50%投资额）
        if (milestone.approvalVotes * 2 > project.raised) {
            milestone.approved = true;
            emit MilestoneApproved(projectId, milestoneIndex);
        }
    }
    
    // 项目方提取里程碑资金
    function withdrawMilestoneFunds(uint256 projectId, uint256 milestoneIndex) 
        external 
        nonReentrant 
    {
        Project storage project = projects[projectId];
        require(msg.sender == project.creator, "Not project creator");
        require(project.state == ProjectState.Successful || 
                project.state == ProjectState.InProgress, 
                "Invalid state");
        require(milestoneIndex < project.milestones.length, "Invalid milestone");
        
        Milestone storage milestone = project.milestones[milestoneIndex];
        require(milestone.approved, "Milestone not approved");
        require(!milestone.completed, "Already withdrawn");
        
        milestone.completed = true;
        
        if (project.state == ProjectState.Successful) {
            project.state = ProjectState.InProgress;
        }
        
        // 计算平台手续费
        uint256 fee = (milestone.amount * platformFeeRate) / 10000;
        uint256 amountAfterFee = milestone.amount - fee;
        
        // 转账
        payable(project.creator).transfer(amountAfterFee);
        payable(owner()).transfer(fee);
        
        emit FundsWithdrawn(projectId, amountAfterFee);
        
        // 检查所有里程碑是否完成
        bool allCompleted = true;
        for (uint256 i = 0; i < project.milestones.length; i++) {
            if (!project.milestones[i].completed) {
                allCompleted = false;
                break;
            }
        }
        
        if (allCompleted) {
            project.state = ProjectState.Completed;
            emit ProjectStateChanged(projectId, ProjectState.Completed);
        }
    }
    
    // 退款（项目失败时）
    function refund(uint256 projectId) external nonReentrant {
        Project storage project = projects[projectId];
        require(project.state == ProjectState.Failed, "Project not failed");
        
        uint256 investmentAmount = investorAmounts[projectId][msg.sender];
        require(investmentAmount > 0, "No investment found");
        
        investorAmounts[projectId][msg.sender] = 0;
        
        payable(msg.sender).transfer(investmentAmount);
        
        emit RefundIssued(projectId, msg.sender, investmentAmount);
    }
    
    // 取消项目（创建者）
    function cancelProject(uint256 projectId) external {
        Project storage project = projects[projectId];
        require(msg.sender == project.creator, "Not creator");
        require(project.state == ProjectState.Fundraising, "Cannot cancel");
        
        project.state = ProjectState.Cancelled;
        emit ProjectStateChanged(projectId, ProjectState.Cancelled);
    }
    
    // 查询函数
    function getProject(uint256 projectId) external view returns (
        uint256 id,
        address creator,
        string memory title,
        string memory description,
        uint256 goal,
        uint256 raised,
        uint256 deadline,
        ProjectState state
    ) {
        Project storage project = projects[projectId];
        return (
            project.id,
            project.creator,
            project.title,
            project.description,
            project.goal,
            project.raised,
            project.deadline,
            project.state
        );
    }
    
    function getMilestone(uint256 projectId, uint256 milestoneIndex) 
        external 
        view 
        returns (
            string memory description,
            uint256 amount,
            uint256 deadline,
            bool approved,
            bool completed,
            uint256 approvalVotes
        ) 
    {
        Milestone storage milestone = projects[projectId].milestones[milestoneIndex];
        return (
            milestone.description,
            milestone.amount,
            milestone.deadline,
            milestone.approved,
            milestone.completed,
            milestone.approvalVotes
        );
    }
    
    function getInvestments(uint256 projectId) 
        external 
        view 
        returns (Investment[] memory) 
    {
        return projectInvestments[projectId];
    }
    
    function getInvestorAmount(uint256 projectId, address investor) 
        external 
        view 
        returns (uint256) 
    {
        return investorAmounts[projectId][investor];
    }
}
```

---

### Part 3: 测试脚本编写 (2小时)

#### 3.1 测试文件

创建 `test/Crowdfunding.test.js`:

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { time } = require("@nomicfoundation/hardhat-network-helpers");

describe("Crowdfunding Platform", function () {
    let rewardToken, badgeNFT, platform;
    let owner, creator, investor1, investor2;
    
    const GOAL = ethers.parseEther("10");
    const DURATION = 7 * 24 * 60 * 60; // 7天
    
    beforeEach(async function () {
        [owner, creator, investor1, investor2] = await ethers.getSigners();
        
        // 部署奖励代币
        const RewardToken = await ethers.getContractFactory("RewardToken");
        rewardToken = await RewardToken.deploy();
        
        // 部署徽章NFT
        const BadgeNFT = await ethers.getContractFactory("BadgeNFT");
        badgeNFT = await BadgeNFT.deploy();
        
        // 部署众筹平台
        const Platform = await ethers.getContractFactory("CrowdfundingPlatform");
        platform = await Platform.deploy(
            await rewardToken.getAddress(),
            await badgeNFT.getAddress()
        );
        
        // 设置权限
        await rewardToken.transferOwnership(await platform.getAddress());
        await badgeNFT.transferOwnership(await platform.getAddress());
    });
    
    describe("项目创建", function () {
        it("应该成功创建项目", async function () {
            const milestones = {
                descriptions: ["开发阶段", "测试阶段", "上线阶段"],
                amounts: [
                    ethers.parseEther("3"),
                    ethers.parseEther("3"),
                    ethers.parseEther("4")
                ],
                deadlines: [
                    Math.floor(Date.now() / 1000) + 30 * 24 * 60 * 60,
                    Math.floor(Date.now() / 1000) + 60 * 24 * 60 * 60,
                    Math.floor(Date.now() / 1000) + 90 * 24 * 60 * 60
                ]
            };
            
            await expect(
                platform.connect(creator).createProject(
                    "去中心化社交平台",
                    "基于区块链的社交网络",
                    GOAL,
                    DURATION,
                    milestones.descriptions,
                    milestones.amounts,
                    milestones.deadlines
                )
            ).to.emit(platform, "ProjectCreated");
            
            const project = await platform.getProject(0);
            expect(project.creator).to.equal(creator.address);
            expect(project.goal).to.equal(GOAL);
        });
        
        it("里程碑总额应该等于目标金额", async function () {
            const milestones = {
                descriptions: ["阶段1", "阶段2"],
                amounts: [
                    ethers.parseEther("3"),
                    ethers.parseEther("5")  // 总计8 ETH，不等于10 ETH
                ],
                deadlines: [
                    Math.floor(Date.now() / 1000) + 30 * 24 * 60 * 60,
                    Math.floor(Date.now() / 1000) + 60 * 24 * 60 * 60
                ]
            };
            
            await expect(
                platform.connect(creator).createProject(
                    "项目",
                    "描述",
                    GOAL,
                    DURATION,
                    milestones.descriptions,
                    milestones.amounts,
                    milestones.deadlines
                )
            ).to.be.revertedWith("Total milestone amount must equal goal");
        });
    });
    
    describe("投资功能", function () {
        let projectId;
        
        beforeEach(async function () {
            const milestones = {
                descriptions: ["里程碑1"],
                amounts: [GOAL],
                deadlines: [Math.floor(Date.now() / 1000) + 30 * 24 * 60 * 60]
            };
            
            await platform.connect(creator).createProject(
                "测试项目",
                "描述",
                GOAL,
                DURATION,
                milestones.descriptions,
                milestones.amounts,
                milestones.deadlines
            );
            projectId = 0;
        });
        
        it("应该成功投资", async function () {
            const investAmount = ethers.parseEther("5");
            
            await expect(
                platform.connect(investor1).invest(projectId, { value: investAmount })
            ).to.emit(platform, "InvestmentMade");
            
            const project = await platform.getProject(projectId);
            expect(project.raised).to.equal(investAmount);
            
            const investorAmount = await platform.getInvestorAmount(projectId, investor1.address);
            expect(investorAmount).to.equal(investAmount);
        });
        
        it("应该铸造奖励代币", async function () {
            const investAmount = ethers.parseEther("1");
            await platform.connect(investor1).invest(projectId, { value: investAmount });
            
            const rewardBalance = await rewardToken.balanceOf(investor1.address);
            expect(rewardBalance).to.equal(investAmount * 100n);
        });
        
        it("应该铸造徽章NFT", async function () {
            const investAmount = ethers.parseEther("1");
            await platform.connect(investor1).invest(projectId, { value: investAmount });
            
            const balance = await badgeNFT.balanceOf(investor1.address);
            expect(balance).to.equal(1);
        });
        
        it("达到目标后状态应该变为Successful", async function () {
            await platform.connect(investor1).invest(projectId, { value: GOAL });
            
            const project = await platform.getProject(projectId);
            expect(project.state).to.equal(2); // Successful
        });
    });
    
    describe("里程碑投票", function () {
        let projectId;
        
        beforeEach(async function () {
            const milestones = {
                descriptions: ["里程碑1"],
                amounts: [GOAL],
                deadlines: [Math.floor(Date.now() / 1000) + 30 * 24 * 60 * 60]
            };
            
            await platform.connect(creator).createProject(
                "测试项目",
                "描述",
                GOAL,
                DURATION,
                milestones.descriptions,
                milestones.amounts,
                milestones.deadlines
            );
            projectId = 0;
            
            // 投资达到目标
            await platform.connect(investor1).invest(projectId, { 
                value: ethers.parseEther("6") 
            });
            await platform.connect(investor2).invest(projectId, { 
                value: ethers.parseEther("4") 
            });
        });
        
        it("投资者应该能投票", async function () {
            await expect(
                platform.connect(investor1).approveMilestone(projectId, 0)
            ).to.emit(platform, "MilestoneApproved");
        });
        
        it("超过50%投资额应该批准里程碑", async function () {
            await platform.connect(investor1).approveMilestone(projectId, 0);
            
            const milestone = await platform.getMilestone(projectId, 0);
            expect(milestone.approved).to.be.true;
        });
        
        it("非投资者不能投票", async function () {
            const [, , , , nonInvestor] = await ethers.getSigners();
            
            await expect(
                platform.connect(nonInvestor).approveMilestone(projectId, 0)
            ).to.be.revertedWith("Not an investor");
        });
    });
    
    describe("资金提取", function () {
        let projectId;
        
        beforeEach(async function () {
            const milestones = {
                descriptions: ["里程碑1"],
                amounts: [GOAL],
                deadlines: [Math.floor(Date.now() / 1000) + 30 * 24 * 60 * 60]
            };
            
            await platform.connect(creator).createProject(
                "测试项目",
                "描述",
                GOAL,
                DURATION,
                milestones.descriptions,
                milestones.amounts,
                milestones.deadlines
            );
            projectId = 0;
            
            // 投资并批准
            await platform.connect(investor1).invest(projectId, { value: GOAL });
            await platform.connect(investor1).approveMilestone(projectId, 0);
        });
        
        it("创建者应该能提取批准的里程碑资金", async function () {
            const creatorBalanceBefore = await ethers.provider.getBalance(creator.address);
            
            const tx = await platform.connect(creator).withdrawMilestoneFunds(projectId, 0);
            const receipt = await tx.wait();
            const gasUsed = receipt.gasUsed * receipt.gasPrice;
            
            const creatorBalanceAfter = await ethers.provider.getBalance(creator.address);
            
            // 计算扣除手续费后的金额
            const fee = (GOAL * 300n) / 10000n;
            const expectedAmount = GOAL - fee;
            
            expect(creatorBalanceAfter).to.equal(
                creatorBalanceBefore + expectedAmount - gasUsed
            );
        });
        
        it("未批准的里程碑不能提取", async function () {
            const milestones = {
                descriptions: ["里程碑1", "里程碑2"],
                amounts: [ethers.parseEther("5"), ethers.parseEther("5")],
                deadlines: [
                    Math.floor(Date.now() / 1000) + 30 * 24 * 60 * 60,
                    Math.floor(Date.now() / 1000) + 60 * 24 * 60 * 60
                ]
            };
            
            await platform.connect(creator).createProject(
                "测试项目2",
                "描述",
                GOAL,
                DURATION,
                milestones.descriptions,
                milestones.amounts,
                milestones.deadlines
            );
            
            await platform.connect(investor1).invest(1, { value: GOAL });
            
            await expect(
                platform.connect(creator).withdrawMilestoneFunds(1, 0)
            ).to.be.revertedWith("Milestone not approved");
        });
    });
    
    describe("退款机制", function () {
        let projectId;
        
        beforeEach(async function () {
            const milestones = {
                descriptions: ["里程碑1"],
                amounts: [GOAL],
                deadlines: [Math.floor(Date.now() / 1000) + 30 * 24 * 60 * 60]
            };
            
            await platform.connect(creator).createProject(
                "测试项目",
                "描述",
                GOAL,
                DURATION,
                milestones.descriptions,
                milestones.amounts,
                milestones.deadlines
            );
            projectId = 0;
        });
        
        it("项目失败后投资者应该能退款", async function () {
            const investAmount = ethers.parseEther("5");
            await platform.connect(investor1).invest(projectId, { value: investAmount });
            
            // 时间快进到截止日期后
            await time.increase(DURATION + 1);
            
            // 检查状态
            await platform.checkProjectStatus(projectId);
            
            const balanceBefore = await ethers.provider.getBalance(investor1.address);
            
            const tx = await platform.connect(investor1).refund(projectId);
            const receipt = await tx.wait();
            const gasUsed = receipt.gasUsed * receipt.gasPrice;
            
            const balanceAfter = await ethers.provider.getBalance(investor1.address);
            
            expect(balanceAfter).to.equal(balanceBefore + investAmount - gasUsed);
        });
    });
});
```

---

## 🌐 Day 7: 前端开发 (6-7小时)

### Part 1: 前端项目初始化 (0.5小时)

```bash
# 创建前端目录
mkdir frontend
cd frontend

# 初始化React项目
npx create-react-app .

# 安装依赖
npm install ethers@6 @rainbow-me/rainbowkit wagmi viem
```

### Part 2: 前端核心组件 (3小时)

#### 2.1 项目列表组件

创建 `src/components/ProjectList.js`:

```javascript
import React, { useState, useEffect } from 'react';
import { useContract, useSigner } from 'wagmi';
import { ethers } from 'ethers';

const PLATFORM_ADDRESS = "YOUR_CONTRACT_ADDRESS";
const PLATFORM_ABI = [/* ABI */];

function ProjectList() {
    const [projects, setProjects] = useState([]);
    const { data: signer } = useSigner();
    const contract = useContract({
        address: PLATFORM_ADDRESS,
        abi: PLATFORM_ABI,
        signerOrProvider: signer
    });
    
    useEffect(() => {
        loadProjects();
    }, [contract]);
    
    const loadProjects = async () => {
        if (!contract) return;
        
        const projectCount = 10; // 假设有10个项目
        const projectsData = [];
        
        for (let i = 0; i < projectCount; i++) {
            try {
                const project = await contract.getProject(i);
                projectsData.push({
                    id: i,
                    ...project
                });
            } catch (error) {
                break;
            }
        }
        
        setProjects(projectsData);
    };
    
    const getStateText = (state) => {
        const states = [
            '募资中', '失败', '成功', '执行中', '已完成', '已取消'
        ];
        return states[state] || '未知';
    };
    
    return (
        <div className="project-list">
            <h2>众筹项目列表</h2>
            <div className="projects-grid">
                {projects.map(project => (
                    <div key={project.id} className="project-card">
                        <h3>{project.title}</h3>
                        <p>{project.description}</p>
                        <div className="project-stats">
                            <div>
                                <span>目标：</span>
                                {ethers.formatEther(project.goal)} ETH
                            </div>
                            <div>
                                <span>已筹集：</span>
                                {ethers.formatEther(project.raised)} ETH
                            </div>
                            <div>
                                <span>进度：</span>
                                {((project.raised * 100n) / project.goal).toString()}%
                            </div>
                            <div>
                                <span>状态：</span>
                                {getStateText(project.state)}
                            </div>
                        </div>
                        <button onClick={() => window.location.href = `/project/${project.id}`}>
                            查看详情
                        </button>
                    </div>
                ))}
            </div>
        </div>
    );
}

export default ProjectList;
```

#### 2.2 项目详情组件

创建 `src/components/ProjectDetail.js`:

```javascript
import React, { useState, useEffect } from 'react';
import { useParams } from 'react-router-dom';
import { useContract, useSigner } from 'wagmi';
import { ethers } from 'ethers';

function ProjectDetail() {
    const { id } = useParams();
    const [project, setProject] = useState(null);
    const [milestones, setMilestones] = useState([]);
    const [investAmount, setInvestAmount] = useState('');
    const { data: signer } = useSigner();
    
    const contract = useContract({
        address: PLATFORM_ADDRESS,
        abi: PLATFORM_ABI,
        signerOrProvider: signer
    });
    
    useEffect(() => {
        loadProjectData();
    }, [id, contract]);
    
    const loadProjectData = async () => {
        if (!contract) return;
        
        const projectData = await contract.getProject(id);
        setProject(projectData);
        
        // 加载里程碑
        const milestonesData = [];
        for (let i = 0; i < 10; i++) {
            try {
                const milestone = await contract.getMilestone(id, i);
                milestonesData.push(milestone);
            } catch (error) {
                break;
            }
        }
        setMilestones(milestonesData);
    };
    
    const handleInvest = async () => {
        if (!investAmount || !contract) return;
        
        try {
            const tx = await contract.invest(id, {
                value: ethers.parseEther(investAmount)
            });
            await tx.wait();
            alert('投资成功！');
            loadProjectData();
        } catch (error) {
            console.error(error);
            alert('投资失败：' + error.message);
        }
    };
    
    const handleApproveMilestone = async (milestoneIndex) => {
        try {
            const tx = await contract.approveMilestone(id, milestoneIndex);
            await tx.wait();
            alert('投票成功！');
            loadProjectData();
        } catch (error) {
            console.error(error);
            alert('投票失败：' + error.message);
        }
    };
    
    if (!project) return <div>加载中...</div>;
    
    return (
        <div className="project-detail">
            <div className="project-header">
                <h1>{project.title}</h1>
                <p>{project.description}</p>
            </div>
            
            <div className="project-funding">
                <h2>募资情况</h2>
                <div className="funding-stats">
                    <div>目标：{ethers.formatEther(project.goal)} ETH</div>
                    <div>已筹集：{ethers.formatEther(project.raised)} ETH</div>
                    <div>创建者：{project.creator}</div>
                </div>
                
                {project.state === 0 && (
                    <div className="invest-form">
                        <input
                            type="number"
                            step="0.01"
                            placeholder="投资金额(ETH)"
                            value={investAmount}
                            onChange={(e) => setInvestAmount(e.target.value)}
                        />
                        <button onClick={handleInvest}>投资</button>
                    </div>
                )}
            </div>
            
            <div className="milestones">
                <h2>项目里程碑</h2>
                {milestones.map((milestone, index) => (
                    <div key={index} className="milestone-card">
                        <h3>里程碑 {index + 1}</h3>
                        <p>{milestone.description}</p>
                        <div>金额：{ethers.formatEther(milestone.amount)} ETH</div>
                        <div>
                            状态：
                            {milestone.completed ? '已完成' : 
                             milestone.approved ? '已批准' : '待批准'}
                        </div>
                        {!milestone.approved && !milestone.completed && (
                            <button onClick={() => handleApproveMilestone(index)}>
                                投票批准
                            </button>
                        )}
                    </div>
                ))}
            </div>
        </div>
    );
}

export default ProjectDetail;
```

### Part 3: 样式和部署 (2.5小时)

#### 3.1 添加样式

创建 `src/App.css`:

```css
.project-list {
    padding: 20px;
}

.projects-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 20px;
    margin-top: 20px;
}

.project-card {
    border: 1px solid #ddd;
    border-radius: 8px;
    padding: 20px;
    background: white;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.project-card h3 {
    margin-top: 0;
    color: #333;
}

.project-stats {
    margin: 15px 0;
}

.project-stats > div {
    margin: 5px 0;
    font-size: 14px;
}

.project-stats span {
    font-weight: bold;
}

button {
    background: #007bff;
    color: white;
    border: none;
    padding: 10px 20px;
    border-radius: 4px;
    cursor: pointer;
}

button:hover {
    background: #0056b3;
}

.invest-form {
    margin: 20px 0;
    display: flex;
    gap: 10px;
}

.invest-form input {
    flex: 1;
    padding: 10px;
    border: 1px solid #ddd;
    border-radius: 4px;
}

.milestone-card {
    border: 1px solid #ddd;
    padding: 15px;
    margin: 10px 0;
    border-radius: 4px;
}
```

---

## 📝 今日作业

### 作业1: 功能扩展

为众筹平台添加以下功能：
1. 项目更新功能（创建者发布进度更新）
2. 评论系统
3. 项目收藏/关注
4. 推荐算法
5. 搜索和过滤

### 作业2: 优化和安全

1. 添加紧急暂停功能
2. 实现时间锁机制
3. 添加多签管理
4. Gas优化
5. 安全审计

### 作业3: 高级特性

1. 支持ERC20代币投资（不仅ETH）
2. 自动化里程碑验证（预言机）
3. DAO治理
4. 跨链支持
5. IPFS存储项目资料

---

## ✅ 项目检查清单

- [ ] 所有合约通过测试
- [ ] 前端正常运行
- [ ] 完成基础功能
- [ ] 代码有注释
- [ ] 写了README文档
- [ ] 部署到测试网

---

## 🆘 常见问题FAQ

### Q1: 如何防止项目方跑路？

A: 使用里程碑+投票机制，资金分批释放，投资者可以投票决定是否继续。

### Q2: 如何确保投票的公平性？

A: 按投资金额加权投票，防止女巫攻击。

### Q3: 失败项目的退款如何保证？

A: 使用pull payment模式，资金锁定在合约中，失败后投资者主动提取。

---

## 📚 扩展资源

- [Kickstarter](https://www.kickstarter.com/) - 传统众筹平台
- [Gitcoin](https://gitcoin.co/) - Web3众筹
- [Juicebox](https://juicebox.money/) - DAO众筹协议

---

## 🎉 项目完成

恭喜完成Week 2的综合项目！

下周预告：
- Week 3: Solidity深入与安全
- 高级Solidity特性
- 智能合约安全
- DeFi协议学习

**继续加油！🚀**
