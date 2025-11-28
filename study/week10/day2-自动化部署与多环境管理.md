# Week 10 Day 2: 自动化部署与多环境管理

**预计学习时间**: 6-8小时  
**难度**: ⭐⭐⭐⭐

## 学习目标

通过本节课程,你将能够:
- 理解多环境部署策略和最佳实践
- 实现智能合约的自动化部署流程
- 管理不同环境的配置和密钥
- 实现部署验证和回滚机制
- 掌握蓝绿部署和金丝雀发布策略

## 课程内容

### 1. 多环境管理策略

#### 1.1 环境分类

**标准环境划分**:
```
开发环境 (Development)
├── 本地开发网络 (Hardhat/Ganache)
├── 测试网络 (Sepolia/Goerli)
└── 特点: 快速迭代,频繁部署

测试环境 (Testing/Staging)
├── 公共测试网 (Sepolia/Mumbai)
├── 私有测试网
└── 特点: 接近生产,完整测试

预生产环境 (Pre-production)
├── 主网分叉
├── 影子链
└── 特点: 最终验证,风险评估

生产环境 (Production)
├── 主网 (Ethereum/Polygon/BSC)
├── Layer2 (Arbitrum/Optimism)
└── 特点: 真实资金,高可用性
```

#### 1.2 环境配置管理

**配置文件结构**:
```javascript
// config/environments.js
module.exports = {
  development: {
    network: 'localhost',
    rpcUrl: 'http://127.0.0.1:8545',
    chainId: 31337,
    gasPrice: 'auto',
    gas: 'auto',
    confirmations: 1,
    skipDryRun: true,
    features: {
      debugging: true,
      verbose: true,
      autoMining: true
    }
  },
  
  testnet: {
    network: 'sepolia',
    rpcUrl: process.env.SEPOLIA_RPC_URL,
    chainId: 11155111,
    gasPrice: 'auto',
    gas: 8000000,
    confirmations: 2,
    skipDryRun: false,
    timeoutBlocks: 200,
    features: {
      debugging: true,
      verbose: true,
      etherscan: {
        apiKey: process.env.ETHERSCAN_API_KEY,
        autoVerify: true
      }
    }
  },
  
  staging: {
    network: 'polygon-mumbai',
    rpcUrl: process.env.MUMBAI_RPC_URL,
    chainId: 80001,
    gasPrice: 'auto',
    gas: 8000000,
    confirmations: 3,
    skipDryRun: false,
    timeoutBlocks: 200,
    features: {
      debugging: false,
      verbose: false,
      monitoring: true,
      alerting: true,
      polygonscan: {
        apiKey: process.env.POLYGONSCAN_API_KEY,
        autoVerify: true
      }
    }
  },
  
  production: {
    network: 'mainnet',
    rpcUrl: process.env.MAINNET_RPC_URL,
    chainId: 1,
    gasPrice: process.env.GAS_PRICE || 'auto',
    gas: 8000000,
    confirmations: 5,
    skipDryRun: false,
    timeoutBlocks: 200,
    features: {
      debugging: false,
      verbose: false,
      monitoring: true,
      alerting: true,
      backup: true,
      multiSig: true,
      etherscan: {
        apiKey: process.env.ETHERSCAN_API_KEY,
        autoVerify: true
      }
    }
  }
};
```

**Hardhat 多环境配置**:
```javascript
// hardhat.config.js
require('@nomiclabs/hardhat-waffle');
require('@nomiclabs/hardhat-etherscan');
require('hardhat-deploy');
require('hardhat-gas-reporter');
require('solidity-coverage');
require('dotenv').config();

const environments = require('./config/environments');

// 获取私钥
function getAccounts(environment) {
  const privateKey = process.env[`${environment.toUpperCase()}_PRIVATE_KEY`];
  return privateKey ? [privateKey] : [];
}

module.exports = {
  solidity: {
    version: '0.8.19',
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
      forking: process.env.MAINNET_RPC_URL ? {
        url: process.env.MAINNET_RPC_URL,
        blockNumber: parseInt(process.env.FORK_BLOCK_NUMBER || '0')
      } : undefined
    },
    
    localhost: {
      url: 'http://127.0.0.1:8545',
      chainId: 31337
    },
    
    sepolia: {
      url: environments.testnet.rpcUrl,
      chainId: environments.testnet.chainId,
      accounts: getAccounts('sepolia'),
      gasPrice: environments.testnet.gasPrice,
      gas: environments.testnet.gas
    },
    
    mumbai: {
      url: environments.staging.rpcUrl,
      chainId: environments.staging.chainId,
      accounts: getAccounts('mumbai'),
      gasPrice: environments.staging.gasPrice,
      gas: environments.staging.gas
    },
    
    mainnet: {
      url: environments.production.rpcUrl,
      chainId: environments.production.chainId,
      accounts: getAccounts('mainnet'),
      gasPrice: environments.production.gasPrice,
      gas: environments.production.gas
    },
    
    polygon: {
      url: process.env.POLYGON_RPC_URL,
      chainId: 137,
      accounts: getAccounts('polygon'),
      gasPrice: 'auto',
      gas: 8000000
    }
  },
  
  namedAccounts: {
    deployer: {
      default: 0,
      1: process.env.MAINNET_DEPLOYER_ADDRESS,
      137: process.env.POLYGON_DEPLOYER_ADDRESS
    },
    admin: {
      default: 1,
      1: process.env.MAINNET_ADMIN_ADDRESS
    }
  },
  
  etherscan: {
    apiKey: {
      mainnet: process.env.ETHERSCAN_API_KEY,
      sepolia: process.env.ETHERSCAN_API_KEY,
      polygon: process.env.POLYGONSCAN_API_KEY,
      polygonMumbai: process.env.POLYGONSCAN_API_KEY
    }
  },
  
  gasReporter: {
    enabled: process.env.REPORT_GAS === 'true',
    currency: 'USD',
    coinmarketcap: process.env.COINMARKETCAP_API_KEY,
    outputFile: 'gas-report.txt',
    noColors: true
  },
  
  paths: {
    sources: './contracts',
    tests: './test',
    cache: './cache',
    artifacts: './artifacts',
    deploy: './deploy',
    deployments: './deployments'
  }
};
```

