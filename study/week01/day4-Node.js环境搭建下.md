# Week 1 - Day 4: Node.js环境搭建（下）

**学习日期**: ___________
**预计用时**: 4-5小时  
**难度等级**: ⭐⭐ (入门)

## 📚 今日学习目标

- ✅ 安装并配置Git版本控制
- ✅ 安装并配置VSCode开发环境
- ✅ 深入理解Node.js异步编程
- ✅ 学习Node.js模块系统详解
- ✅ 创建第一个完整Node.js项目
- ✅ 掌握调试技巧

---

## 🔧 Part 1: Git安装与配置 (1小时)

### 1.1 什么是Git？

**Git简介：**
```
Git = 分布式版本控制系统

作用：
✓ 追踪代码变更历史
✓ 多人协作开发
✓ 回退到任意版本
✓ 分支管理
✓ 备份代码

为什么Web3开发需要Git？
- 管理智能合约代码
- 与GitHub集成
- 团队协作
- 开源项目贡献
```

### 1.2 安装Git

**Windows安装步骤：**

```powershell
# 方法1: 下载安装（推荐）
1. 访问 https://git-scm.com/download/win
2. 下载64-bit Git for Windows Setup
3. 运行安装程序

# 方法2: 使用包管理器
winget install Git.Git
```

**安装向导配置：**
```
步骤1: 选择组件
[✓] Windows Explorer integration
[✓] Git Bash Here
[✓] Git GUI Here
[✓] Associate .git* configuration files
[✓] Associate .sh files to be run with Bash

步骤2: 选择默认编辑器
推荐: Visual Studio Code (需先安装VSCode)
或: Notepad++

步骤3: PATH环境设置
选择: Git from the command line and also from 3rd-party software
(推荐，让PowerShell也能用git)

步骤4: SSH可执行文件
选择: Use bundled OpenSSH

步骤5: HTTPS传输后端
选择: Use the OpenSSL library

步骤6: 行尾转换
选择: Checkout Windows-style, commit Unix-style line endings

步骤7: 终端模拟器
选择: Use Windows' default console window

步骤8: git pull行为
选择: Default (fast-forward or merge)

步骤9: 凭据管理器
选择: Git Credential Manager

步骤10: 额外选项
[✓] Enable file system caching
[✓] Enable symbolic links

完成安装
```

### 1.3 验证Git安装

```powershell
# 检查版本
git --version
# 输出: git version 2.43.0.windows.1

# 查看git配置
git config --list

# 测试git help
git help
```

### 1.4 Git初始配置

**配置用户信息：**

```powershell
# 设置用户名
git config --global user.name "Your Name"

# 设置邮箱
git config --global user.email "your.email@example.com"

# 设置默认分支名
git config --global init.defaultBranch main

# 设置默认编辑器
git config --global core.editor "code --wait"

# 查看配置
git config --global --list
```

**配置Git别名（可选）：**

```powershell
# 常用别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'

# 现在可以使用
git st  # 等同于 git status
```

### 1.5 Git基础命令

**创建仓库：**

```powershell
# 创建项目目录
cd E:\Seadragon\WEB3
mkdir git-practice
cd git-practice

# 初始化Git仓库
git init
# 输出: Initialized empty Git repository in E:/Seadragon/WEB3/git-practice/.git/

# 查看状态
git status
```

**基本工作流：**

```powershell
# 1. 创建文件
echo "# Git Practice" > README.md

# 2. 查看状态
git status
# 输出: Untracked files: README.md

# 3. 添加到暂存区
git add README.md
# 或添加所有文件
git add .

# 4. 提交
git commit -m "Initial commit: Add README"

# 5. 查看历史
git log
git log --oneline
```

**常用命令速查：**

```powershell
# 查看状态
git status

# 查看差异
git diff              # 工作区vs暂存区
git diff --staged     # 暂存区vs仓库

# 撤销修改
git checkout -- file  # 撤销工作区修改
git reset HEAD file   # 撤销暂存

# 分支操作
git branch            # 查看分支
git branch dev        # 创建分支
git checkout dev      # 切换分支
git checkout -b dev   # 创建并切换

# 查看历史
git log
git log --oneline --graph --all
```

### 1.6 .gitignore文件

**创建.gitignore：**

```gitignore
# .gitignore for Node.js projects

# Dependencies
node_modules/
package-lock.json
yarn.lock
pnpm-lock.yaml

# Environment variables
.env
.env.local
.env.*.local

# Build output
dist/
build/
out/
*.log

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Hardhat files
cache/
artifacts/
typechain/
typechain-types/

# Coverage
coverage/
.nyc_output/

# Temp files
*.tmp
*.temp
```

