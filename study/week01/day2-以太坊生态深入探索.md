# Week 1 - Day 2: 以太坊生态深入探索

**学习日期**: ___________
**预计用时**: 4-5小时
**难度等级**: ⭐⭐ (入门)

## 📚 今日学习目标

- ✅ 深入理解Etherscan的高级功能
- ✅ 掌握ERC20代币标准
- ✅ 学会添加自定义代币到钱包
- ✅ 理解DApp的工作原理
- ✅ 体验钱包与DApp的交互
- ✅ 了解Web3技术栈全貌

---

## 📖 Part 1: Etherscan深度使用 (1.5小时)

### 1.1 区块浏览器的作用

**为什么需要区块浏览器？**
```
区块浏览器 = 区块链的搜索引擎

功能：
✓ 查询交易详情
✓ 查看账户余额
✓ 追踪资金流向
✓ 验证智能合约
✓ 分析链上数据
✓ 监控网络状态
```

### 1.2 查看智能合约

#### 访问一个真实的ERC20代币合约

**示例：查看USDT合约（测试网示例合约）**

```
步骤：
1. 访问 Sepolia Etherscan
2. 搜索一个示例ERC20合约地址
   (我们将使用一个测试代币地址)
3. 点击 "Contract" 标签
```

**合约页面解读：**
```
┌──────────────────────────────────────┐
│ Contract Overview                     │
├──────────────────────────────────────┤
│ Balance: 0 ETH                       │
│ Transactions: 1,234                  │
│ Contract Creator: 0xabc...           │
│ Creation TxHash: 0x123...            │
│ Token Tracker: MyToken (MTK)         │
├──────────────────────────────────────┤
│ Code | Read Contract | Write Contract│
└──────────────────────────────────────┘
```

**重要标签页：**

1. **Code**（代码）
   - 显示合约源代码
   - 如果已验证，可以看到Solidity代码
   - 未验证只显示字节码

2. **Read Contract**（读取合约）
   - 查询合约状态（不需要Gas）
   - 例如：余额、总供应量
   
3. **Write Contract**（写入合约）
   - 执行合约函数（需要Gas）
   - 需要连接钱包
   - 例如：转账、授权

#### 实践：读取合约信息

**任务：使用Etherscan读取代币信息**

```
1. 访问一个测试代币合约
2. 点击 "Read Contract"
3. 尝试查询以下信息：

   name()          → 代币名称
   symbol()        → 代币符号
   decimals()      → 小数位数
   totalSupply()   → 总供应量
   balanceOf(地址) → 查询某地址余额
```

**记录你的查询结果：**
```
代币名称: _________
代币符号: _________
小数位数: _________
总供应量: _________
你的余额: _________
```

### 1.3 追踪交易流向

**场景：分析一笔复杂交易**

```
复杂交易示例：
- 用户A → DEX合约
- DEX合约 → Token合约
- Token合约 → 用户A和用户B

如何在Etherscan追踪？
1. 找到主交易哈希
2. 查看 "Internal Txns" 标签
3. 查看 "Logs" 标签
4. 查看 "State" 变化
```

**Internal Transactions（内部交易）：**
```
内部交易 = 合约内部的ETH转移

示例：
┌─────────────────────────────────┐
│ Internal Transactions            │
├─────────────────────────────────┤
│ Parent Txn Hash: 0xabc...       │
│                                  │
│ From: Contract A → Contract B    │
│ Value: 0.1 ETH                  │
│                                  │
│ From: Contract B → User         │
│ Value: 0.05 ETH                 │
└─────────────────────────────────┘
```

**Logs（事件日志）：**
```
事件日志 = 合约触发的事件

常见事件：
- Transfer: 代币转账
- Approval: 授权
- Swap: 交换
- Deposit: 存款
- Withdraw: 取款

查看事件详情：
┌────────────────────────────────┐
│ Event: Transfer                 │
├────────────────────────────────┤
│ from: 0x123...                 │
│ to: 0x456...                   │
│ value: 1000000000000000000     │
│       (= 1 Token)              │
└────────────────────────────────┘
```

