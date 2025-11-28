# Week 1 - Day 3: Node.js环境搭建（上）

**学习日期**: ___________
**预计用时**: 4-5小时
**难度等级**: ⭐⭐ (入门)

## 📚 今日学习目标

- ✅ 安装并配置Node.js开发环境
- ✅ 理解npm/pnpm包管理器
- ✅ 掌握JavaScript ES6+基础语法
- ✅ 学会使用Git进行版本控制
- ✅ 配置VSCode开发环境
- ✅ 创建第一个Node.js项目

---

## 🔧 Part 1: Node.js安装与配置 (1.5小时)

### 1.1 什么是Node.js？

**Node.js简介：**
```
Node.js = JavaScript运行时环境

传统：
JavaScript只能在浏览器中运行

Node.js：
JavaScript可以在服务器端运行

特点：
✓ 基于Chrome V8引擎
✓ 事件驱动、非阻塞I/O
✓ 单线程但高并发
✓ 丰富的npm生态
```

**为什么Web3开发需要Node.js？**
```
1. 运行开发工具
   - Hardhat
   - Truffle
   - Foundry (部分工具)

2. 编写测试脚本
   - JavaScript/TypeScript测试
   - 自动化测试

3. 部署脚本
   - 合约部署自动化
   - 配置管理

4. 前端开发
   - React/Vue DApp
   - Web3.js/Ethers.js集成
```

### 1.2 下载Node.js

**Windows下载步骤：**

```powershell
# 方法1: 官网下载（推荐）

1. 访问 https://nodejs.org/
2. 下载LTS版本（长期支持版）
   - 当前推荐: v20.x LTS
   - 文件名: node-v20.x.x-x64.msi

3. 不要下载Current版本（除非你知道自己在做什么）
```

**版本选择建议：**
```
LTS (Long Term Support) - 推荐 ✅
- 稳定可靠
- 长期维护
- 适合生产环境
- 兼容性好

Current - 不推荐 ❌
- 最新特性
- 可能有bug
- 短期支持
- 适合实验
```

### 1.3 安装Node.js

**详细安装步骤：**

```
步骤1: 运行安装程序
双击 node-v20.x.x-x64.msi

步骤2: 安装向导
┌────────────────────────────────┐
│ Node.js Setup                  │
├────────────────────────────────┤
│ Welcome to the Node.js Setup   │
│ Wizard                         │
│                                │
│ [Next >]                       │
└────────────────────────────────┘
点击 Next

步骤3: 接受许可协议
[✓] I accept the terms in the License Agreement
点击 Next

步骤4: 选择安装路径
默认: C:\Program Files\nodejs\
建议：保持默认
点击 Next

步骤5: 自定义安装（重要！）
确保以下项都被选中：
[✓] Node.js runtime
[✓] npm package manager
[✓] Online documentation shortcuts
[✓] Add to PATH  ← 非常重要！

点击 Next

步骤6: 自动安装必要工具（可选但推荐）
[✓] Automatically install the necessary tools
    (Python, Visual Studio Build Tools)

这会安装：
- Python 3.x
- Visual Studio Build Tools
- Windows SDK

⚠️ 注意：这个过程可能需要10-20分钟
如果网络不好，可以取消勾选，后续手动安装

点击 Next

步骤7: 开始安装
点击 Install
等待安装完成（2-5分钟）

步骤8: 完成安装
[✓] Launch Node.js
点击 Finish
```

### 1.4 验证安装

**打开PowerShell验证：**

```powershell
# 检查Node.js版本
node --version
# 期望输出: v20.11.0 (或类似版本号)

# 检查npm版本
npm --version
# 期望输出: 10.2.4 (或类似版本号)

# 检查Node.js是否在PATH中
where.exe node
# 期望输出: C:\Program Files\nodejs\node.exe

# 测试Node.js REPL
node
> console.log("Hello Web3!")
# 输出: Hello Web3!
> .exit
# 退出REPL
```

**如果命令无法识别：**
```powershell
# 原因：PATH环境变量未设置

# 解决方法1: 重启PowerShell
关闭并重新打开PowerShell

# 解决方法2: 手动添加到PATH
# 1. 右键"此电脑" → 属性
# 2. 高级系统设置
# 3. 环境变量
# 4. 系统变量中找到Path
# 5. 编辑 → 新建 → 添加：
C:\Program Files\nodejs\

# 解决方法3: 重新安装Node.js
# 确保勾选 "Add to PATH"
```

