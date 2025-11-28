# Week 10 Day 6-7: 综合 DevOps 与可升级合约项目

**预计学习时间**: 10-12小时  
**难度**: ⭐⭐⭐⭐⭐

## 项目概述

本周末项目将整合第10周所学的所有 DevOps 和合约运维知识。你将从零构建一个企业级的 Web3 项目脚手架，包含可升级的治理代币和质押合约，并为其配置完整的自动化测试、部署流水线和监控告警。

### 项目目标

1.  **开发可升级合约**: 实现 UUPS 模式的 ERC20 治理代币和可升级的质押池。
2.  **构建 CI/CD**: 配置 GitHub Actions 实现自动编译、测试和部署。
3.  **多环境管理**: 配置 Hardhat 支持 Local, Sepolia, Mainnet 多环境。
4.  **质量保证**: 集成 Slither 静态分析和 Solhint 代码风格检查。
5.  **自动化运维**: 编写脚本实现自动验证、升级和紧急暂停。

## 项目需求

### 1. 智能合约架构

-   **GovernanceToken (UUPS)**:
    -   基于 ERC20Upgradeable。
    -   支持快照 (ERC20Snapshot) 和投票 (ERC20Votes)。
    -   拥有者必须是多签钱包 (部署时先设为 Deployer，后移交)。
-   **StakingPool (UUPS)**:
    -   用户质押 GovernanceToken，赚取奖励。
    -   **V1 功能**: 基本的质押 (Stake) 和提款 (Withdraw)。
    -   **V2 功能 (模拟升级)**: 增加锁定期 (Lock Period) 和动态调整奖励率。
    -   集成 `Pausable`，允许管理员在紧急情况下暂停质押和提款。

### 2. DevOps 基础设施

-   **Linting**: Solhint + Prettier。
-   **Testing**: Hardhat Waffle (Unit Tests) + Forking Tests (Integration)。
-   **CI**: Push 代码时自动运行 Lint, Test, Gas Report。
-   **CD**: Release 发布时自动部署到测试网，并验证源码。
-   **Monitoring**: 集成 Tenderly 监控大额转账。

## 实施步骤

### 第一阶段: 初始化与配置 (2小时)

1.  **项目初始化**:
    ```bash
    mkdir devops-project
    cd devops-project
    npm init -y
    npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox @openzeppelin/hardhat-upgrades hardhat-deploy dotenv
    npm install @openzeppelin/contracts-upgradeable
    npx hardhat init # 选择 TypeScript 或 JavaScript
    ```

2.  **配置 Hardhat**:
    -   设置多网络 (localhost, sepolia)。
    -   配置 Etherscan API Key。
    -   配置 Gas Reporter。

3.  **设置 Linting**:
    -   安装 `solhint`, `prettier`, `prettier-plugin-solidity`。
    -   创建 `.solhint.json` 和 `.prettierrc`。

### 第二阶段: 开发可升级合约 (3小时)

1.  **编写 GovernanceToken.sol**:
    ```solidity
    // SPDX-License-Identifier: MIT
    pragma solidity ^0.8.9;

    import "@openzeppelin/contracts-upgradeable/token/ERC20/extensions/ERC20VotesUpgradeable.sol";
    import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
    import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
    import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

    contract GovernanceToken is Initializable, ERC20VotesUpgradeable, OwnableUpgradeable, UUPSUpgradeable {
        function initialize() public initializer {
            __ERC20_init("DevOps DAO", "DOD");
            __ERC20Permit_init("DevOps DAO");
            __Ownable_init();
            __UUPSUpgradeable_init();
            
            _mint(msg.sender, 1000000 * 10 ** decimals());
        }

        function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}

        //以此类推实现 required overrides
    }
    ```

2.  **编写 StakingPoolV1.sol**:
    -   使用 `Initializable`, `PausableUpgradeable`, `UUPSUpgradeable`。
    -   实现 `stake(uint256 amount)` 和 `withdraw(uint256 amount)`。

3.  **编写单元测试**:
    -   测试 `initialize` 是否只能调用一次。
    -   测试 Proxy 模式下的状态变更。
    -   测试暂停功能。

### 第三阶段: 模拟升级流程 (2小时)

