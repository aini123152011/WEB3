# Week 10 Day 5: 合约升级与维护

**预计学习时间**: 6-8小时  
**难度**: ⭐⭐⭐⭐⭐

## 学习目标

通过本节课程,你将能够:
- 深入理解三种主流的代理升级模式 (Transparent, UUPS, Beacon)
- 使用 Hardhat Upgrades 插件编写和部署可升级合约
- 实现合约的暂停 (Pausable) 和紧急提款机制
- 使用 Gnosis Safe 多签钱包管理合约升级权限
- 掌握存储布局 (Storage Layout) 原理及冲突避免策略

## 课程内容

### 1. 代理合约模式详解

#### 1.1 为什么合约需要升级

区块链的不可篡改性是双刃剑。一旦部署，代码无法修改。但现实中：
- 发现严重 Bug
- 业务逻辑变更
- 优化 Gas 消耗

#### 1.2 代理模式原理

用户 -> **Proxy (存储状态)** -> `delegatecall` -> **Implementation (逻辑代码)**

- **Proxy**: 只有 fallback 函数，负责转发调用，存储数据。
- **Implementation**: 包含实际业务逻辑，不存储数据（数据存在 Proxy 中）。

#### 1.3 主流模式对比

| 模式 | Transparent Proxy | UUPS (Universal Upgradeable Proxy Standard) | Beacon Proxy |
| :--- | :--- | :--- | :--- |
| **升级逻辑位置** | 在 Proxy 合约中 (ProxyAdmin) | 在 Implementation 合约中 | 在 Beacon 合约中 |
| **Gas 成本** | 部署较高，调用略高 (每次检查 Admin) | 部署更低，调用更低 | 适用于批量部署相同逻辑 |
| **安全性** | 防止函数选择器冲突 (Clashing) | 如果新逻辑忘记包含升级函数，会导致合约永久无法升级 | 统一管理多个 Proxy 的版本 |
| **推荐场景** | 传统项目，无需关心 Implementation 大小 | 推荐标准，更省 Gas | 需要生成大量相同合约 (如钱包工厂) |

### 2. 使用 Hardhat Upgrades 插件

#### 2.1 安装与配置

```bash
npm install --save-dev @openzeppelin/hardhat-upgrades
npm install @openzeppelin/contracts-upgradeable
```

```javascript
// hardhat.config.js
require('@openzeppelin/hardhat-upgrades');
```

#### 2.2 编写可升级合约 (Initializable)

**注意**: 可升级合约不能有 `constructor`，必须使用 `initialize` 函数。

```solidity
// contracts/MyTokenV1.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract MyTokenV1 is Initializable, ERC20Upgradeable, OwnableUpgradeable, UUPSUpgradeable {
    // 1. 使用 initialize 代替 constructor
    function initialize() public initializer {
        __ERC20_init("MyToken", "MTK");
        __Ownable_init();
        __UUPSUpgradeable_init();
        
        _mint(msg.sender, 1000 * 10 ** decimals());
    }

    // 2. 必须实现此函数以授权升级 (UUPS)
    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
}
```

#### 2.3 部署 V1 版本

```javascript
// scripts/deploy-v1.js
const { ethers, upgrades } = require("hardhat");

async function main() {
  const MyTokenV1 = await ethers.getContractFactory("MyTokenV1");
  
  console.log("Deploying MyTokenV1...");
  // kind: 'uups' 指定使用 UUPS 模式，默认为 transparent
  const myToken = await upgrades.deployProxy(MyTokenV1, [], { kind: 'uups' });
  
  await myToken.deployed();
  
  console.log("MyTokenV1 deployed to:", myToken.address); // 这是 Proxy 地址
  console.log("Implementation address:", await upgrades.erc1967.getImplementationAddress(myToken.address));
}

main();
```

#### 2.4 编写 V2 版本

**注意**: 
- 不能修改现有状态变量的类型和顺序。
- 新变量必须添加在最后。
- 不能移除变量。

```solidity
// contracts/MyTokenV2.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./MyTokenV1.sol";

contract MyTokenV2 is MyTokenV1 {
    // 新增状态变量
    uint256 public fee;

    // 新增功能
    function setFee(uint256 _fee) public onlyOwner {
        fee = _fee;
    }

    // 修改现有逻辑 (例如 transfer)
    function transfer(address to, uint256 amount) public virtual override returns (bool) {
        // 简单的扣费逻辑示例
        if (fee > 0) {
            uint256 feeAmount = (amount * fee) / 10000;
            super.transfer(owner(), feeAmount);
            amount -= feeAmount;
        }
        return super.transfer(to, amount);
    }
    
    // V2 初始化函数 (如果需要) - 使用 reinitializer(2)
    function initializeV2(uint256 _fee) public reinitializer(2) {
        fee = _fee;
    }
}
```

#### 2.5 升级到 V2

```javascript
// scripts/upgrade-v2.js
const { ethers, upgrades } = require("hardhat");

async function main() {
  const proxyAddress = "YOUR_PROXY_ADDRESS"; // V1 部署后的 Proxy 地址
  
  const MyTokenV2 = await ethers.getContractFactory("MyTokenV2");
  
  console.log("Upgrading to MyTokenV2...");
  // 如果有新的初始化逻辑，可以使用 call 选项
  const myTokenV2 = await upgrades.upgradeProxy(proxyAddress, MyTokenV2, {
    call: { fn: 'initializeV2', args: [100] } // 设置费率为 1%
  });
  
  console.log("MyToken upgraded successfully");
  console.log("New Implementation address:", await upgrades.erc1967.getImplementationAddress(myTokenV2.address));
}
```