### 1.4 Etherscan高级搜索

**多维度搜索：**
```
1. 按地址搜索
   - 查看所有交易历史
   - 查看代币持有情况
   - 查看NFT持有情况

2. 按交易哈希搜索
   - 交易详情
   - 区块确认
   - Gas使用情况

3. 按区块号搜索
   - 区块内所有交易
   - 区块奖励
   - 打包者信息

4. 按代币搜索
   - 代币持有者排行
   - 转账记录
   - 合约地址
```

**实践任务：**
```
1. 搜索你的MetaMask地址
2. 查看 "ERC-20 Token Txns" 标签
3. 记录你钱包中的所有代币（如果有）
4. 点击任意代币，查看该代币的完整信息
```

---

## 🪙 Part 2: ERC20代币标准 (1.5小时)

### 2.1 什么是ERC20？

**ERC = Ethereum Request for Comments**
```
ERC20 = 以太坊上的代币标准

类比：
就像USB接口标准，所有遵循该标准的设备都能互相兼容

ERC20的意义：
✓ 统一的代币接口
✓ 钱包通用支持
✓ DEX可以交易所有ERC20代币
✓ 开发者遵循相同规范
```

### 2.2 ERC20核心函数

**必须实现的6个函数：**

```solidity
// 1. 返回代币名称
function name() public view returns (string)

// 2. 返回代币符号
function symbol() public view returns (string)

// 3. 返回小数位数
function decimals() public view returns (uint8)

// 4. 返回总供应量
function totalSupply() public view returns (uint256)

// 5. 返回某地址的余额
function balanceOf(address _owner) public view returns (uint256 balance)

// 6. 转账
function transfer(address _to, uint256 _value) public returns (bool success)

// 7. 授权额度
function approve(address _spender, uint256 _value) public returns (bool success)

// 8. 查询授权额度
function allowance(address _owner, address _spender) public view returns (uint256 remaining)

// 9. 从授权额度中转账
function transferFrom(address _from, address _to, uint256 _value) public returns (bool success)
```

**关键概念：Decimals（小数位）**

```
为什么需要decimals？
- 区块链不支持小数
- 使用整数 × 10^decimals 表示

示例：
USDT: decimals = 6
1 USDT = 1,000,000 (链上存储)
0.5 USDT = 500,000

ETH: decimals = 18
1 ETH = 1,000,000,000,000,000,000 Wei
```

**计算公式：**
```javascript
// 链上值转显示值
显示值 = 链上值 / (10 ** decimals)

// 显示值转链上值
链上值 = 显示值 * (10 ** decimals)

// JavaScript示例
const amount = 1.5; // USDT
const decimals = 6;
const onChainValue = amount * (10 ** decimals); // 1500000
```

### 2.3 ERC20授权机制

**Approve + TransferFrom模式：**

```
为什么需要授权？
场景：使用DEX交易代币

步骤1: 授权DEX可以使用你的代币
用户 → approve(DEX地址, 金额)

步骤2: DEX执行交换
DEX → transferFrom(用户, 池子, 金额)

安全考虑：
⚠️ 不要授权无限额度
⚠️ 定期检查和撤销授权
⚠️ 只授权可信合约
```

**授权流程图：**
```
用户钱包
   ↓ [1. approve(DEX, 100 USDT)]
代币合约
   ├─ 记录: 用户授权给DEX 100 USDT
   
DEX合约
   ↓ [2. 用户点击swap]
   ↓ [3. transferFrom(用户, 池子, 50 USDT)]
代币合约
   ├─ 检查授权额度: 100 ≥ 50 ✓
   ├─ 扣除授权额度: 100 - 50 = 50
   └─ 转账: 用户 → 池子 (50 USDT)
```

### 2.4 创建测试ERC20代币