1.  **编写 StakingPoolV2.sol**:
    -   继承 V1。
    -   新增 `uint256 public lockTime;`。
    -   重写 `withdraw`，增加时间检查。

2.  **编写升级测试脚本**:
    ```javascript
    const { ethers, upgrades } = require("hardhat");

    it("Should upgrade to V2 and preserve state", async function () {
        const StakingPoolV1 = await ethers.getContractFactory("StakingPoolV1");
        const StakingPoolV2 = await ethers.getContractFactory("StakingPoolV2");

        const instance = await upgrades.deployProxy(StakingPoolV1, [tokenAddress]);
        await instance.waitForDeployment();
        
        // 用户质押
        await instance.connect(user).stake(100);

        // 升级
        const upgraded = await upgrades.upgradeProxy(await instance.getAddress(), StakingPoolV2);
        
        // 验证状态保留
        expect(await upgraded.balanceOf(user.address)).to.equal(100);
        
        // 验证新功能
        await upgraded.setLockTime(86400);
        expect(await upgraded.lockTime()).to.equal(86400);
    });
    ```

### 第四阶段: CI/CD 流水线搭建 (2小时)

1.  **创建 `.github/workflows/test.yml`**:
    -   触发条件: Push to main / Pull Request。
    -   步骤: Checkout -> Install Node -> Install Deps -> Lint -> Compile -> Test。

2.  **创建 `.github/workflows/deploy-sepolia.yml`**:
    -   触发条件: Release published。
    -   步骤:
        -   读取 `SEPOLIA_RPC_URL` 和 `PRIVATE_KEY` (从 GitHub Secrets)。
        -   运行 `npx hardhat run scripts/deploy.js --network sepolia`。
        -   运行 `npx hardhat verify ...`。

### 第五阶段: 运维与监控 (1小时)

1.  **编写运维脚本**:
    -   `scripts/pause.js`: 紧急暂停合约。
    -   `scripts/transfer-admin.js`: 将 ProxyAdmin 移交给多签地址。

2.  **配置 Tenderly (可选)**:
    -   将部署后的 Sepolia 合约添加到 Tenderly Dashboard。
    -   模拟一笔失败的交易，查看 Debug 信息。

## 交付物清单

1.  **GitHub 仓库**:
    -   完整的 Hardhat 项目结构。
    -   覆盖率 > 90% 的测试报告。
    -   配置好的 GitHub Actions workflows。
2.  **部署记录**:
    -   Sepolia 网络上的 Token Proxy 地址。
    -   Sepolia 网络上的 StakingPool Proxy 地址。
    -   Etherscan 上的已验证源码链接。
3.  **操作文档 (`OPS.md`)**:
    -   如何部署 V1。
    -   如何升级到 V2。
    -   如何执行紧急暂停。
    -   多签钱包地址 (如果是模拟的)。

## 常见问题 FAQ

**Q: 升级时报错 "New storage layout is incompatible" 怎么办？**
A: 检查 V2 合约。你是否改变了 V1 中变量的顺序或类型？是否在中间插入了新变量？新变量必须添加在最后，或者使用 Storage Gap。

**Q: 验证合约时 Etherscan 提示 "Fail - Unable to verify"？**
A: 确保本地编译的 settings (optimizer runs) 与 Hardhat config 中完全一致。对于 Proxy 合约，通常只需验证 Implementation 合约，Proxy 合约 Etherscan 会自动识别（或者手动在该页面点击 "Is this a proxy?"）。

**Q: 为什么测试通过但 CI 失败？**
A: 检查 GitHub Secrets 是否正确配置了 `RPC_URL` 或 `API_KEY`。如果是 Gas Reporter 导致的失败，检查是否配置了 `COINMARKETCAP_API_KEY` 或者暂时在 CI 中禁用它。

## 总结

完成这个项目，标志着你已经具备了 Web3 领域高级 DevOps 工程师的能力。你不仅能写合约，还能构建一套安全、可靠、自动化的软件交付生命周期。这是区分"写代码的人"和"工程化团队"的关键分水岭。

下一周，我们将进入 Web3 安全审计的深水区，学习如何像黑客一样思考，发现并修复合约漏洞。