### 2. 自动化部署脚本

#### 2.1 部署脚本结构

**使用 Hardhat Deploy 插件**:
```javascript
// deploy/01_deploy_token.js
module.exports = async ({ getNamedAccounts, deployments, network }) => {
  const { deploy, log, get } = deployments;
  const { deployer } = await getNamedAccounts();
  
  log('----------------------------------------------------');
  log(`Deploying MyToken to ${network.name}`);
  log(`Deployer: ${deployer}`);
  
  // 部署前检查
  const balance = await ethers.provider.getBalance(deployer);
  log(`Deployer balance: ${ethers.utils.formatEther(balance)} ETH`);
  
  if (balance.lt(ethers.utils.parseEther('0.1'))) {
    throw new Error('Insufficient balance for deployment');
  }
  
  // 获取构造函数参数
  const args = [
    'MyToken',
    'MTK',
    ethers.utils.parseEther('1000000')
  ];
  
  // 部署合约
  const token = await deploy('MyToken', {
    from: deployer,
    args: args,
    log: true,
    waitConfirmations: network.config.confirmations || 1,
    skipIfAlreadyDeployed: true
  });
  
  log(`MyToken deployed at: ${token.address}`);
  
  // 验证合约
  if (network.name !== 'hardhat' && network.name !== 'localhost') {
    log('Waiting for block confirmations...');
    await token.deployTransaction.wait(6);
    await verify(token.address, args);
  }
  
  // 保存部署信息
  await saveDeploymentInfo(network.name, 'MyToken', token);
  
  log('----------------------------------------------------');
};

module.exports.tags = ['Token', 'All'];
```

**部署 NFT 合约**:
```javascript
// deploy/02_deploy_nft.js
module.exports = async ({ getNamedAccounts, deployments, network }) => {
  const { deploy, log, get } = deployments;
  const { deployer, admin } = await getNamedAccounts();
  
  log('----------------------------------------------------');
  log(`Deploying MyNFT to ${network.name}`);
  
  // 获取依赖合约
  const token = await get('MyToken');
  
  const args = [
    'MyNFT',
    'MNFT',
    'https://api.mynft.com/metadata/',
    token.address,
    admin
  ];
  
  const nft = await deploy('MyNFT', {
    from: deployer,
    args: args,
    log: true,
    waitConfirmations: network.config.confirmations || 1,
    proxy: {
      proxyContract: 'OpenZeppelinTransparentProxy',
      execute: {
        init: {
          methodName: 'initialize',
          args: args
        }
      }
    }
  });
  
  log(`MyNFT deployed at: ${nft.address}`);
  log(`Implementation: ${nft.implementation}`);
  log(`Proxy Admin: ${nft.proxyAdmin}`);
  
  // 配置角色
  if (network.name !== 'hardhat') {
    await setupRoles(nft, deployer, admin);
  }
  
  // 验证合约
  if (network.name !== 'hardhat' && network.name !== 'localhost') {
    await nft.deployTransaction.wait(6);
    await verify(nft.implementation, args);
  }
  
  log('----------------------------------------------------');
};

module.exports.tags = ['NFT', 'All'];
module.exports.dependencies = ['Token'];
```