### 1.5 Node.js基础命令

**常用命令：**

```powershell
# 1. 运行JavaScript文件
node script.js

# 2. 进入交互模式(REPL)
node
> 1 + 1
2
> .exit

# 3. 查看帮助
node --help

# 4. 检查语法（不执行）
node --check script.js

# 5. 执行代码字符串
node -e "console.log('Hello')"
```

**实践：创建第一个Node.js程序**

```powershell
# 1. 创建测试文件
cd E:\Seadragon\WEB3
New-Item -ItemType File -Name "hello.js"

# 2. 编辑文件（用记事本或VSCode）
notepad hello.js
```

```javascript
// hello.js 内容
console.log("Hello, Web3!");
console.log("Node.js版本:", process.version);
console.log("当前目录:", __dirname);
console.log("当前文件:", __filename);
```

```powershell
# 3. 运行程序
node hello.js

# 期望输出:
# Hello, Web3!
# Node.js版本: v20.11.0
# 当前目录: E:\Seadragon\WEB3
# 当前文件: E:\Seadragon\WEB3\hello.js
```

---

## 📦 Part 2: npm包管理器 (1.5小时)

### 2.1 什么是npm？

**npm = Node Package Manager**

```
npm的作用：
✓ 安装第三方库（如web3.js）
✓ 管理项目依赖
✓ 发布自己的包
✓ 运行脚本命令

类比：
npm就像手机的应用商店
- 可以下载应用（安装包）
- 可以更新应用（更新包）
- 可以卸载应用（删除包）
```

### 2.2 npm基础命令

**常用命令速查：**

```powershell
# 1. 初始化项目
npm init
# 或快速初始化（使用默认值）
npm init -y

# 2. 安装包
npm install <package-name>
# 简写
npm i <package-name>

# 3. 安装开发依赖
npm install --save-dev <package-name>
# 简写
npm i -D <package-name>

# 4. 全局安装
npm install -g <package-name>

# 5. 卸载包
npm uninstall <package-name>

# 6. 更新包
npm update <package-name>

# 7. 查看已安装的包
npm list
npm list --depth=0  # 只显示顶层

# 8. 查看包信息
npm info <package-name>

# 9. 搜索包
npm search <keyword>

# 10. 运行脚本
npm run <script-name>
```

### 2.3 package.json详解

**创建package.json：**

```powershell
# 创建项目目录
mkdir web3-learning
cd web3-learning

# 初始化项目
npm init -y
```

**package.json结构：**

```json
{
  "name": "web3-learning",          // 项目名称
  "version": "1.0.0",                // 版本号
  "description": "",                 // 项目描述
  "main": "index.js",                // 入口文件
  "scripts": {                       // 脚本命令
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],                    // 关键词
  "author": "",                      // 作者
  "license": "ISC",                  // 许可证
  "dependencies": {},                // 生产依赖
  "devDependencies": {}              // 开发依赖
}
```

**完善的package.json示例：**

```json
{
  "name": "web3-learning",
  "version": "1.0.0",
  "description": "Web3测试工程师学习项目",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest",
    "lint": "eslint ."
  },
  "keywords": ["web3", "ethereum", "blockchain"],
  "author": "Your Name <your.email@example.com>",
  "license": "MIT",
  "dependencies": {
    "web3": "^4.0.0",
    "ethers": "^6.10.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.0",
    "jest": "^29.0.0"
  }
}
```

**理解dependencies vs devDependencies：**

```javascript
// dependencies（生产依赖）
// 项目运行必需的包
{
  "dependencies": {
    "web3": "^4.0.0",      // 与区块链交互
    "ethers": "^6.10.0",   // 以太坊库
    "express": "^4.18.0"   // Web服务器
  }
}

// devDependencies（开发依赖）
// 只在开发时需要的包
{
  "devDependencies": {
    "hardhat": "^2.19.0",  // 开发框架
    "jest": "^29.0.0",     // 测试框架
    "eslint": "^8.0.0"     // 代码检查
  }
}

// 区别：
// npm install --production  只安装dependencies
// npm install                安装所有依赖
```

### 2.4 版本号规则

**语义化版本（Semver）：**

