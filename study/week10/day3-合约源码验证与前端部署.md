# Week 10 Day 3: 合约源码验证与前端部署

**预计学习时间**: 6-8小时  
**难度**: ⭐⭐⭐

## 学习目标

通过本节课程,你将能够:
- 掌握智能合约源码自动化验证方法
- 实现多链环境下的合约验证
- 部署 Web3 前端应用到 Vercel/Netlify
- 将前端应用部署到去中心化存储 (IPFS/Arweave)
- 配置自定义域名和 CDN 加速

## 课程内容

### 1. 智能合约源码验证

#### 1.1 为什么需要源码验证

- **透明度**: 允许用户审计合约逻辑
- **信任**: 确保部署的字节码与源码匹配
- **交互**: 允许在区块浏览器上直接读写合约
- **调试**: 方便开发者追踪交易和状态

#### 1.2 使用 Hardhat 插件验证

**配置 Hardhat Verify**:
```javascript
// hardhat.config.js
require("@nomiclabs/hardhat-etherscan");

module.exports = {
  // ... 其他配置
  etherscan: {
    apiKey: {
      mainnet: process.env.ETHERSCAN_API_KEY,
      sepolia: process.env.ETHERSCAN_API_KEY,
      polygon: process.env.POLYGONSCAN_API_KEY,
      polygonMumbai: process.env.POLYGONSCAN_API_KEY,
      arbitrumOne: process.env.ARBISCAN_API_KEY,
      optimisticEthereum: process.env.OPTIMISTIC_ETHERSCAN_API_KEY,
    },
    customChains: [
      {
        network: "polygonMumbai",
        chainId: 80001,
        urls: {
          apiURL: "https://api-testnet.polygonscan.com/api",
          browserURL: "https://mumbai.polygonscan.com"
        }
      }
    ]
  }
};
```

**验证脚本**:
```javascript
// scripts/verify-contract.js
const hre = require("hardhat");

async function main() {
  const contractAddress = process.env.CONTRACT_ADDRESS;
  const network = hre.network.name;

  console.log(`Verifying contract at ${contractAddress} on ${network}...`);

  // 构造函数参数
  const args = [
    "MyToken",
    "MTK",
    hre.ethers.utils.parseEther("1000000")
  ];

  try {
    await hre.run("verify:verify", {
      address: contractAddress,
      constructorArguments: args,
      // 如果合约使用了库，需要指定库地址
      // libraries: {
      //   Utils: "0x..."
      // }
    });
    console.log("Verification successful!");
  } catch (error) {
    if (error.message.toLowerCase().includes("already verified")) {
      console.log("Contract already verified!");
    } else {
      console.error("Verification failed:", error);
    }
  }
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

#### 1.3 复杂合约验证

**验证代理合约**:
```javascript
// scripts/verify-proxy.js
const hre = require("hardhat");

async function verifyProxy(proxyAddress, implementationAddress, args) {
  console.log("Verifying Proxy and Implementation...");

  // 1. 验证实现合约
  console.log("Verifying implementation...");
  try {
    await hre.run("verify:verify", {
      address: implementationAddress,
      constructorArguments: [] // 实现合约通常在 initialize 中初始化
    });
  } catch (e) {
    console.log("Implementation verification info:", e.message);
  }

  // 2. 验证代理合约 (通常是标准合约，只需验证一次或 Etherscan 自动识别)
  // 如果是自定义代理，需要验证
  console.log("Verifying proxy...");
  try {
    await hre.run("verify:verify", {
      address: proxyAddress,
      constructorArguments: [
        implementationAddress,
        process.env.PROXY_ADMIN_ADDRESS,
        "0x" // 初始化数据
      ]
    });
  } catch (e) {
    console.log("Proxy verification info:", e.message);
  }
  
  console.log("Verification process completed.");
  console.log(`Check: https://${hre.network.name}.etherscan.io/address/${proxyAddress}#code`);
}
```

**多文件/多编译器版本验证**:
- 确保 `hardhat.config.js` 中的编译器设置与部署时完全一致
- 确保 Flatten 后的文件不包含重复的 SPDX 标识

### 2. 前端应用部署 (Web2 方式)

#### 2.1 准备生产构建

**环境变量配置**:
```bash
# .env.production
VITE_APP_CHAIN_ID=1
VITE_APP_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY
VITE_APP_CONTRACT_ADDRESS=0x...
VITE_APP_API_URL=https://api.yourapp.com
```

**构建脚本**:
```json
// package.json
{
  "scripts": {
    "build": "tsc && vite build",
    "preview": "vite preview"
  }
}
```

#### 2.2 Vercel 部署

**Vercel 配置文件 (`vercel.json`)**:
```json
{
  "framework": "vite",
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-XSS-Protection", "value": "1; mode=block" }
      ]
    }
  ]
}
```

**GitHub Actions 部署到 Vercel**:
```yaml
# .github/workflows/deploy-frontend.yaml
name: Deploy Frontend to Vercel

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Install Vercel CLI
        run: npm install --global vercel@latest
        
      - name: Pull Vercel Environment Information
        run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}
        
      - name: Build Project Artifacts
        run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}
        
      - name: Deploy Project Artifacts to Vercel
        run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