**实践：创建Git仓库**

```powershell
cd E:\Seadragon\WEB3\web3-learning
git init
echo "node_modules/" > .gitignore
git add .
git commit -m "Initial commit"
```

---

## 💻 Part 2: VSCode安装与配置 (1小时)

### 2.1 安装VSCode

**下载安装：**

```powershell
# 方法1: 官网下载
1. 访问 https://code.visualstudio.com/
2. 下载User Installer (64-bit)
3. 运行安装程序

# 方法2: 使用包管理器
winget install Microsoft.VisualStudioCode
```

**安装选项：**
```
[✓] Add "Open with Code" action to Windows Explorer file context menu
[✓] Add "Open with Code" action to Windows Explorer directory context menu
[✓] Register Code as an editor for supported file types
[✓] Add to PATH
```

### 2.2 必装插件

**打开VSCode，按Ctrl+Shift+X打开扩展面板：**

```
必装插件列表：

1. Chinese (Simplified) Language Pack
   - 中文界面（可选）

2. ESLint
   - JavaScript代码检查

3. Prettier - Code formatter
   - 代码格式化

4. GitLens
   - Git增强工具

5. Path Intellisense
   - 路径自动补全

6. Auto Rename Tag
   - HTML标签同步重命名

7. Solidity (by Juan Blanco)
   - Solidity语言支持

8. Hardhat for VSCode
   - Hardhat集成

9. Ethereum Security Bundle
   - 智能合约安全检查
```

**安装插件命令：**

```powershell
# 通过命令行安装
code --install-extension ms-ceintl.vscode-language-pack-zh-hans
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension eamodio.gitlens
code --install-extension christian-kohler.path-intellisense
code --install-extension formulahendry.auto-rename-tag
code --install-extension juanblanco.solidity
code --install-extension nomicfoundation.hardhat-solidity
```

### 2.3 VSCode配置

**打开设置（Ctrl+,）或创建settings.json：**

```json
{
  // 编辑器
  "editor.fontSize": 14,
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  
  // 文件
  "files.autoSave": "afterDelay",
  "files.autoSaveDelay": 1000,
  "files.encoding": "utf8",
  "files.eol": "\n",
  
  // 终端
  "terminal.integrated.defaultProfile.windows": "PowerShell",
  "terminal.integrated.fontSize": 13,
  
  // Solidity
  "solidity.compileUsingRemoteVersion": "v0.8.20",
  "solidity.formatter": "prettier",
  
  // 其他
  "workbench.startupEditor": "none",
  "explorer.confirmDelete": false,
  "git.autofetch": true
}
```

### 2.4 VSCode快捷键

**必知快捷键：**

```
文件操作：
Ctrl+N          新建文件
Ctrl+O          打开文件
Ctrl+S          保存
Ctrl+Shift+S    另存为
Ctrl+W          关闭当前文件
Ctrl+K Ctrl+W   关闭所有文件

编辑：
Ctrl+X          剪切行
Ctrl+C          复制行
Ctrl+V          粘贴
Ctrl+Z          撤销
Ctrl+Shift+Z    重做
Ctrl+D          选择下一个相同内容
Ctrl+Shift+L    选择所有相同内容
Alt+↑/↓         移动行
Shift+Alt+↑/↓   复制行

查找替换：
Ctrl+F          查找
Ctrl+H          替换
Ctrl+Shift+F    全局查找

代码：
Ctrl+/          注释/取消注释
Shift+Alt+F     格式化代码
F2              重命名
F12             跳转到定义

视图：
Ctrl+B          切换侧边栏
Ctrl+J          切换终端
Ctrl+`          打开终端
Ctrl+Shift+E    资源管理器
Ctrl+Shift+G    Git视图

多光标：
Alt+Click       添加光标
Ctrl+Alt+↑/↓    上下添加光标
```

### 2.5 创建VSCode工作区

**项目配置文件：**

```json
// .vscode/settings.json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "[solidity]": {
    "editor.defaultFormatter": "JuanBlanco.solidity"
  },
  "solidity.compileUsingRemoteVersion": "v0.8.20",
  "solidity.remappings": [
    "@openzeppelin/=node_modules/@openzeppelin/"
  ]
}
```

```json
// .vscode/extensions.json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "juanblanco.solidity",
    "nomicfoundation.hardhat-solidity"
  ]
}
```

---

## 🔄 Part 3: Node.js异步编程深入 (1.5小时)

### 3.1 理解事件循环

**Event Loop工作原理：**

```javascript
/*
JavaScript执行模型：

Call Stack (调用栈)
     ↓
  执行同步代码
     ↓
Callback Queue (回调队列)
     ↓
  执行异步回调
*/