**复杂系统部署**:
```javascript
// deploy/03_deploy_marketplace.js
module.exports = async ({ getNamedAccounts, deployments, network, ethers }) => {
  const { deploy, log, get, execute } = deployments;
  const { deployer, admin } = await getNamedAccounts();
  
  log('----------------------------------------------------');
  log('Deploying Marketplace System');
  
  // 1. 部署市场合约
  const marketplace = await deploy('NFTMarketplace', {
    from: deployer,
    args: [],
    log: true,
    waitConfirmations: network.config.confirmations || 1
  });
  
  // 2. 部署拍卖合约
  const auction = await deploy('NFTAuction', {
    from: deployer,
    args: [marketplace.address],
    log: true,
    waitConfirmations: network.config.confirmations || 1
  });
  
  // 3. 部署版税管理合约
  const royalty = await deploy('RoyaltyManager', {
    from: deployer,
    args: [],
    log: true,
    waitConfirmations: network.config.confirmations || 1
  });
  
  // 4. 配置合约关系
  log('Configuring contract relationships...');
  
  await execute(
    'NFTMarketplace',
    { from: deployer, log: true },
    'setAuctionContract',
    auction.address
  );
  
  await execute(
    'NFTMarketplace',
    { from: deployer, log: true },
    'setRoyaltyManager',
    royalty.address
  );
  
  await execute(
    'NFTAuction',
    { from: deployer, log: true },
    'setRoyaltyManager',
    royalty.address
  );
  
  // 5. 设置手续费
  const feePercent = 250; // 2.5%
  await execute(
    'NFTMarketplace',
    { from: deployer, log: true },
    'setFeePercent',
    feePercent
  );
  
  // 6. 转移所有权
  if (admin !== deployer) {
    log('Transferring ownership to admin...');
    await execute(
      'NFTMarketplace',
      { from: deployer, log: true },
      'transferOwnership',
      admin
    );
    
    await execute(
      'NFTAuction',
      { from: deployer, log: true },
      'transferOwnership',
      admin
    );
    
    await execute(
      'RoyaltyManager',
      { from: deployer, log: true },
      'transferOwnership',
      admin
    );
  }
  
  // 7. 保存部署清单
  await saveDeploymentManifest(network.name, {
    marketplace: marketplace.address,
    auction: auction.address,
    royalty: royalty.address,
    deployer: deployer,
    admin: admin,
    feePercent: feePercent,
    timestamp: Date.now()
  });
  
  log('Marketplace system deployed successfully');
  log('----------------------------------------------------');
};

module.exports.tags = ['Marketplace', 'All'];
module.exports.dependencies = ['NFT'];
```

#### 2.2 部署工具函数

**验证合约**:
```javascript
// scripts/utils/verify.js
const { run } = require('hardhat');

async function verify(contractAddress, args) {
  console.log('Verifying contract...');
  try {
    await run('verify:verify', {
      address: contractAddress,
      constructorArguments: args
    });
    console.log('Contract verified successfully');
  } catch (error) {
    if (error.message.toLowerCase().includes('already verified')) {
      console.log('Contract already verified');
    } else {
      console.error('Verification failed:', error);
    }
  }
}

module.exports = { verify };
```

**保存部署信息**:
```javascript
// scripts/utils/deployment.js
const fs = require('fs');
const path = require('path');

async function saveDeploymentInfo(network, contractName, deployment) {
  const deploymentDir = path.join(__dirname, '../../deployments-info', network);
  
  if (!fs.existsSync(deploymentDir)) {
    fs.mkdirSync(deploymentDir, { recursive: true });
  }
  
  const info = {
    name: contractName,
    address: deployment.address,
    implementation: deployment.implementation,
    transactionHash: deployment.transactionHash,
    deployer: deployment.receipt.from,
    blockNumber: deployment.receipt.blockNumber,
    gasUsed: deployment.receipt.gasUsed.toString(),
    timestamp: new Date().toISOString()
  };
  
  const filePath = path.join(deploymentDir, `${contractName}.json`);
  fs.writeFileSync(filePath, JSON.stringify(info, null, 2));
  
  console.log(`Deployment info saved to ${filePath}`);
}

async function saveDeploymentManifest(network, data) {
  const filePath = path.join(
    __dirname,
    '../../deployments-info',
    network,
    'manifest.json'
  );
  
  let manifest = {};
  if (fs.existsSync(filePath)) {
    manifest = JSON.parse(fs.readFileSync(filePath, 'utf8'));
  }
  
  manifest = { ...manifest, ...data };
  fs.writeFileSync(filePath, JSON.stringify(manifest, null, 2));
  
  console.log(`Manifest updated: ${filePath}`);
}

async function loadDeploymentInfo(network, contractName) {
  const filePath = path.join(
    __dirname,
    '../../deployments-info',
    network,
    `${contractName}.json`
  );
  
  if (!fs.existsSync(filePath)) {
    throw new Error(`Deployment info not found for ${contractName} on ${network}`);
  }
  
  return JSON.parse(fs.readFileSync(filePath, 'utf8'));
}

module.exports = {
  saveDeploymentInfo,
  saveDeploymentManifest,
  loadDeploymentInfo
};
```

**设置角色和权限**:
```javascript
// scripts/utils/roles.js
async function setupRoles(deployment, deployer, admin) {
  const contract = await ethers.getContractAt(
    deployment.abi,
    deployment.address
  );
  
  console.log('Setting up roles...');
  
  // 授予管理员角色
  const ADMIN_ROLE = await contract.DEFAULT_ADMIN_ROLE();
  const MINTER_ROLE = await contract.MINTER_ROLE();
  const PAUSER_ROLE = await contract.PAUSER_ROLE();
  
  const hasAdminRole = await contract.hasRole(ADMIN_ROLE, admin);
  if (!hasAdminRole) {
    const tx = await contract.grantRole(ADMIN_ROLE, admin);
    await tx.wait();
    console.log(`Granted ADMIN_ROLE to ${admin}`);
  }
  
  // 授予铸造者角色
  const hasMinterRole = await contract.hasRole(MINTER_ROLE, admin);
  if (!hasMinterRole) {
    const tx = await contract.grantRole(MINTER_ROLE, admin);
    await tx.wait();
    console.log(`Granted MINTER_ROLE to ${admin}`);
  }
  
  // 授予暂停者角色
  const hasPauserRole = await contract.hasRole(PAUSER_ROLE, admin);
  if (!hasPauserRole) {
    const tx = await contract.grantRole(PAUSER_ROLE, admin);
    await tx.wait();
    console.log(`Granted PAUSER_ROLE to ${admin}`);
  }
  
  // 撤销部署者的铸造权限(如果需要)
  if (deployer !== admin) {
    const deployerHasMinter = await contract.hasRole(MINTER_ROLE, deployer);
    if (deployerHasMinter) {
      const tx = await contract.revokeRole(MINTER_ROLE, deployer);
      await tx.wait();
      console.log(`Revoked MINTER_ROLE from deployer ${deployer}`);
    }
  }
  
  console.log('Roles setup completed');
}

module.exports = { setupRoles };
```