```
版本格式: MAJOR.MINOR.PATCH
示例: 1.2.3

MAJOR: 主版本号（不兼容的API修改）
MINOR: 次版本号（向后兼容的功能性新增）
PATCH: 修订号（向后兼容的问题修正）

版本前缀：
^1.2.3   允许更新MINOR和PATCH  → 1.x.x
~1.2.3   只允许更新PATCH       → 1.2.x
1.2.3    精确版本              → 1.2.3
>=1.2.3  大于等于              → ≥1.2.3
*        任意版本              → latest

推荐：
生产环境使用精确版本或~
开发时使用^
```

### 2.5 npm配置与优化

**配置国内镜像源（加速下载）：**

```powershell
# 1. 查看当前源
npm config get registry

# 2. 设置淘宝镜像
npm config set registry https://registry.npmmirror.com

# 3. 验证
npm config get registry
# 输出: https://registry.npmmirror.com/

# 4. 恢复官方源（如果需要）
npm config set registry https://registry.npmjs.org/
```

**常用npm配置：**

```powershell
# 查看所有配置
npm config list

# 设置配置
npm config set <key> <value>

# 删除配置
npm config delete <key>

# 常用配置：
npm config set save-exact true      # 保存精确版本
npm config set init-author-name "Your Name"
npm config set init-author-email "your@email.com"
npm config set init-license "MIT"
```

### 2.6 认识pnpm（推荐使用）

**为什么选择pnpm？**

```
npm的问题：
- 重复安装相同包（浪费磁盘空间）
- node_modules臃肿
- 安装速度慢

pnpm的优势：
✓ 节省磁盘空间（硬链接）
✓ 安装速度快3倍+
✓ 严格的依赖管理
✓ 完全兼容npm
```

**安装pnpm：**

```powershell
# 方法1: 使用npm安装（推荐）
npm install -g pnpm

# 方法2: 使用PowerShell脚本
iwr https://get.pnpm.io/install.ps1 -useb | iex

# 验证安装
pnpm --version
# 输出: 8.15.0 (或更高版本)
```

**pnpm命令对照：**

| npm命令 | pnpm命令 | 说明 |
|---------|----------|------|
| npm install | pnpm install | 安装所有依赖 |
| npm i pkg | pnpm add pkg | 添加包 |
| npm i -D pkg | pnpm add -D pkg | 添加开发依赖 |
| npm uninstall pkg | pnpm remove pkg | 删除包 |
| npm run script | pnpm script | 运行脚本 |

**pnpm特有命令：**

```powershell
# 查看磁盘空间占用
pnpm store status

# 清理未使用的包
pnpm store prune

# 列出全局安装的包
pnpm list -g
```

---

## 🔍 Part 3: JavaScript ES6+快速复习 (1.5小时)

### 3.1 变量声明

**let, const, var的区别：**

```javascript
// var（旧语法，不推荐）
var x = 1;
var x = 2;  // 可以重复声明 ❌
if (true) {
  var x = 3;  // 没有块级作用域
}
console.log(x);  // 3

// let（推荐用于可变变量）
let y = 1;
// let y = 2;  // ❌ 不能重复声明
if (true) {
  let y = 3;  // 块级作用域
  console.log(y);  // 3
}
console.log(y);  // 1

// const（推荐用于常量）
const z = 1;
// z = 2;  // ❌ 不能重新赋值
const obj = { name: "Alice" };
obj.name = "Bob";  // ✅ 可以修改对象属性
// obj = {};  // ❌ 不能重新赋值对象

// 最佳实践：
// 1. 默认使用const
// 2. 需要改变时使用let
// 3. 永远不用var
```

### 3.2 箭头函数

```javascript
// 传统函数
function add(a, b) {
  return a + b;
}

// 箭头函数
const add = (a, b) => {
  return a + b;
};

// 简写（单个表达式）
const add = (a, b) => a + b;

// 单个参数可以省略括号
const double = x => x * 2;

// 无参数必须保留括号
const sayHello = () => console.log("Hello");

// 返回对象需要括号
const makePerson = (name, age) => ({ name, age });

// this绑定区别（重要！）
function Traditional() {
  this.value = 1;
  setTimeout(function() {
    this.value++;  // this指向window ❌
  }, 1000);
}

const Arrow = () => {
  this.value = 1;
  setTimeout(() => {
    this.value++;  // this指向外层 ✅
  }, 1000);
}
```

### 3.3 模板字符串

```javascript
// 旧方式
const name = "Alice";
const age = 20;
const message = "My name is " + name + " and I'm " + age + " years old.";

// 模板字符串
const message = `My name is ${name} and I'm ${age} years old.`;