// 示例
console.log("1");

setTimeout(() => {
  console.log("2");
}, 0);

console.log("3");

// 输出顺序: 1, 3, 2
// 原因：setTimeout是异步的，即使delay为0
```

**宏任务vs微任务：**

```javascript
// 微任务 (Microtask)
// - Promise.then/catch/finally
// - process.nextTick
// - MutationObserver

// 宏任务 (Macrotask)
// - setTimeout
// - setInterval
// - setImmediate
// - I/O操作

console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve().then(() => console.log("3"));

console.log("4");

// 输出: 1, 4, 3, 2
// 微任务优先于宏任务
```

### 3.2 Promise深入

**创建Promise：**

```javascript
// 基础Promise
function delay(ms) {
  return new Promise(resolve => {
    setTimeout(resolve, ms);
  });
}

// 使用
delay(2000).then(() => {
  console.log("2秒后执行");
});

// 带返回值的Promise
function fetchData(url) {
  return new Promise((resolve, reject) => {
    // 模拟网络请求
    setTimeout(() => {
      if (url) {
        resolve({ data: "some data" });
      } else {
        reject(new Error("URL is required"));
      }
    }, 1000);
  });
}
```

**Promise链式调用：**

```javascript
fetchData("https://api.example.com")
  .then(response => {
    console.log("Step 1:", response);
    return response.data;
  })
  .then(data => {
    console.log("Step 2:", data);
    return data.toUpperCase();
  })
  .then(upperData => {
    console.log("Step 3:", upperData);
  })
  .catch(error => {
    console.error("Error:", error);
  })
  .finally(() => {
    console.log("Cleanup");
  });
```

**Promise并行操作：**

```javascript
// Promise.all - 全部成功才resolve
const promises = [
  fetchData("url1"),
  fetchData("url2"),
  fetchData("url3")
];

Promise.all(promises)
  .then(results => {
    console.log("All success:", results);
  })
  .catch(error => {
    console.error("One failed:", error);
  });

// Promise.allSettled - 全部完成(成功或失败)
Promise.allSettled(promises)
  .then(results => {
    results.forEach((result, index) => {
      if (result.status === "fulfilled") {
        console.log(`Promise ${index} fulfilled:`, result.value);
      } else {
        console.log(`Promise ${index} rejected:`, result.reason);
      }
    });
  });

// Promise.race - 第一个完成的Promise
Promise.race(promises)
  .then(result => {
    console.log("First done:", result);
  });

// Promise.any - 第一个成功的Promise
Promise.any(promises)
  .then(result => {
    console.log("First success:", result);
  })
  .catch(error => {
    console.error("All failed:", error);
  });
```

### 3.3 async/await深入

**错误处理：**

```javascript
// 基础用法
async function getData() {
  try {
    const response = await fetch("url");
    const data = await response.json();
    return data;
  } catch (error) {
    console.error("Error:", error);
    throw error;  // 重新抛出
  }
}

// 多个try-catch
async function multipleOperations() {
  let result1, result2;
  
  try {
    result1 = await operation1();
  } catch (error) {
    console.error("Operation 1 failed:", error);
    result1 = defaultValue1;
  }
  
  try {
    result2 = await operation2(result1);
  } catch (error) {
    console.error("Operation 2 failed:", error);
    result2 = defaultValue2;
  }
  
  return { result1, result2 };
}
```

**并行执行：**

```javascript
// 顺序执行（慢）
async function sequential() {
  const result1 = await fetchData("url1");  // 等1秒
  const result2 = await fetchData("url2");  // 等1秒
  const result3 = await fetchData("url3");  // 等1秒
  return [result1, result2, result3];
}
// 总耗时: 3秒

// 并行执行（快）
async function parallel() {
  const [result1, result2, result3] = await Promise.all([
    fetchData("url1"),
    fetchData("url2"),
    fetchData("url3")
  ]);
  return [result1, result2, result3];
}
// 总耗时: 1秒