### 3. GitHub Actions 自动部署

#### 3.1 测试网自动部署

```yaml
# .github/workflows/deploy-testnet.yaml
name: Deploy to Testnet

on:
  push:
    branches:
      - develop
  workflow_dispatch:
    inputs:
      network:
        description: 'Target network'
        required: true
        default: 'sepolia'
        type: choice
        options:
          - sepolia
          - mumbai
          - goerli

env:
  NODE_VERSION: '18'

jobs:
  deploy:
    name: Deploy Contracts
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Compile contracts
        run: npm run compile
      
      - name: Run tests
        run: npm test
      
      - name: Set network
        id: network
        run: |
          if [ "${{ github.event_name }}" == "workflow_dispatch" ]; then
            echo "NETWORK=${{ github.event.inputs.network }}" >> $GITHUB_ENV
          else
            echo "NETWORK=sepolia" >> $GITHUB_ENV
          fi
      
      - name: Deploy to ${{ env.NETWORK }}
        env:
          SEPOLIA_RPC_URL: ${{ secrets.SEPOLIA_RPC_URL }}
          SEPOLIA_PRIVATE_KEY: ${{ secrets.SEPOLIA_PRIVATE_KEY }}
          MUMBAI_RPC_URL: ${{ secrets.MUMBAI_RPC_URL }}
          MUMBAI_PRIVATE_KEY: ${{ secrets.MUMBAI_PRIVATE_KEY }}
          GOERLI_RPC_URL: ${{ secrets.GOERLI_RPC_URL }}
          GOERLI_PRIVATE_KEY: ${{ secrets.GOERLI_PRIVATE_KEY }}
          ETHERSCAN_API_KEY: ${{ secrets.ETHERSCAN_API_KEY }}
          POLYGONSCAN_API_KEY: ${{ secrets.POLYGONSCAN_API_KEY }}
        run: |
          npx hardhat deploy --network ${{ env.NETWORK }} --tags All
      
      - name: Verify contracts
        env:
          ETHERSCAN_API_KEY: ${{ secrets.ETHERSCAN_API_KEY }}
          POLYGONSCAN_API_KEY: ${{ secrets.POLYGONSCAN_API_KEY }}
        run: |
          npx hardhat --network ${{ env.NETWORK }} etherscan-verify
      
      - name: Generate deployment report
        run: |
          npm run deployment:report -- --network ${{ env.NETWORK }}
      
      - name: Upload deployment artifacts
        uses: actions/upload-artifact@v3
        with:
          name: deployment-${{ env.NETWORK }}-${{ github.sha }}
          path: |
            deployments/
            deployments-info/
            deployment-report.md
      
      - name: Comment on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('deployment-report.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Deployment Report\n\n${report}`
            });
      
      - name: Notify on failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Testnet deployment failed!'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

#### 3.2 主网部署工作流