// 多行字符串
const html = `
  <div>
    <h1>${name}</h1>
    <p>Age: ${age}</p>
  </div>
`;

// 表达式计算
const price = 100;
console.log(`Total: ${price * 1.1} USD`);

// Web3中的应用
const address = "0x1234...5678";
const balance = 1.5;
console.log(`Address ${address} has ${balance} ETH`);
```

### 3.4 解构赋值

```javascript
// 数组解构
const arr = [1, 2, 3, 4, 5];
const [first, second, ...rest] = arr;
console.log(first);   // 1
console.log(second);  // 2
console.log(rest);    // [3, 4, 5]

// 对象解构
const person = {
  name: "Alice",
  age: 20,
  city: "Beijing"
};
const { name, age } = person;
console.log(name);  // Alice
console.log(age);   // 20

// 重命名
const { name: userName, age: userAge } = person;

// 默认值
const { country = "China" } = person;

// Web3中的应用
const { ethers } = require("ethers");
const { provider, signer } = ethers;

// 函数参数解构
function transfer({ from, to, amount }) {
  console.log(`Transfer ${amount} from ${from} to ${to}`);
}
transfer({ from: "0x123", to: "0x456", amount: 100 });
```

### 3.5 展开运算符

```javascript
// 数组展开
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];
const combined = [...arr1, ...arr2];  // [1,2,3,4,5,6]

// 对象展开
const obj1 = { a: 1, b: 2 };
const obj2 = { c: 3, d: 4 };
const merged = { ...obj1, ...obj2 };  // {a:1, b:2, c:3, d:4}

// 覆盖属性
const person = { name: "Alice", age: 20 };
const updated = { ...person, age: 21 };  // age被覆盖

// 函数参数
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b, 0);
}
console.log(sum(1, 2, 3, 4));  // 10

// Web3中的应用
const defaultConfig = {
  network: "mainnet",
  gasLimit: 21000
};
const userConfig = {
  network: "sepolia",
  gasPrice: 20
};
const config = { ...defaultConfig, ...userConfig };
```

### 3.6 Promise与async/await

**Promise基础：**

```javascript
// 创建Promise
const myPromise = new Promise((resolve, reject) => {
  setTimeout(() => {
    const success = true;
    if (success) {
      resolve("Success!");
    } else {
      reject("Error!");
    }
  }, 1000);
});

// 使用Promise
myPromise
  .then(result => console.log(result))
  .catch(error => console.error(error));

// Promise链
fetch("https://api.example.com/data")
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => console.error(error));
```

**async/await（推荐）：**

```javascript
// async函数
async function fetchData() {
  try {
    const response = await fetch("https://api.example.com/data");
    const data = await response.json();
    return data;
  } catch (error) {
    console.error("Error:", error);
  }
}

// Web3中的应用
async function getBalance(address) {
  try {
    const provider = new ethers.providers.JsonRpcProvider();
    const balance = await provider.getBalance(address);
    console.log(`Balance: ${ethers.utils.formatEther(balance)} ETH`);
  } catch (error) {
    console.error("Failed to get balance:", error);
  }
}

// 并行执行多个异步操作
async function parallel() {
  const [balance1, balance2, balance3] = await Promise.all([
    getBalance("0x123"),
    getBalance("0x456"),
    getBalance("0x789")
  ]);
}
```

### 3.7 模块系统

**CommonJS（Node.js默认）：**

```javascript
// 导出 (exports.js)
module.exports = {
  add: (a, b) => a + b,
  multiply: (a, b) => a * b
};

// 或
exports.add = (a, b) => a + b;
exports.multiply = (a, b) => a * b;

// 导入
const math = require('./exports.js');
console.log(math.add(1, 2));

// 或解构
const { add, multiply } = require('./exports.js');
console.log(add(1, 2));
```

**ES Modules（现代方式）：**

```javascript
// 导出 (exports.js)
export const add = (a, b) => a + b;
export const multiply = (a, b) => a * b;

// 默认导出
export default function subtract(a, b) {
  return a - b;
}

// 导入
import { add, multiply } from './exports.js';
import subtract from './exports.js';

// 导入所有
import * as math from './exports.js';
```

**在Node.js中使用ES Modules：**

```json
// package.json中添加
{
  "type": "module"
}
```

---

## 📝 今日作业

### 作业1: Node.js环境验证

```markdown
# Node.js环境配置报告

## 1. 版本信息
Node.js版本: _________
npm版本: _________
pnpm版本: _________ (如果安装了)