// Web3中的应用
async function getMultipleBalances(addresses) {
  const balances = await Promise.all(
    addresses.map(addr => provider.getBalance(addr))
  );
  return balances;
}
```

**处理循环中的异步：**

```javascript
// ❌ 错误：forEach不支持async/await
async function wrong(items) {
  items.forEach(async (item) => {
    await processItem(item);
  });
}

// ✅ 正确：使用for...of
async function correct(items) {
  for (const item of items) {
    await processItem(item);
  }
}

// ✅ 正确：并行处理
async function parallel(items) {
  await Promise.all(
    items.map(item => processItem(item))
  );
}

// 实际例子
async function deployContracts(contracts) {
  const deployed = [];
  
  for (const contract of contracts) {
    console.log(`Deploying ${contract.name}...`);
    const instance = await contract.deploy();
    await instance.deployed();
    deployed.push(instance);
  }
  
  return deployed;
}
```

---

## 📦 Part 4: Node.js模块系统详解 (1小时)

### 4.1 CommonJS模块

**导出方式：**

```javascript
// math.js

// 方式1: exports
exports.add = (a, b) => a + b;
exports.multiply = (a, b) => a * b;

// 方式2: module.exports
module.exports = {
  add: (a, b) => a + b,
  multiply: (a, b) => a * b
};

// 方式3: 默认导出单个函数
module.exports = function subtract(a, b) {
  return a - b;
};

// ⚠️ 注意：不能混用
// 错误示例：
exports.add = () => {};
module.exports = {};  // 这会覆盖前面的exports
```

**导入方式：**

```javascript
// 导入整个模块
const math = require('./math.js');
console.log(math.add(1, 2));

// 解构导入
const { add, multiply } = require('./math.js');
console.log(add(1, 2));

// 导入默认导出
const subtract = require('./math.js');
console.log(subtract(5, 2));

// 导入核心模块
const fs = require('fs');
const path = require('path');

// 导入npm包
const ethers = require('ethers');
```

### 4.2 ES Modules

**在Node.js中启用ESM：**

```json
// package.json
{
  "type": "module"
}
```

**导出方式：**

```javascript
// math.mjs (或.js with "type": "module")

// 命名导出
export const add = (a, b) => a + b;
export const multiply = (a, b) => a * b;

// 或
const add = (a, b) => a + b;
const multiply = (a, b) => a * b;
export { add, multiply };

// 默认导出
export default function subtract(a, b) {
  return a - b;
}

// 混合使用
export const PI = 3.14159;
export default class Calculator {
  add(a, b) { return a + b; }
}
```

**导入方式：**

```javascript
// 导入命名导出
import { add, multiply } from './math.mjs';

// 导入默认导出
import subtract from './math.mjs';

// 导入全部
import * as math from './math.mjs';
console.log(math.add(1, 2));

// 混合导入
import Calculator, { PI } from './math.mjs';

// 动态导入
const math = await import('./math.mjs');

// 重命名
import { add as addition } from './math.mjs';
```

### 4.3 模块解析规则

**路径解析：**

```javascript
// 1. 相对路径
require('./module.js');      // 当前目录
require('../module.js');     // 父目录
require('./dir/module.js');  // 子目录

// 2. 绝对路径（不推荐）
require('C:/path/to/module.js');

// 3. 核心模块
require('fs');
require('path');
require('http');

// 4. node_modules查找
require('express');
// 查找顺序：
// ./node_modules/express
// ../node_modules/express
// ../../node_modules/express
// ...直到根目录

// 5. 省略扩展名
require('./module');  // 自动查找：
// module.js
// module.json
// module.node
```

### 4.4 创建自己的npm包

**package.json配置：**

```json
{
  "name": "my-web3-utils",
  "version": "1.0.0",
  "main": "index.js",          // CommonJS入口
  "module": "index.mjs",       // ESM入口
  "exports": {
    ".": {
      "import": "./index.mjs",
      "require": "./index.js"
    },
    "./utils": {
      "import": "./utils/index.mjs",
      "require": "./utils/index.js"
    }
  },
  "files": [
    "index.js",
    "index.mjs",
    "utils/"
  ]
}
```

**创建包结构：**

```
my-web3-utils/
├── package.json
├── index.js          # CommonJS入口
├── index.mjs         # ESM入口
├── utils/
│   ├── format.js
│   └── validate.js
├── README.md
└── .gitignore
```

---

## 🎯 Part 5: 实践项目 (30分钟)

### 5.1 创建完整Node.js项目

**项目结构：**

```
web3-utils/
├── src/
│   ├── utils/
│   │   ├── format.js
│   │   └── validate.js
│   └── index.js
├── test/
│   └── utils.test.js
├── .gitignore
├── package.json
└── README.md
```

**代码实现：**

```javascript
// src/utils/format.js
/**
 * 格式化以太坊地址
 * @param {string} address - 完整地址
 * @returns {string} 格式化后的地址
 */