```yaml
# .github/workflows/deploy-mainnet.yaml
name: Deploy to Mainnet

on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Type "DEPLOY" to confirm mainnet deployment'
        required: true
      network:
        description: 'Target mainnet'
        required: true
        default: 'mainnet'
        type: choice
        options:
          - mainnet
          - polygon
          - arbitrum
          - optimism

jobs:
  pre-deployment-checks:
    name: Pre-deployment Checks
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Verify confirmation
        if: github.event_name == 'workflow_dispatch'
        run: |
          if [ "${{ github.event.inputs.confirm }}" != "DEPLOY" ]; then
            echo "Deployment not confirmed!"
            exit 1
          fi
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run full test suite
        run: npm run test:full
      
      - name: Run security checks
        run: |
          npm run security:check
          npm run audit
      
      - name: Check gas costs
        run: npm run gas:report
      
      - name: Generate coverage report
        run: npm run coverage
      
      - name: Upload reports
        uses: actions/upload-artifact@v3
        with:
          name: pre-deployment-reports
          path: |
            coverage/
            gas-report.txt
            security-report.json

  approve-deployment:
    name: Manual Approval
    runs-on: ubuntu-latest
    needs: pre-deployment-checks
    environment:
      name: production
      url: https://etherscan.io
    
    steps:
      - name: Wait for approval
        run: echo "Deployment approved"

  deploy-mainnet:
    name: Deploy to Mainnet
    runs-on: ubuntu-latest
    needs: approve-deployment
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Set network
        id: network
        run: |
          if [ "${{ github.event_name }}" == "workflow_dispatch" ]; then
            echo "NETWORK=${{ github.event.inputs.network }}" >> $GITHUB_ENV
          else
            echo "NETWORK=mainnet" >> $GITHUB_ENV
          fi
      
      - name: Check deployer balance
        env:
          MAINNET_RPC_URL: ${{ secrets.MAINNET_RPC_URL }}
          MAINNET_PRIVATE_KEY: ${{ secrets.MAINNET_PRIVATE_KEY }}
        run: |
          node scripts/check-balance.js --network ${{ env.NETWORK }}
      
      - name: Deploy contracts
        env:
          MAINNET_RPC_URL: ${{ secrets.MAINNET_RPC_URL }}
          MAINNET_PRIVATE_KEY: ${{ secrets.MAINNET_PRIVATE_KEY }}
          POLYGON_RPC_URL: ${{ secrets.POLYGON_RPC_URL }}
          POLYGON_PRIVATE_KEY: ${{ secrets.POLYGON_PRIVATE_KEY }}
          ARBITRUM_RPC_URL: ${{ secrets.ARBITRUM_RPC_URL }}
          ARBITRUM_PRIVATE_KEY: ${{ secrets.ARBITRUM_PRIVATE_KEY }}
          OPTIMISM_RPC_URL: ${{ secrets.OPTIMISM_RPC_URL }}
          OPTIMISM_PRIVATE_KEY: ${{ secrets.OPTIMISM_PRIVATE_KEY }}
          ETHERSCAN_API_KEY: ${{ secrets.ETHERSCAN_API_KEY }}
          POLYGONSCAN_API_KEY: ${{ secrets.POLYGONSCAN_API_KEY }}
          ARBISCAN_API_KEY: ${{ secrets.ARBISCAN_API_KEY }}
          OPTIMISTIC_ETHERSCAN_API_KEY: ${{ secrets.OPTIMISTIC_ETHERSCAN_API_KEY }}
          GAS_PRICE: ${{ secrets.GAS_PRICE }}
        run: |
          npx hardhat deploy \
            --network ${{ env.NETWORK }} \
            --tags All \
            --export deployments-export.json
      
      - name: Verify contracts
        run: |
          npx hardhat --network ${{ env.NETWORK }} etherscan-verify
      
      - name: Run post-deployment tests
        run: |
          npm run test:integration -- --network ${{ env.NETWORK }}
      
      - name: Generate deployment documentation
        run: |
          node scripts/generate-docs.js --network ${{ env.NETWORK }}
      
      - name: Backup deployment data
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          npm run backup:deployment -- --network ${{ env.NETWORK }}
      
      - name: Create deployment summary
        run: |
          node scripts/create-summary.js --network ${{ env.NETWORK }}
      
      - name: Upload deployment artifacts
        uses: actions/upload-artifact@v3
        with:
          name: mainnet-deployment-${{ github.sha }}
          path: |
            deployments/
            deployments-info/
            deployments-export.json
            deployment-docs/
      
      - name: Notify success
        uses: 8398a7/action-slack@v3
        with:
          status: success
          text: 'Mainnet deployment successful! 🚀'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
      
      - name: Notify failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          text: 'Mainnet deployment failed! ⚠️'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}

  post-deployment:
    name: Post-deployment Tasks
    runs-on: ubuntu-latest
    needs: deploy-mainnet
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup monitoring
        run: |
          node scripts/setup-monitoring.js --network ${{ env.NETWORK }}
      
      - name: Configure alerts
        run: |
          node scripts/configure-alerts.js --network ${{ env.NETWORK }}
      
      - name: Update documentation
        run: |
          node scripts/update-docs.js
      
      - name: Create GitHub release notes
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const summary = fs.readFileSync('deployment-summary.md', 'utf8');
            await github.rest.repos.updateRelease({
              owner: context.repo.owner,
              repo: context.repo.repo,
              release_id: context.payload.release.id,
              body: context.payload.release.body + '\n\n## Deployment Summary\n\n' + summary
            });
```

### 4. 部署验证和回滚

#### 4.1 部署后验证脚本