**使用Remix IDE快速创建：**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MyToken {
    string public name = "My Test Token";
    string public symbol = "MTK";
    uint8 public decimals = 18;
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    constructor(uint256 _initialSupply) {
        totalSupply = _initialSupply * 10 ** uint256(decimals);
        balanceOf[msg.sender] = totalSupply;
    }
    
    function transfer(address _to, uint256 _value) public returns (bool success) {
        require(balanceOf[msg.sender] >= _value, "Insufficient balance");
        balanceOf[msg.sender] -= _value;
        balanceOf[_to] += _value;
        emit Transfer(msg.sender, _to, _value);
        return true;
    }
    
    function approve(address _spender, uint256 _value) public returns (bool success) {
        allowance[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }
    
    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success) {
        require(_value <= balanceOf[_from], "Insufficient balance");
        require(_value <= allowance[_from][msg.sender], "Insufficient allowance");
        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;
        allowance[_from][msg.sender] -= _value;
        emit Transfer(_from, _to, _value);
        return true;
    }
}
```

**部署步骤：**
```
1. 访问 https://remix.ethereum.org/
2. 创建新文件 MyToken.sol
3. 粘贴上面的代码
4. 编译合约（Ctrl+S）
5. 切换环境到 "Injected Provider - MetaMask"
6. 确保MetaMask在Sepolia网络
7. 部署合约，设置初始供应量（如1000000）
8. 记录合约地址
```

---

## 💼 Part 3: 添加自定义代币到钱包 (30分钟)

### 3.1 添加代币的两种方法

**方法1: 通过合约地址添加（推荐）**

```
步骤：
1. 打开MetaMask
2. 确保在Sepolia测试网
3. 点击 "导入代币" (Import Tokens)
4. 选择 "自定义代币" (Custom Token)
5. 输入代币合约地址
6. 自动填充代币符号和小数位
7. 点击 "添加自定义代币"
8. 确认导入

示例：
┌────────────────────────────┐
│ 导入代币                   │
├────────────────────────────┤
│ 代币合约地址:              │
│ [0x1234567890abcdef...]   │
│                            │
│ 代币符号: MTK  (自动填充)  │
│ 代币小数: 18   (自动填充)  │
├────────────────────────────┤
│     [取消]  [添加代币]     │
└────────────────────────────┘
```

**方法2: 从列表搜索（仅主流代币）**
```
仅适用于主网知名代币：
- USDT
- USDC  
- DAI
- UNI
等

测试网代币必须使用方法1
```

### 3.2 实践：添加你的测试代币

**任务清单：**
```
1. [ ] 部署MyToken合约到Sepolia（或使用示例合约）
2. [ ] 复制合约地址
3. [ ] 在MetaMask导入该代币
4. [ ] 验证余额显示正确
5. [ ] 向Account 2转账一些代币
6. [ ] 在Etherscan查看转账记录
```

**转账代币步骤：**
```
1. 在MetaMask选择你的代币
2. 点击 "发送"
3. 输入Account 2地址
4. 输入金额（如100 MTK）
5. 确认Gas费用
6. 发送并等待确认
7. 切换到Account 2，查看是否收到
```

### 3.3 理解代币转账vs ETH转账

**对比分析：**

| 项目 | ETH转账 | ERC20转账 |
|------|---------|-----------|
| **交易类型** | 简单转账 | 合约调用 |
| **Gas消耗** | 21,000 | 约65,000 |
| **交易数据** | 空 | 函数调用数据 |
| **to地址** | 接收方地址 | 代币合约地址 |
| **value** | 转账金额 | 0 ETH |

**ERC20转账的交易结构：**
```javascript
{
  from: "0x你的地址",
  to: "0x代币合约地址",      // ← 注意：不是接收方！
  value: "0",                  // ← ETH数量为0
  data: "0xa9059cbb..." +      // transfer函数签名
        "接收方地址" +
        "代币数量",
  gas: 65000,
  gasPrice: "2000000000"
}
```

**为什么Gas费更高？**
```
ETH转账：
- 简单的账户余额修改
- 固定21,000 Gas

