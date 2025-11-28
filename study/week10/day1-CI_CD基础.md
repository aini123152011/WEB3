# Week 10 - Day 1: CI/CD基础

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 理解DevOps与CI/CD
- ✅ 掌握GitHub Actions基础
- ✅ 编写自动化测试工作流
- ✅ 实现代码质量检查
- ✅ 自动化发布流程

---

## Part 1: CI/CD 基础概念 (1.5小时)

### 1.1 什么是CI/CD？

- **CI (Continuous Integration)**: 持续集成。开发者频繁将代码合并到主分支，每次合并触发自动构建和测试。
- **CD (Continuous Delivery/Deployment)**: 持续交付/部署。自动将代码部署到测试环境或生产环境。

### 1.2 Web3 CI/CD 特点

与Web2不同，Web3 CI/CD面临特殊挑战：
- **不可变性**: 合约一旦部署无法修改，测试必须非常严格。
- **私钥管理**: 部署需要私钥，必须高度安全。
- **网络依赖**: 需要连接RPC节点。
- **Gas成本**: 部署和测试需要消耗真实资产（即使是测试币）。

---

## Part 2: GitHub Actions 入门 (2小时)

### 2.1 Workflow 文件结构

```yaml
# .github/workflows/basics.yml
name: Basic Workflow

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run script
        run: echo "Hello World"
```

### 2.2 环境变量与Secrets

在GitHub仓库设置中配置Secrets (Settings -> Secrets and variables -> Actions)。

```yaml
env:
  RPC_URL: ${{ secrets.RPC_URL }}
  PRIVATE_KEY: ${{ secrets.PRIVATE_KEY }}
```

---

## Part 3: 自动化测试流程 (2小时)

### 3.1 智能合约测试

```yaml
# .github/workflows/contracts.yml
name: Smart Contracts CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Compile contracts
        run: npx hardhat compile
        
      - name: Run unit tests
        run: npx hardhat test
        env:
          REPORT_GAS: true
          
      - name: Run coverage
        run: npx hardhat coverage
```

### 3.2 前端测试

```yaml
# .github/workflows/frontend.yml
name: Frontend CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./frontend
        
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Lint check
        run: npm run lint
        
      - name: Type check
        run: npm run type-check
        
      - name: Run tests
        run: npm run test
```

---

## Part 4: 代码质量检查 (1小时)

### 4.1 Solidity Linting

使用 `solhint` 检查代码风格。

```bash
npm install -D solhint
npx solhint --init
```

配置 `.solhint.json`:
```json
{
  "extends": "solhint:recommended",
  "rules": {
    "compiler-version": ["error", "^0.8.0"],
    "func-visibility": ["warn", { "ignoreConstructors": true }]
  }
}
```

### 4.2 Slither 静态分析

集成Slither进行安全性检查。

```yaml
# .github/workflows/slither.yml
name: Slither Analysis

on: [push]

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Slither
        uses: crytic/slither-action@v0.3.0
        with:
          target: 'contracts/'
          slither-args: '--exclude-optimization --exclude-informational'
```

---

## 📝 今日作业

### 作业1: 配置Hardhat CI

为之前的Hardhat项目添加GitHub Actions配置：
1. 自动编译合约
2. 运行所有测试用例
3. 生成Gas报告
4. 检查代码覆盖率（要求 > 80%）

### 作业2: 集成Slither

1. 在本地安装并运行Slither
2. 修复发现的所有High/Medium风险
3. 添加GitHub Action自动运行Slither

### 作业3: 自动化发布

编写一个Workflow：
1. 当推送到 `release` 分支时触发
2. 自动生成新的版本号 (Semantic Versioning)
3. 创建GitHub Release
4. 将合约ABI发布到NPM包

---

## ✅ 检查清单

- [ ] 理解YAML语法
- [ ] 成功运行第一个GitHub Action
- [ ] 配置Secrets保护私钥
- [ ] 实现自动化测试流水线
- [ ] 集成静态分析工具

---

## 📅 明日预告

明天学习自动化部署：
- 脚本化部署流程
- 多环境管理 (Dev/Staging/Prod)
- 验证合约源码
- 前端自动化部署到IPFS/Vercel

**🎉 完成Day 1！你的代码质量有了自动保障！**