```javascript
// scripts/verify-deployment.js
const hre = require('hardhat');
const { loadDeploymentInfo } = require('./utils/deployment');

async function verifyDeployment(network, contractName) {
  console.log(`Verifying ${contractName} deployment on ${network}...`);
  
  // 1. 加载部署信息
  const deployment = await loadDeploymentInfo(network, contractName);
  console.log(`Contract address: ${deployment.address}`);
  
  // 2. 验证合约代码
  const code = await hre.ethers.provider.getCode(deployment.address);
  if (code === '0x') {
    throw new Error('No contract code at address');
  }
  console.log('✓ Contract code exists');
  
  // 3. 获取合约实例
  const contract = await hre.ethers.getContractAt(
    contractName,
    deployment.address
  );
  
  // 4. 验证合约状态
  try {
    const name = await contract.name();
    console.log(`✓ Contract name: ${name}`);
    
    const symbol = await contract.symbol();
    console.log(`✓ Contract symbol: ${symbol}`);
    
    const owner = await contract.owner();
    console.log(`✓ Contract owner: ${owner}`);
    
    const paused = await contract.paused();
    console.log(`✓ Contract paused: ${paused}`);
  } catch (error) {
    console.error('Contract state verification failed:', error);
    throw error;
  }
  
  // 5. 验证权限配置
  const ADMIN_ROLE = await contract.DEFAULT_ADMIN_ROLE();
  const MINTER_ROLE = await contract.MINTER_ROLE();
  
  const adminAddress = process.env[`${network.toUpperCase()}_ADMIN_ADDRESS`];
  const hasAdminRole = await contract.hasRole(ADMIN_ROLE, adminAddress);
  const hasMinterRole = await contract.hasRole(MINTER_ROLE, adminAddress);
  
  if (!hasAdminRole) {
    throw new Error(`Admin does not have ADMIN_ROLE`);
  }
  console.log('✓ Admin role configured correctly');
  
  if (!hasMinterRole) {
    throw new Error(`Admin does not have MINTER_ROLE`);
  }
  console.log('✓ Minter role configured correctly');
  
  // 6. 测试基本功能
  console.log('Testing basic functionality...');
  
  const signer = await hre.ethers.getSigner();
  const balance = await contract.balanceOf(signer.address);
  console.log(`✓ Balance query works: ${balance.toString()}`);
  
  // 7. 验证事件
  const filter = contract.filters.Transfer();
  const events = await contract.queryFilter(filter, -100);
  console.log(`✓ Found ${events.length} Transfer events in last 100 blocks`);
  
  console.log('\n✅ Deployment verification completed successfully');
  return true;
}

async function verifySystemDeployment(network) {
  console.log(`\n${'='.repeat(60)}`);
  console.log(`Verifying complete system deployment on ${network}`);
  console.log('='.repeat(60));
  
  const contracts = ['MyToken', 'MyNFT', 'NFTMarketplace', 'NFTAuction'];
  
  for (const contract of contracts) {
    try {
      await verifyDeployment(network, contract);
    } catch (error) {
      console.error(`\n❌ Verification failed for ${contract}:`, error.message);
      return false;
    }
  }
  
  // 验证合约之间的关系
  console.log('\nVerifying contract relationships...');
  
  const marketplace = await hre.ethers.getContract('NFTMarketplace');
  const auction = await hre.ethers.getContract('NFTAuction');
  
  const auctionAddress = await marketplace.auctionContract();
  if (auctionAddress !== auction.address) {
    throw new Error('Auction contract not properly linked to marketplace');
  }
  console.log('✓ Contract relationships verified');
  
  console.log('\n✅ System deployment verification completed successfully');
  return true;
}

// 主函数
async function main() {
  const network = process.env.HARDHAT_NETWORK || hre.network.name;
  const success = await verifySystemDeployment(network);
  
  if (!success) {
    process.exit(1);
  }
}

if (require.main === module) {
  main()
    .then(() => process.exit(0))
    .catch((error) => {
      console.error(error);
      process.exit(1);
    });
}

module.exports = { verifyDeployment, verifySystemDeployment };
```

#### 4.2 回滚机制