function formatAddress(address) {
  if (!address || address.length < 10) {
    return address;
  }
  return `${address.slice(0, 6)}...${address.slice(-4)}`;
}

/**
 * Wei转ETH
 * @param {string|number} wei - Wei数量
 * @returns {string} ETH数量
 */
function weiToEth(wei) {
  return (Number(wei) / 1e18).toFixed(4);
}

module.exports = {
  formatAddress,
  weiToEth
};
```

```javascript
// src/utils/validate.js
/**
 * 验证以太坊地址
 * @param {string} address - 地址
 * @returns {boolean} 是否有效
 */
function isValidAddress(address) {
  return /^0x[a-fA-F0-9]{40}$/.test(address);
}

/**
 * 验证交易哈希
 * @param {string} hash - 交易哈希
 * @returns {boolean} 是否有效
 */
function isValidTxHash(hash) {
  return /^0x[a-fA-F0-9]{64}$/.test(hash);
}

module.exports = {
  isValidAddress,
  isValidTxHash
};
```

```javascript
// src/index.js
const format = require('./utils/format');
const validate = require('./utils/validate');

module.exports = {
  ...format,
  ...validate
};
```

```json
// package.json
{
  "name": "web3-utils",
  "version": "1.0.0",
  "description": "Web3工具函数库",
  "main": "src/index.js",
  "scripts": {
    "test": "node test/utils.test.js"
  },
  "keywords": ["web3", "ethereum", "utils"],
  "author": "Your Name",
  "license": "MIT"
}
```

**测试代码：**

```javascript
// test/utils.test.js
const { formatAddress, weiToEth, isValidAddress } = require('../src/index');

// 测试formatAddress
console.log("Testing formatAddress...");
const address = "0x742d35Cc6634C0532925a3b844E291e6A7E834";
const formatted = formatAddress(address);
console.log(`Result: ${formatted}`);
console.assert(formatted === "0x742d...E834", "❌ formatAddress failed");
console.log("✅ formatAddress passed");

// 测试weiToEth
console.log("\nTesting weiToEth...");
const wei = "1000000000000000000";
const eth = weiToEth(wei);
console.log(`Result: ${eth} ETH`);
console.assert(eth === "1.0000", "❌ weiToEth failed");
console.log("✅ weiToEth passed");

// 测试isValidAddress
console.log("\nTesting isValidAddress...");
console.assert(isValidAddress("0x742d35Cc6634C0532925a3b844E291e6A7E834") === true, "❌ Valid address check failed");
console.assert(isValidAddress("invalid") === false, "❌ Invalid address check failed");
console.log("✅ isValidAddress passed");

console.log("\n🎉 All tests passed!");
```

**运行测试：**

```powershell
npm test
```

---

## 📝 今日作业

### 作业1: Git实践

```markdown
# Git版本控制实践

## 1. 创建仓库
[ ] 初始化web3-learning仓库
[ ] 创建.gitignore文件
[ ] 第一次提交

记录：
初始提交哈希: _________

## 2. 分支操作
[ ] 创建dev分支
[ ] 在dev分支添加新文件
[ ] 切换回main分支
[ ] 合并dev分支

命令记录：
```bash
# 粘贴你使用的git命令


```

## 3. 查看历史
[ ] 使用git log查看提交历史
[ ] 使用git log --oneline --graph

提交历史截图或记录：
```

### 作业2: VSCode配置

```markdown
# VSCode开发环境配置

## 1. 插件安装
已安装的插件：
[ ] ESLint
[ ] Prettier
[ ] GitLens
[ ] Solidity
[ ] 其他: _________

## 2. 快捷键练习
完成以下操作并记录使用的快捷键：
- 格式化代码: _________
- 多行注释: _________
- 跳转定义: _________
- 全局搜索: _________

## 3. 工作区配置
[ ] 创建.vscode/settings.json
[ ] 配置自动保存
[ ] 配置格式化

粘贴你的settings.json:
```json


```
```

### 作业3: 异步编程练习

```javascript
// 完成以下异步编程练习

// 1. 实现延迟函数
async function delay(ms) {
  // TODO: 返回一个在ms毫秒后resolve的Promise
}