## 2. 环境测试
[ ] node --version 执行成功
[ ] npm --version 执行成功
[ ] 创建并运行hello.js成功

## 3. 第一个程序
代码:
```javascript
// 粘贴你的hello.js代码


```

输出:
```
// 粘贴运行结果


```
```

### 作业2: npm实践

```markdown
# npm包管理实践

## 1. 初始化项目
[ ] 创建web3-learning目录
[ ] 执行npm init -y
[ ] 查看生成的package.json

## 2. 安装包
安装以下包并记录：
[ ] npm install lodash
[ ] npm install -D jest

记录你的package.json:
```json
// 粘贴内容


```

## 3. 配置镜像源
[ ] 设置淘宝镜像
[ ] 验证registry设置

当前registry: _________
```

### 作业3: JavaScript练习

```javascript
// 完成以下练习

// 1. 使用箭头函数实现
const calculateGas = (gasUsed, gasPrice) => {
  // TODO: 计算Gas费用(gasUsed * gasPrice)
  // 返回结果（单位：Wei）
};

// 2. 使用模板字符串
const address = "0x742d35Cc6634C0532925a3b844E291e6A7E834";
const balance = 1.5;
// TODO: 输出 "Address 0x742d...E834 has 1.5 ETH"

// 3. 使用对象解构
const transaction = {
  from: "0x123",
  to: "0x456",
  value: 1000000,
  gas: 21000
};
// TODO: 解构出from, to, value

// 4. 使用Promise
async function getBlockNumber() {
  // TODO: 模拟异步获取区块号
  // 返回Promise，2秒后resolve(12345678)
}

// 5. 使用模块
// TODO: 创建utils.js，导出formatAddress函数
// formatAddress("0x742d35Cc6634C0532925a3b844E291e6A7E834")
// 返回: "0x742d...E834"
```

---

## ✅ 今日检查清单

### 环境配置
- [ ] Node.js安装成功（v18+）
- [ ] npm可以正常使用
- [ ] pnpm安装成功（可选）
- [ ] 配置npm镜像源
- [ ] 创建第一个.js文件并运行成功

### 知识掌握
- [ ] 理解Node.js的作用
- [ ] 掌握npm基本命令
- [ ] 理解package.json结构
- [ ] 掌握ES6+基础语法
- [ ] 理解async/await

### 实践操作
- [ ] 创建web3-learning项目
- [ ] 初始化package.json
- [ ] 安装至少2个npm包
- [ ] 运行简单的JavaScript程序

---

## 🆘 常见问题FAQ

### Q1: node命令找不到？
```powershell
# 检查PATH
$env:Path -split ';' | Select-String nodejs

# 如果没有，手动添加或重新安装Node.js
# 确保勾选 "Add to PATH"
```

### Q2: npm install很慢？
```powershell
# 1. 使用国内镜像
npm config set registry https://registry.npmmirror.com

# 2. 或使用pnpm
pnpm install

# 3. 或使用代理
npm config set proxy http://proxy.example.com:8080
```

### Q3: 权限错误？
```powershell
# Windows下避免使用sudo/管理员权限
# 如果遇到EACCES错误：

# 方法1: 修改npm全局目录
npm config set prefix '~/.npm-global'

# 方法2: 使用pnpm（推荐）
npm install -g pnpm
```

### Q4: package-lock.json是什么？
```
package-lock.json:
- 锁定依赖的精确版本
- 确保团队使用相同版本
- 加速后续安装

应该提交到Git吗？
✅ Yes - 确保可重现的构建
```

---

## 📅 明日预告: Node.js环境搭建（下）

明天我们将：
- 深入学习JavaScript异步编程
- 理解Promise和async/await
- 学习Node.js模块系统
- 创建完整的Node.js项目
- 学习调试技巧

**今晚练习**：
- 复习ES6+语法
- 练习async/await
- 尝试安装几个npm包

---

## ✍️ 我的学习记录

**完成日期**: ___________
**实际耗时**: _____ 小时

### ✅ 完成情况
- [ ] Node.js安装配置
- [ ] npm包管理学习
- [ ] JavaScript复习
- [ ] 作业完成

### 💡 今日收获
1. 最有价值的知识:
2. 遇到的困难:
3. 解决方案:

### 📝 环境信息
- Node.js版本: _________
- npm版本: _________
- 操作系统: Windows ___

**🎉 完成Day 3！继续前进！**