ERC20转账：
- 调用合约函数
- 读取mapping（余额）
- 修改mapping（发送方-接收方余额）
- 触发事件
- 总计约65,000 Gas
```

---

## 🌐 Part 4: DApp工作原理 (1.5小时)

### 4.1 什么是DApp？

**DApp = Decentralized Application（去中心化应用）**

```
传统App vs DApp

传统App:
用户 → 前端 → 中心化服务器 → 数据库
              ↑
          单点故障

DApp:
用户 → 前端 → 钱包 → 区块链节点 → 智能合约
                      ↓
                  去中心化网络
```

**DApp的特点：**
```
✓ 开源代码
✓ 数据存储在链上
✓ 使用加密货币
✓ 共识机制验证
✓ 无需中间人
✗ 无法修改部署的合约
✗ 交易不可撤销
```

### 4.2 DApp技术栈

**完整的DApp由以下部分组成：**

```
┌────────────────────────────────┐
│        前端 (Frontend)          │
│  React/Vue + Web3.js/Ethers.js │
├────────────────────────────────┤
│       钱包 (Wallet)             │
│    MetaMask / WalletConnect    │
├────────────────────────────────┤
│      节点 (RPC Node)            │
│   Infura / Alchemy / Ankr      │
├────────────────────────────────┤
│     智能合约 (Smart Contract)   │
│         Solidity              │
├────────────────────────────────┤
│      区块链 (Blockchain)        │
│        Ethereum Network        │
└────────────────────────────────┘
```

**各层详解：**

1. **前端层**
```javascript
// 使用Web3.js与合约交互
import Web3 from 'web3';

const web3 = new Web3(window.ethereum);
const contract = new web3.eth.Contract(ABI, contractAddress);

// 读取数据（免费）
const balance = await contract.methods.balanceOf(address).call();

// 写入数据（需要Gas）
await contract.methods.transfer(to, amount).send({ from: address });
```

2. **钱包层**
```javascript
// 连接MetaMask
if (window.ethereum) {
  const accounts = await window.ethereum.request({ 
    method: 'eth_requestAccounts' 
  });
  console.log('Connected:', accounts[0]);
}

// 切换网络
await window.ethereum.request({
  method: 'wallet_switchEthereumChain',
  params: [{ chainId: '0xaa36a7' }], // Sepolia
});
```

3. **智能合约层**
```solidity
// 合约提供业务逻辑
contract MyDApp {
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) public {
        require(balances[msg.sender] >= amount);
        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
}
```

### 4.3 体验真实DApp

**推荐初学者体验的DApp：**

#### 1. Uniswap（DEX去中心化交易所）

```
访问：https://app.uniswap.org/ (切换到Sepolia测试网)

体验流程：
1. 连接MetaMask钱包
2. 切换到Sepolia测试网
3. 选择代币对（需要测试代币）
4. 查看兑换比例
5. 了解流动性池概念

⚠️ 测试网可能缺乏流动性，但可以理解界面
```

#### 2. OpenSea（NFT市场）

```
访问：https://testnets.opensea.io/ (测试网版本)

体验流程：
1. 连接钱包
2. 浏览NFT
3. 查看NFT详情
4. 理解NFT元数据
```

#### 3. Aave（借贷协议）

```
访问测试网版本

功能：
- 存款赚取利息
- 抵押借款
- 查看利率
```

### 4.4 DApp交互流程详解

**完整的交互流程：**

```
步骤1: 用户连接钱包
用户点击"连接钱包" → MetaMask弹窗 → 用户授权

步骤2: 前端读取链上数据
DApp → 通过RPC节点 → 读取智能合约状态
显示：余额、价格、可用额度等

步骤3: 用户发起操作
用户点击"Swap" → 前端构造交易数据

步骤4: 钱包签名
MetaMask弹出确认窗口 → 显示Gas费 → 用户确认
私钥签名交易（私钥不离开钱包）