### 3. 存储冲突 (Storage Collision)

#### 3.1 原理

Proxy 使用 `delegatecall`，数据存储在 Proxy 的 Storage Slot 中。如果 V1 定义了 `uint a` (Slot 0)，而 V2 修改为 `address b` (Slot 0)，会导致数据错乱。

#### 3.2 存储间隙 (Storage Gaps)

为了给未来父合约升级预留空间，通常在合约末尾添加 `__gap`。

```solidity
contract MyBaseContract is Initializable {
    uint256 public x;
    
    // 预留 49 个 slot (默认 OpenZeppelin 合约会预留 50 个)
    uint256[49] private __gap;
}
```

### 4. 运维与治理

#### 4.1 暂停与紧急提款

集成 `PausableUpgradeable`。

```solidity
import "@openzeppelin/contracts-upgradeable/security/PausableUpgradeable.sol";

contract MyDefi is Initializable, PausableUpgradeable, OwnableUpgradeable {
    function initialize() public initializer {
        __Pausable_init();
        __Ownable_init();
    }

    function pause() public onlyOwner {
        _pause();
    }

    function unpause() public onlyOwner {
        _unpause();
    }

    function deposit() public payable whenNotPaused {
        // ...
    }
    
    // 紧急提款：即使暂停也允许管理员提取资金（慎用，需多签）
    function emergencyWithdraw() public onlyOwner {
        payable(owner()).transfer(address(this).balance);
    }
}
```

#### 4.2 移交权限给多签钱包 (Gnosis Safe)

在生产环境，`owner` 和 `ProxyAdmin` 绝不能是单签 EOA。

```javascript
// scripts/transfer-ownership.js
const { upgrades } = require("hardhat");

async function main() {
  const gnosisSafeAddress = "0x...";
  
  console.log("Transferring ownership of ProxyAdmin...");
  // 转移 ProxyAdmin 的所有权 (仅限 Transparent 模式)
  await upgrades.admin.transferProxyAdminOwnership(gnosisSafeAddress);
  
  // 转移合约自身的 Ownable 权限
  const myToken = await ethers.getContractAt("MyTokenV1", "PROXY_ADDRESS");
  await myToken.transferOwnership(gnosisSafeAddress);
  
  console.log("Ownership transferred to Gnosis Safe");
}
```

#### 4.3 通过 OpenZeppelin Defender 进行升级

1.  在 Defender Admin 中添加 Gnosis Safe 合约。
2.  部署新版本的 Implementation 合约。
3.  在 Defender 中发起 "Upgrade" 提案。
4.  多签持有者在 Gnosis Safe 中签名批准。

### 5. 数据迁移策略

如果新逻辑需要的数据结构与旧版本完全不同，可能需要数据迁移。

1.  **链上迁移**: 在 `initializeV2` 中遍历处理 (Gas 昂贵，可能 OOG)。
2.  **懒惰迁移**: 在用户下次交互时进行数据转换。
3.  **快照迁移**: 部署全新合约，根据快照空投/铸造新代币 (最彻底，但改变了合约地址)。

## 实践练习

### 练习1: 编写并升级 UUPS 合约

1.  创建一个 `Box` 合约，存储一个 `uint256 val`。
2.  部署 V1 版本。
3.  编写 V2 版本，增加 `increment()` 函数，**并且将 val 存储变量的位置故意写错 (引起冲突)**，尝试运行 `upgradeProxy`，观察 OpenZeppelin 插件的报错。
4.  修正 V2，成功升级。

### 练习2: 集成多签升级

1.  在 Goerli 测试网部署一个 Transparent Proxy 合约。
2.  创建一个 Gnosis Safe (可以使用网页版 UI)。
3.  将 `ProxyAdmin` 的所有权转移给 Gnosis Safe。
4.  尝试直接用原来的 Deployer 账号升级合约 (应该失败)。
5.  通过 Gnosis Safe 提案执行升级。

### 练习3: 紧急暂停开关

1.  给你的 DeFi 合约添加 `Pausable` 功能。
2.  编写测试脚本：
    - 管理员调用 `pause()`。
    - 普通用户尝试 `deposit()`，断言 revert `Pausable: paused`。
    - 管理员调用 `unpause()`。
    - 用户再次 `deposit()` 成功。

## 学习检查清单

- [ ] 清楚区分 Proxy 和 Implementation 的职责
- [ ] 理解 Delegatecall 的上下文保留机制
- [ ] 能够分辨 Transparent, UUPS, Beacon 的区别
- [ ] 熟练使用 Hardhat Upgrades 部署和升级
- [ ] 掌握 Storage Gaps 的作用
- [ ] 能够集成 Pausable 模式
- [ ] 理解为何生产环境必须使用多签管理升级权限

## 扩展阅读

- [OpenZeppelin Upgrades Plugins Docs](https://docs.openzeppelin.com/upgrades-plugins/1.x/)
- [Writing Upgradeable Contracts](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable)
- [Proxy Patterns - The State of Art](https://blog.openzeppelin.com/proxy-patterns/)
- [UUPS vs Transparent Proxy](https://docs.openzeppelin.com/contracts/4.x/api/proxy#transparent-vs-uups)

## 下节预告

下节是 **Week 10 Weekend Project: 综合 DevOps 与可升级合约项目**。
我们将整合本周所学，从零搭建一个具备自动化测试、CI/CD 流水线、多环境配置、且包含可升级治理代币和质押合约的完整工程。