// 测试
await delay(2000);
console.log("2秒后执行");

// 2. 实现重试机制
async function retry(fn, maxAttempts = 3) {
  // TODO: 如果fn失败，最多重试maxAttempts次
  // 成功则返回结果，全部失败则抛出最后的错误
}

// 3. 实现超时控制
async function timeout(promise, ms) {
  // TODO: 如果promise在ms毫秒内没有resolve，则reject
}

// 4. 批量获取数据（限制并发）
async function fetchBatch(urls, concurrency = 3) {
  // TODO: 同时最多发起concurrency个请求
}

// 5. Web3应用：批量获取余额
async function getBalances(addresses, provider) {
  // TODO: 并行获取多个地址的余额
  // 使用Promise.all
}
```

### 作业4: 模块化项目

```markdown
# 创建模块化项目

## 任务
创建一个eth-tools包，包含以下功能：

1. 地址工具 (address.js)
   - formatAddress(address): 格式化显示
   - checksumAddress(address): 校验和格式
   - isValidAddress(address): 验证地址

2. 单位转换 (units.js)
   - weiToEth(wei): Wei转ETH
   - ethToWei(eth): ETH转Wei  
   - gweiToEth(gwei): Gwei转ETH

3. 交易工具 (transaction.js)
   - formatTxHash(hash): 格式化交易哈希
   - calculateGasCost(gasUsed, gasPrice): 计算Gas费用

要求：
- 使用CommonJS模块系统
- 每个模块独立文件
- 统一在index.js导出
- 编写测试文件
- 创建README.md文档

提交：
仓库地址: _________
```

---

## ✅ 今日检查清单

### 环境配置
- [ ] Git安装并配置
- [ ] VSCode安装并配置必要插件
- [ ] 创建.gitignore文件
- [ ] 配置Git用户信息

### 知识掌握
- [ ] 理解Git基本概念和工作流
- [ ] 掌握VSCode快捷键
- [ ] 深入理解async/await
- [ ] 理解模块系统(CommonJS vs ESM)
- [ ] 掌握Promise高级用法

### 实践操作
- [ ] 创建Git仓库并提交代码
- [ ] 在VSCode中编写和调试代码
- [ ] 创建完整的Node.js模块化项目
- [ ] 编写并运行测试

---

## 🆘 常见问题FAQ

### Q1: Git命令找不到？
```powershell
# 重启PowerShell
# 或检查PATH
where.exe git

# 手动添加到PATH
# Git默认路径: C:\Program Files\Git\cmd
```

### Q2: VSCode无法格式化代码？
```
1. 安装Prettier插件
2. 设置为默认格式化程序
3. 右键 → 格式化文档
4. 或使用快捷键 Shift+Alt+F
```

### Q3: 异步函数中无法使用await？
```javascript
// ❌ 错误
function wrong() {
  await doSomething();  // SyntaxError
}

// ✅ 正确
async function correct() {
  await doSomething();
}

// ✅ 顶层await (Node.js 14.8+, ESM)
// 在.mjs文件或"type": "module"中
await doSomething();
```

### Q4: 模块导入出错？
```javascript
// 检查：
// 1. 路径是否正确（./或../）
// 2. 文件扩展名
// 3. CommonJS vs ESM
// 4. package.json中的"type"字段

// CommonJS
const mod = require('./module');

// ESM (需要"type": "module")
import mod from './module.mjs';
```

---

## 📅 明日预告: Hardhat框架入门

明天我们将：
- 安装Hardhat开发框架
- 理解Hardhat项目结构
- 编写第一个智能合约
- 部署合约到本地网络
- 使用Hardhat Console交互

**今晚准备**：
- 确保Node.js环境正常
- 完成今天的Git和VSCode配置
- 复习JavaScript异步编程

---

## ✍️ 我的学习记录

**完成日期**: ___________
**实际耗时**: _____ 小时

### ✅ 完成情况
- [ ] Git安装配置
- [ ] VSCode配置
- [ ] 异步编程深入学习
- [ ] 模块系统学习
- [ ] 实践项目完成

### 💡 今日收获
1. 最有价值的知识:
2. 最复杂的部分:
3. 成功的项目:

### 📝 环境配置
- Git版本: _________
- VSCode版本: _________
- 已安装插件数: _________

### 🤔 疑问与思考
- 问题1:
- 问题2:

**🎉 完成Day 4！明天开始Hardhat！**