```javascript
// scripts/rollback.js
const hre = require('hardhat');
const fs = require('fs');
const path = require('path');

async function loadBackup(network, timestamp) {
  const backupPath = path.join(
    __dirname,
    '../backups',
    network,
    `${timestamp}.json`
  );
  
  if (!fs.existsSync(backupPath)) {
    throw new Error(`Backup not found: ${backupPath}`);
  }
  
  return JSON.parse(fs.readFileSync(backupPath, 'utf8'));
}

async function createBackup(network) {
  const timestamp = Date.now();
  const backupDir = path.join(__dirname, '../backups', network);
  
  if (!fs.existsSync(backupDir)) {
    fs.mkdirSync(backupDir, { recursive: true });
  }
  
  // 读取当前部署状态
  const deployments = {};
  const deploymentDir = path.join(__dirname, '../deployments', network);
  
  if (fs.existsSync(deploymentDir)) {
    const files = fs.readdirSync(deploymentDir);
    for (const file of files) {
      if (file.endsWith('.json') && file !== '.chainId') {
        const content = fs.readFileSync(
          path.join(deploymentDir, file),
          'utf8'
        );
        deployments[file] = JSON.parse(content);
      }
    }
  }
  
  const backup = {
    timestamp,
    network,
    deployments,
    git: {
      commit: process.env.GITHUB_SHA || 'unknown',
      branch: process.env.GITHUB_REF || 'unknown'
    }
  };
  
  const backupPath = path.join(backupDir, `${timestamp}.json`);
  fs.writeFileSync(backupPath, JSON.stringify(backup, null, 2));
  
  console.log(`Backup created: ${backupPath}`);
  return timestamp;
}

async function rollbackTo(network, timestamp) {
  console.log(`Rolling back ${network} to ${timestamp}...`);
  
  // 1. 创建当前状态备份
  console.log('Creating backup of current state...');
  await createBackup(network);
  
  // 2. 加载目标备份
  console.log('Loading target backup...');
  const backup = await loadBackup(network, timestamp);
  
  // 3. 恢复部署文件
  console.log('Restoring deployment files...');
  const deploymentDir = path.join(__dirname, '../deployments', network);
  
  if (!fs.existsSync(deploymentDir)) {
    fs.mkdirSync(deploymentDir, { recursive: true });
  }
  
  for (const [filename, content] of Object.entries(backup.deployments)) {
    const filePath = path.join(deploymentDir, filename);
    fs.writeFileSync(filePath, JSON.stringify(content, null, 2));
    console.log(`✓ Restored ${filename}`);
  }
  
  console.log('Rollback completed successfully');
}

async function listBackups(network) {
  const backupDir = path.join(__dirname, '../backups', network);
  
  if (!fs.existsSync(backupDir)) {
    console.log('No backups found');
    return [];
  }
  
  const files = fs.readdirSync(backupDir);
  const backups = [];
  
  for (const file of files) {
    if (file.endsWith('.json')) {
      const backup = JSON.parse(
        fs.readFileSync(path.join(backupDir, file), 'utf8')
      );
      backups.push({
        timestamp: backup.timestamp,
        date: new Date(backup.timestamp).toISOString(),
        commit: backup.git.commit,
        contracts: Object.keys(backup.deployments).length
      });
    }
  }
  
  backups.sort((a, b) => b.timestamp - a.timestamp);
  
  console.log('\nAvailable backups:');
  console.log('─'.repeat(80));
  backups.forEach((backup, index) => {
    console.log(
      `${index + 1}. ${backup.date} | ` +
      `Commit: ${backup.commit.substring(0, 8)} | ` +
      `Contracts: ${backup.contracts}`
    );
  });
  console.log('─'.repeat(80));
  
  return backups;
}

// 主函数
async function main() {
  const command = process.argv[2];
  const network = process.env.HARDHAT_NETWORK || hre.network.name;
  
  switch (command) {
    case 'backup':
      await createBackup(network);
      break;
    
    case 'list':
      await listBackups(network);
      break;
    
    case 'rollback':
      const timestamp = process.argv[3];
      if (!timestamp) {
        console.error('Please provide a timestamp');
        process.exit(1);
      }
      await rollbackTo(network, parseInt(timestamp));
      break;
    
    default:
      console.log('Usage:');
      console.log('  npm run rollback backup');
      console.log('  npm run rollback list');
      console.log('  npm run rollback rollback <timestamp>');
      process.exit(1);
  }
}

if (require.main === module) {
  main()
    .then(() => process.exit(0))
    .catch((error) => {
      console.error(error);
      process.exit(1);
    });
}

module.exports = {
  createBackup,
  rollbackTo,
  listBackups
};
```

### 5. 高级部署策略

#### 5.1 蓝绿部署

```javascript
// scripts/blue-green-deployment.js
const hre = require('hardhat');

async function blueGreenDeployment(contractName, args) {
  console.log(`Starting blue-green deployment for ${contractName}...`);
  
  // 1. 获取当前活跃的合约 (蓝)
  const registry = await hre.ethers.getContract('ContractRegistry');
  const blueAddress = await registry.getContract(contractName);
  
  console.log(`Current (blue) contract: ${blueAddress}`);
  
  // 2. 部署新版本 (绿)
  console.log('Deploying new (green) version...');
  const Contract = await hre.ethers.getContractFactory(contractName);
  const greenContract = await Contract.deploy(...args);
  await greenContract.deployed();
  
  console.log(`New (green) contract deployed: ${greenContract.address}`);
  
  // 3. 验证新合约
  console.log('Verifying new contract...');
  await verifyDeployment(hre.network.name, contractName);
  
  // 4. 迁移状态(如果需要)
  if (blueAddress !== hre.ethers.constants.AddressZero) {
    console.log('Migrating state...');
    await migrateState(blueAddress, greenContract.address);
  }
  
  // 5. 切换流量到新合约
  console.log('Switching traffic to green...');
  const tx = await registry.updateContract(contractName, greenContract.address);
  await tx.wait();
  
  console.log('Traffic switched to green contract');
  
  // 6. 监控新合约
  console.log('Monitoring new contract...');
  const healthy = await monitorContract(greenContract.address, 300); // 5分钟
  
  if (!healthy) {
    // 回滚到蓝色版本
    console.error('Health check failed! Rolling back...');
    const rollbackTx = await registry.updateContract(contractName, blueAddress);
    await rollbackTx.wait();
    throw new Error('Deployment rolled back due to health check failure');
  }
  
  console.log('Health check passed! Deployment successful');
  
  // 7. 保留蓝色版本一段时间以便快速回滚
  console.log(`Blue contract ${blueAddress} will be kept for 24 hours`);
  
  return {
    blue: blueAddress,
    green: greenContract.address,
    registry: registry.address
  };
}

async function migrateState(oldAddress, newAddress) {
  // 实现状态迁移逻辑
  const oldContract = await hre.ethers.getContractAt('MyContract', oldAddress);
  const newContract = await hre.ethers.getContractAt('MyContract', newAddress);
  
  // 示例: 迁移所有者列表
  const ownerCount = await oldContract.getOwnerCount();
  for (let i = 0; i < ownerCount; i++) {
    const owner = await oldContract.getOwnerAt(i);
    const tx = await newContract.addOwner(owner);
    await tx.wait();
    console.log(`Migrated owner ${i + 1}/${ownerCount}: ${owner}`);
  }
}

async function monitorContract(address, duration) {
  const contract = await hre.ethers.getContractAt('MyContract', address);
  const startTime = Date.now();
  const endTime = startTime + duration * 1000;
  
  while (Date.now() < endTime) {
    try {
      // 检查合约是否响应
      await contract.healthCheck();
      
      // 检查错误率
      const errorCount = await contract.getErrorCount();
      if (errorCount > 10) {
        console.error('Too many errors detected');
        return false;
      }
      
      // 等待下一次检查
      await new Promise(resolve => setTimeout(resolve, 10000)); // 10秒
    } catch (error) {
      console.error('Health check failed:', error);
      return false;
    }
  }
  
  return true;
}

module.exports = { blueGreenDeployment };
```