```

#### 2.3 Netlify 部署

**Netlify 配置文件 (`netlify.toml`)**:
```toml
[build]
  command = "npm run build"
  publish = "dist"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-XSS-Protection = "1; mode=block"
    Content-Security-Policy = "default-src 'self' https: wss: data: 'unsafe-inline' 'unsafe-eval'"
```

### 3. 去中心化前端部署 (Web3 方式)

#### 3.1 什么是去中心化前端

- 前端代码存储在 IPFS 或 Arweave 等去中心化网络上
- 通过 ENS (Ethereum Name Service) 或 IPNS 解析
- 抗审查，永不宕机

#### 3.2 部署到 IPFS (使用 Pinata)

**安装工具**:
```bash
npm install -g ipfs-deploy
# 或者使用专门的脚本
npm install @pinata/sdk
```

**上传脚本**:
```javascript
// scripts/deploy-ipfs.js
const pinataSDK = require('@pinata/sdk');
const path = require('path');
require('dotenv').config();

const pinata = new pinataSDK(
  process.env.PINATA_API_KEY,
  process.env.PINATA_SECRET_KEY
);

const buildPath = path.join(__dirname, '../dist');

async function deployToIPFS() {
  console.log(`Deploying build folder: ${buildPath} to IPFS...`);

  const options = {
    pinataMetadata: {
      name: 'MyDApp-Frontend-v1.0',
      keyvalues: {
        env: 'production',
        commit: process.env.GITHUB_SHA || 'local'
      }
    },
    pinataOptions: {
      cidVersion: 0
    }
  };

  try {
    const result = await pinata.pinFromFS(buildPath, options);
    console.log('Uploaded to IPFS successfully!');
    console.log('CID:', result.IpfsHash);
    console.log(`View at: https://gateway.pinata.cloud/ipfs/${result.IpfsHash}`);
    
    // 更新 DNSLink (可选)
    // await updateDNSLink(result.IpfsHash);
    
  } catch (error) {
    console.error('Error deploying to IPFS:', error);
    process.exit(1);
  }
}

deployToIPFS();
```

**GitHub Actions 自动部署到 IPFS**:
```yaml
# .github/workflows/deploy-ipfs.yaml
name: Deploy to IPFS

on:
  release:
    types: [published]

jobs:
  deploy-ipfs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'
          
      - run: npm ci
      - run: npm run build
      
      - name: Pin to IPFS via Pinata
        uses: aquiladev/ipfs-action@v1
        with:
          path: ./dist
          service: pinata
          pinataKey: ${{ secrets.PINATA_API_KEY }}
          pinataSecret: ${{ secrets.PINATA_SECRET_KEY }}
          pinName: 'MyDApp-${{ github.sha }}'
          
      - name: Update ENS ContentHash (Optional)
        uses: ./.github/actions/update-ens
        with:
          cid: ${{ steps.ipfs.outputs.cid }}
          privateKey: ${{ secrets.ENS_UPDATER_KEY }}
```

#### 3.3 部署到 Arweave

**使用 arweave-deploy**:
```bash
npm install -g arweave-deploy
arweave deploy-dir dist --key-file arweave-key.json
```

**使用 Bundlr (现 Irys) 进行高效上传**:
```javascript
// scripts/deploy-arweave.js
const Bundlr = require("@bundlr-network/client").default;
const fs = require("fs");
const path = require("path");
require("dotenv").config();

async function main() {
  const key = JSON.parse(fs.readFileSync("wallet.json").toString());
  const bundlr = new Bundlr("http://node1.bundlr.network", "arweave", key);
  
  const dirPath = "./dist";
  
  // 计算价格
  const size = await getDirectorySize(dirPath);
  const price = await bundlr.getPrice(size);
  
  console.log(`Upload size: ${size} bytes`);
  console.log(`Price: ${bundlr.utils.unitConverter(price).toString()} AR`);
  
  // 充值
  await bundlr.fund(price);
  
  // 上传文件夹
  const result = await bundlr.uploadFolder(dirPath, {
    indexFile: "index.html"
  });
  
  console.log(`Uploaded to Arweave!`);
  console.log(`Manifest URL: https://arweave.net/${result.id}`);
}