步骤5: 交易广播
签名的交易 → RPC节点 → 区块链网络 → 内存池

步骤6: 交易确认
矿工打包 → 区块确认 → 状态更新

步骤7: 前端更新
监听事件 → 更新UI → 显示最新状态
```

**MetaMask弹窗解读：**
```
┌────────────────────────────────┐
│ 🦊 MetaMask 确认                │
├────────────────────────────────┤
│ Uniswap V3                     │
│ 请求权限                        │
│                                │
│ 合约交互:                       │
│ Swap 1 ETH → 3000 USDC        │
│                                │
│ Gas 费用:                      │
│ 0.005 ETH ≈ $12.50            │
│                                │
│ 总计:                          │
│ 1.005 ETH ≈ $2512.50          │
├────────────────────────────────┤
│    [拒绝]        [确认]        │
└────────────────────────────────┘

⚠️ 确认前检查：
1. 合约地址是否正确
2. 交易金额是否正确
3. Gas费是否合理
4. 不要签名可疑请求
```

---

## 🔧 Part 5: 实践项目 (30分钟)

### 5.1 项目：创建并交互自己的代币

**完整流程：**

```
任务清单：
[ ] 1. 使用Remix部署ERC20代币
[ ] 2. 在Etherscan验证合约（可选）
[ ] 3. 添加代币到MetaMask
[ ] 4. 向朋友转账代币（或自己的Account 2）
[ ] 5. 在Etherscan查看转账记录
[ ] 6. 记录所有重要地址和交易哈希
```

**项目记录表：**
```markdown
# 我的第一个ERC20代币

## 基本信息
- 代币名称: ___________
- 代币符号: ___________
- 小数位数: ___________
- 总供应量: ___________
- 合约地址: 0x___________

## 部署信息
- 部署交易: 0x___________
- 部署区块: ___________
- Gas消耗: ___________
- 部署时间: ___________

## 转账记录
### 转账 #1
- 从: 0x___________
- 到: 0x___________
- 金额: ___________
- 交易哈希: 0x___________
- Gas费用: ___________

### 转账 #2
- 从: 0x___________
- 到: 0x___________
- 金额: ___________
- 交易哈希: 0x___________
- Gas费用: ___________

## Etherscan链接
- 合约: https://sepolia.etherscan.io/address/0x___________
- 转账1: https://sepolia.etherscan.io/tx/0x___________
- 转账2: https://sepolia.etherscan.io/tx/0x___________
```

### 5.2 深入分析你的代币转账

**在Etherscan查看：**

1. **Transaction Details**
   - 记录Gas消耗
   - 对比ETH转账的差异

2. **Logs标签**
   - 查看Transfer事件
   - 理解事件参数

3. **State Changes**
   - 查看余额变化
   - 理解状态转换

---

## 📝 今日作业

### 作业1: ERC20代币分析报告

```markdown
# ERC20代币深度分析

## 1. 合约分析
选择一个测试网ERC20代币（可以是你部署的）

- 合约地址: ___________
- 代币名称: ___________
- 总持有者: ___________
- 总转账次数: ___________

## 2. 函数调用实践
使用Etherscan的Read Contract功能：

balanceOf(你的地址) = ___________
totalSupply() = ___________
decimals() = ___________

## 3. 转账对比
完成1笔ETH转账和1笔ERC20转账，对比：

|  项目  | ETH转账 | ERC20转账 |
|--------|---------|-----------|
| Gas Used | _____ | _____ |
| Gas Price | _____ | _____ |
| Total Fee | _____ | _____ |
| 确认时间 | _____ | _____ |

## 4. 思考题
1. 为什么ERC20转账比ETH转账贵？


2. approve和transfer的区别是什么？


3. 如果代币合约有bug，已经转出的代币能找回吗？


```

### 作业2: DApp交互体验报告

```markdown
# DApp交互体验