#### 5.2 金丝雀发布

```javascript
// scripts/canary-deployment.js
const hre = require('hardhat');

async function canaryDeployment(contractName, args, canaryPercent = 10) {
  console.log(`Starting canary deployment for ${contractName}...`);
  console.log(`Canary traffic percentage: ${canaryPercent}%`);
  
  // 1. 获取当前生产合约
  const registry = await hre.ethers.getContract('ContractRegistry');
  const productionAddress = await registry.getContract(contractName);
  
  // 2. 部署金丝雀版本
  console.log('Deploying canary version...');
  const Contract = await hre.ethers.getContractFactory(contractName);
  const canaryContract = await Contract.deploy(...args);
  await canaryContract.deployed();
  
  console.log(`Canary deployed: ${canaryContract.address}`);
  
  // 3. 注册金丝雀版本
  const tx = await registry.setCanary(
    contractName,
    canaryContract.address,
    canaryPercent
  );
  await tx.wait();
  
  // 4. 逐步增加流量
  const steps = [10, 25, 50, 75, 100];
  
  for (const percent of steps) {
    if (percent <= canaryPercent) continue;
    
    console.log(`\nIncreasing canary traffic to ${percent}%...`);
    
    // 监控指标
    const metrics = await collectMetrics(canaryContract.address, 300);
    
    if (!isHealthy(metrics)) {
      console.error('Canary metrics unhealthy! Rolling back...');
      await registry.removeCanary(contractName);
      throw new Error('Canary deployment failed');
    }
    
    // 增加流量
    const updateTx = await registry.setCanary(
      contractName,
      canaryContract.address,
      percent
    );
    await updateTx.wait();
    
    console.log(`Traffic increased to ${percent}%`);
  }
  
  // 5. 完全切换到金丝雀版本
  console.log('\nPromoting canary to production...');
  const promoteTx = await registry.promoteCanary(contractName);
  await promoteTx.wait();
  
  console.log('Canary successfully promoted to production!');
  
  return {
    old: productionAddress,
    new: canaryContract.address
  };
}

async function collectMetrics(address, duration) {
  // 收集合约指标
  return {
    errorRate: 0.01,
    avgResponseTime: 150,
    successRate: 0.99,
    gasUsage: 50000
  };
}

function isHealthy(metrics) {
  return (
    metrics.errorRate < 0.05 &&
    metrics.avgResponseTime < 500 &&
    metrics.successRate > 0.95
  );
}

module.exports = { canaryDeployment };
```

## 实践练习

### 练习1: 多环境部署配置

创建完整的多环境配置并实现自动化部署:

```javascript
// 任务:
// 1. 配置开发、测试、预生产和生产环境
// 2. 实现环境特定的部署脚本
// 3. 创建部署验证流程
// 4. 实现部署回滚机制
```

### 练习2: CI/CD 流水线

实现完整的 CI/CD 流水线:

```yaml
# 任务:
# 1. 创建测试网自动部署工作流
# 2. 创建主网手动审批部署工作流
# 3. 实现部署后自动验证
# 4. 配置部署通知
```

### 练习3: 蓝绿部署

实现蓝绿部署策略:

```javascript
// 任务:
// 1. 创建合约注册表
// 2. 实现蓝绿部署脚本
// 3. 实现流量切换
// 4. 实现健康检查和自动回滚
```

## 学习检查清单

- [ ] 理解多环境部署策略
- [ ] 能够配置不同环境的参数
- [ ] 掌握 Hardhat Deploy 插件使用
- [ ] 能够编写部署脚本和工具函数
- [ ] 理解 GitHub Actions 部署工作流
- [ ] 能够实现部署验证流程
- [ ] 掌握部署回滚机制
- [ ] 理解蓝绿部署和金丝雀发布
- [ ] 能够实现自动化部署流程
- [ ] 掌握密钥和配置管理

## 扩展阅读

- [Hardhat Deploy 文档](https://github.com/wighawag/hardhat-deploy)
- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [蓝绿部署最佳实践](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- [金丝雀发布策略](https://martinfowler.com/bliki/CanaryRelease.html)

## 下节预告

下一节我们将学习**合约源码验证与前端部署**,包括:
- Etherscan 源码验证
- 多区块链浏览器验证
- 前端应用部署
- IPFS 部署
- CDN 配置和优化