main();
```

### 4. 域名与 CDN 配置

#### 4.1 ENS 域名绑定

将 IPFS 内容哈希绑定到 ENS 域名，实现 `dapp.eth` 访问。

**脚本更新 ENS**:
```javascript
// scripts/update-ens.js
const ethers = require('ethers');
const namehash = require('eth-ens-namehash');
const contentHash = require('content-hash');

async function updateENSContentHash(ensName, ipfsCID) {
  const provider = new ethers.providers.JsonRpcProvider(process.env.RPC_URL);
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
  
  // 获取 ENS Resolver
  const registry = new ethers.Contract(
    '0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e', // ENS Registry Address
    ['function resolver(bytes32 node) view returns (address)'],
    provider
  );
  
  const node = namehash.hash(ensName);
  const resolverAddr = await registry.resolver(node);
  
  if (resolverAddr === ethers.constants.AddressZero) {
    throw new Error('Resolver not found for this ENS name');
  }
  
  const resolver = new ethers.Contract(
    resolverAddr,
    ['function setContenthash(bytes32 node, bytes calldata hash)'],
    wallet
  );
  
  // 编码 IPFS CID 为 contenthash
  const encodedHash = '0x' + contentHash.fromIpfs(ipfsCID);
  
  console.log(`Updating ${ensName} content hash to ${ipfsCID}...`);
  
  const tx = await resolver.setContenthash(node, encodedHash);
  await tx.wait();
  
  console.log(`ENS updated! Visit: https://${ensName}.limo`);
}
```

#### 4.2 配置 Fleek

Fleek 是 Web3 的 Vercel，自动从 GitHub 拉取代码，构建并部署到 IPFS，同时提供 CDN 加速和自定义域名。

1. 登录 [Fleek](https://fleek.co)
2. 连接 GitHub 仓库
3. 配置构建命令 (`npm run build`) 和输出目录 (`dist`)
4. Fleek 自动生成 IPFS CID 和 `*.on.fleek.co` 域名
5. 在设置中绑定 ENS 或传统 DNS 域名

### 5. 部署后检查清单

- [ ] **合约验证**: 所有合约已在 Etherscan/PolygonScan 上验证
- [ ] **前端可访问性**: 应用可通过 HTTPS URL 访问
- [ ] **网络连接**: 前端能正确连接到指定 RPC 节点
- [ ] **钱包连接**: MetaMask/WalletConnect 正常工作
- [ ] **资源加载**: 所有图片、字体加载正常 (CORS 配置正确)
- [ ] **路由**: 刷新页面不出现 404 (SPA 重写规则配置正确)
- [ ] **去中心化访问**: 可通过 IPFS Gateway 访问 (如果已部署)
- [ ] **性能**: Lighthouse 评分合格

## 实践练习

### 练习1: 自动化合约验证

编写一个 Hardhat 任务，实现部署后自动验证合约源码。

```javascript
// 提示：使用 task() 定义新任务，在 run("deploy") 后自动调用 run("verify")
```

### 练习2: IPFS 部署流水线

创建一个 GitHub Action，当代码推送到 `main` 分支时：
1. 运行测试
2. 构建前端
3. 上传到 Pinata
4. 将返回的 CID 更新到 README.md 中

### 练习3: ENS 更新脚本

编写一个脚本，读取最新的 IPFS CID，并更新你的 ENS 测试域名的 ContentHash。

## 学习检查清单

- [ ] 理解合约源码验证的重要性
- [ ] 能够配置和使用 Hardhat Etherscan 插件
- [ ] 掌握 Web2 平台 (Vercel/Netlify) 的 SPA 配置
- [ ] 理解 IPFS/Arweave 部署原理
- [ ] 能够使用脚本上传文件到 IPFS
- [ ] 了解 ENS ContentHash 的工作机制
- [ ] 掌握 Fleek 的基本使用

## 扩展阅读

- [Hardhat Etherscan Plugin](https://hardhat.org/plugins/nomiclabs-hardhat-etherscan)
- [Pinata Docs](https://docs.pinata.cloud/)
- [ENS Documentation - ContentHash](https://docs.ens.domains/contract-api-reference/publicresolver#setcontenthash)
- [Fleek Documentation](https://docs.fleek.co/)
- [Vite Deployment Guide](https://vitejs.dev/guide/static-deploy.html)

## 下节预告

下一节我们将学习**区块链节点服务与基础设施监控**,包括:
- 自建节点 vs RPC 服务商 (Infura/Alchemy)
- Graph Node 索引服务搭建
- 链上数据监控与报警
- 交易失败分析与调试
- 系统状态页与 SLA 监控