## 1. 访问的DApp
名称: ___________
网址: ___________
类型: [ ] DEX [ ] NFT [ ] Lending [ ] 其他

## 2. 连接钱包过程
遇到的问题:


解决方法:


## 3. 界面截图
(可选) 上传MetaMask连接成功的截图

## 4. 学习收获
理解了什么是:
1. 
2.
3.

## 5. 疑问
还不理解的概念:


```

---

## ✅ 今日检查清单

### 理论学习
- [ ] 理解Etherscan的高级功能
- [ ] 掌握ERC20标准的6个核心函数
- [ ] 理解approve授权机制
- [ ] 了解DApp的工作原理
- [ ] 理解Web3技术栈

### 实践操作
- [ ] 成功部署ERC20代币（或使用示例）
- [ ] 在MetaMask添加自定义代币
- [ ] 完成至少2笔代币转账
- [ ] 在Etherscan分析代币交易
- [ ] 连接至少1个DApp

### 作业提交
- [ ] 完成ERC20分析报告
- [ ] 完成DApp体验报告
- [ ] 记录所有重要地址和哈希

---

## 🆘 常见问题FAQ

### Q1: 在Remix部署合约失败？
```
可能原因：
1. MetaMask未切换到Sepolia
2. 账户余额不足
3. Gas设置过低

解决方法：
1. 确认网络：MetaMask右上角显示"Sepolia"
2. 检查余额：至少0.1 SepoliaETH
3. 重新部署时增加Gas Limit
```

### Q2: MetaMask看不到代币余额？
```
原因：
- 未正确导入代币
- 合约地址错误
- 网络不匹配

解决方法：
1. 重新导入，仔细核对合约地址
2. 确保MetaMask在Sepolia网络
3. 等待几分钟后刷新
4. 设置 → 高级 → 重置账户
```

### Q3: 代币转账一直Pending？
```
可能原因：
- Gas价格过低
- 网络拥堵
- 代币合约有问题

解决方法：
1. 等待10-15分钟
2. 加速交易（Speed Up）
3. 检查合约是否正常（Etherscan）
```

### Q4: DApp连接不上钱包？
```
检查项：
1. MetaMask是否安装
2. 是否在正确的网络
3. 网站是否支持该网络
4. 清除浏览器缓存
5. 刷新页面重试
```

---

## 📚 扩展学习

### 推荐阅读
1. [ERC20标准文档](https://eips.ethereum.org/EIPS/eip-20)
2. [OpenZeppelin ERC20实现](https://docs.openzeppelin.com/contracts/erc20)
3. [Web3.js文档](https://web3js.readthedocs.io/)

### 推荐视频
1. "What is an ERC20 Token?" - Whiteboard Crypto
2. "How to Create an ERC20 Token" - EatTheBlocks

---

## 📅 明日预告: Node.js环境搭建

明天我们将：
- 安装Node.js和npm/pnpm
- 学习JavaScript/TypeScript基础
- 配置开发环境
- 创建第一个Node.js项目
- 准备Hardhat开发环境

**今晚准备**：
- 下载Node.js安装包（v18+）
- 下载VSCode（如果还没有）
- 复习JavaScript基础语法

---

## ✍️ 我的学习记录

**完成日期**: ___________
**实际耗时**: _____ 小时

### ✅ 完成情况
- [ ] 理论学习
- [ ] Etherscan深度使用
- [ ] 部署ERC20代币
- [ ] DApp交互体验
- [ ] 作业提交

### 💡 今日收获
1. 最有价值的知识点:
2. 最困难的部分:
3. 成功的实践:

### 📝 重要地址记录
- 我的ERC20合约: 0x____________________
- 部署交易: 0x____________________
- 转账交易1: 0x____________________
- 转账交易2: 0x____________________

### 🤔 疑问与思考
- 问题1:
- 问题2:

### 📅 明日计划
- 重点: Node.js环境搭建
- 预计难点: npm包管理

**🎉 完成Day 2！继续加油！**
