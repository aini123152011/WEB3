# Week 5 - Day 5: Web3.js实战

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 掌握Web3.js基础
- ✅ 理解与Ethers.js差异
- ✅ 学习实战案例
- ✅ 掌握迁移技巧
- ✅ 了解选型建议

---

## Part 1: Web3.js基础 (2小时)

### 1.1 安装与初始化

```bash
# 安装Web3.js v4
npm install web3

# 或使用v1版本（注意差异）
npm install web3@1.10.0
```

```javascript
// Web3.js v4 基础使用
const { Web3 } = require('web3');

// 1. 连接到节点
const web3 = new Web3('https://mainnet.infura.io/v3/YOUR-PROJECT-ID');

// WebSocket连接
const web3Ws = new Web3('wss://mainnet.infura.io/ws/v3/YOUR-PROJECT-ID');

// HTTP Provider
const { HttpProvider } = require('web3-providers-http');
const provider = new HttpProvider('https://mainnet.infura.io/v3/YOUR-PROJECT-ID');
const web3Http = new Web3(provider);

// 2. 检查连接
async function checkConnection() {
  try {
    const isListening = await web3.eth.net.isListening();
    console.log('Connected:', isListening);
    
    const networkId = await web3.eth.net.getId();
    console.log('Network ID:', networkId);
    
    const blockNumber = await web3.eth.getBlockNumber();
    console.log('Current block:', blockNumber);
  } catch (error) {
    console.error('Connection error:', error);
  }
}

checkConnection();
```

### 1.2 账户操作

```javascript
const { Web3 } = require('web3');
const web3 = new Web3('https://sepolia.infura.io/v3/YOUR-PROJECT-ID');

// 1. 创建账户
const account = web3.eth.accounts.create();
console.log('Address:', account.address);
console.log('Private Key:', account.privateKey);

// 2. 从私钥恢复
const privateKey = '0x...';
const accountFromKey = web3.eth.accounts.privateKeyToAccount(privateKey);

// 3. 添加到钱包
web3.eth.accounts.wallet.add(accountFromKey);
console.log('Wallet accounts:', web3.eth.accounts.wallet.length);

// 4. 查询余额
async function getBalance(address) {
  const balance = await web3.eth.getBalance(address);
  console.log('Balance (wei):', balance);
  console.log('Balance (ETH):', web3.utils.fromWei(balance, 'ether'));
}

// 5. 查询Nonce
async function getNonce(address) {
  const nonce = await web3.eth.getTransactionCount(address);
  console.log('Nonce:', nonce);
  return nonce;
}

// 6. 发送ETH
async function sendEth() {
  const fromAccount = web3.eth.accounts.wallet[0];
  const toAddress = '0x...';
  
  const tx = {
    from: fromAccount.address,
    to: toAddress,
    value: web3.utils.toWei('0.1', 'ether'),
    gas: 21000,
    gasPrice: await web3.eth.getGasPrice()
  };
  
  const signedTx = await web3.eth.accounts.signTransaction(tx, fromAccount.privateKey);
  const receipt = await web3.eth.sendSignedTransaction(signedTx.rawTransaction);
  
  console.log('Transaction hash:', receipt.transactionHash);
  console.log('Block number:', receipt.blockNumber);
  console.log('Gas used:', receipt.gasUsed);
}
```

### 1.3 工具函数

```javascript
// Web3.js提供的工具函数
const { Web3 } = require('web3');
const web3 = new Web3();

// 1. 单位转换
const weiValue = web3.utils.toWei('1', 'ether');
console.log('1 ETH =', weiValue, 'wei');

const ethValue = web3.utils.fromWei(weiValue, 'ether');
console.log(weiValue, 'wei =', ethValue, 'ETH');

// 支持多种单位
const units = ['wei', 'kwei', 'mwei', 'gwei', 'szabo', 'finney', 'ether'];
units.forEach(unit => {
  console.log(`1 ether = ${web3.utils.toWei('1', 'ether')} ${unit}`);
});

// 2. 哈希函数
const text = 'Hello Web3';
const hash = web3.utils.sha3(text);
console.log('SHA3:', hash);

const keccak = web3.utils.keccak256(text);
console.log('Keccak256:', keccak);

// 3. 编码解码
const hexString = web3.utils.toHex('Hello');
console.log('To Hex:', hexString);

const string = web3.utils.hexToString(hexString);
console.log('From Hex:', string);

// 4. 地址检查
const address = '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb';
const isValid = web3.utils.isAddress(address);
console.log('Is valid address:', isValid);

const checksumAddress = web3.utils.toChecksumAddress(address.toLowerCase());
console.log('Checksum address:', checksumAddress);

// 5. 数值处理
const bn1 = web3.utils.toBN('1000000000000000000');
const bn2 = web3.utils.toBN('2000000000000000000');
const sum = bn1.add(bn2);
console.log('Sum:', sum.toString());

// 6. 填充
const padded = web3.utils.padLeft('0x1', 64);
console.log('Padded:', padded);
```

---

## Part 2: 合约交互 (2小时)

### 2.1 读取合约

```javascript
const { Web3 } = require('web3');
const web3 = new Web3('https://mainnet.infura.io/v3/YOUR-PROJECT-ID');

// ERC20合约ABI
const erc20ABI = [
  {
    constant: true,
    inputs: [],
    name: 'name',
    outputs: [{ name: '', type: 'string' }],
    type: 'function'
  },
  {
    constant: true,
    inputs: [],
    name: 'symbol',
    outputs: [{ name: '', type: 'string' }],
    type: 'function'
  },
  {
    constant: true,
    inputs: [],
    name: 'decimals',
    outputs: [{ name: '', type: 'uint8' }],
    type: 'function'
  },
  {
    constant: true,
    inputs: [],
    name: 'totalSupply',
    outputs: [{ name: '', type: 'uint256' }],
    type: 'function'
  },
  {
    constant: true,
    inputs: [{ name: '_owner', type: 'address' }],
    name: 'balanceOf',
    outputs: [{ name: 'balance', type: 'uint256' }],
    type: 'function'
  }
];

async function readERC20() {
  // USDC合约地址
  const usdcAddress = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';
  const usdc = new web3.eth.Contract(erc20ABI, usdcAddress);
  
  // 读取基本信息
  const name = await usdc.methods.name().call();
  const symbol = await usdc.methods.symbol().call();
  const decimals = await usdc.methods.decimals().call();
  const totalSupply = await usdc.methods.totalSupply().call();
  
  console.log('Token Info:', {
    name,
    symbol,
    decimals,
    totalSupply: web3.utils.fromWei(totalSupply, 'mwei') // USDC是6位小数
  });
  
  // 查询余额
  const holderAddress = '0x...';
  const balance = await usdc.methods.balanceOf(holderAddress).call();
  console.log('Balance:', web3.utils.fromWei(balance, 'mwei'));
  
  // 批量查询
  const addresses = ['0x...', '0x...', '0x...'];
  const balances = await Promise.all(
    addresses.map(addr => usdc.methods.balanceOf(addr).call())
  );
  
  balances.forEach((balance, i) => {
    console.log(`${addresses[i]}: ${web3.utils.fromWei(balance, 'mwei')}`);
  });
}

readERC20();
```

### 2.2 写入合约

```javascript
const { Web3 } = require('web3');
const web3 = new Web3('https://sepolia.infura.io/v3/YOUR-PROJECT-ID');

const erc20ABI = [
  {
    constant: false,
    inputs: [
      { name: '_to', type: 'address' },
      { name: '_value', type: 'uint256' }
    ],
    name: 'transfer',
    outputs: [{ name: '', type: 'bool' }],
    type: 'function'
  },
  {
    constant: false,
    inputs: [
      { name: '_spender', type: 'address' },
      { name: '_value', type: 'uint256' }
    ],
    name: 'approve',
    outputs: [{ name: '', type: 'bool' }],
    type: 'function'
  }
];

async function writeContract() {
  const tokenAddress = '0x...';
  const token = new web3.eth.Contract(erc20ABI, tokenAddress);
  
  // 添加账户
  const privateKey = process.env.PRIVATE_KEY;
  const account = web3.eth.accounts.privateKeyToAccount(privateKey);
  web3.eth.accounts.wallet.add(account);
  
  // 转账
  const toAddress = '0x...';
  const amount = web3.utils.toWei('100', 'ether');
  
  // 方法1: 使用send
  try {
    const receipt = await token.methods.transfer(toAddress, amount).send({
      from: account.address,
      gas: 100000,
      gasPrice: await web3.eth.getGasPrice()
    });
    
    console.log('Transaction hash:', receipt.transactionHash);
    console.log('Block number:', receipt.blockNumber);
    console.log('Gas used:', receipt.gasUsed);
  } catch (error) {
    console.error('Transaction failed:', error);
  }
  
  // 方法2: 手动签名和发送
  const tx = {
    from: account.address,
    to: tokenAddress,
    data: token.methods.transfer(toAddress, amount).encodeABI(),
    gas: 100000,
    gasPrice: await web3.eth.getGasPrice(),
    nonce: await web3.eth.getTransactionCount(account.address)
  };
  
  const signedTx = await account.signTransaction(tx);
  const receipt2 = await web3.eth.sendSignedTransaction(signedTx.rawTransaction);
  console.log('Receipt:', receipt2);
  
  // Gas估算
  const gasEstimate = await token.methods.transfer(toAddress, amount).estimateGas({
    from: account.address
  });
  console.log('Estimated gas:', gasEstimate);
}
```

### 2.3 事件监听

```javascript
const { Web3 } = require('web3');
const web3 = new Web3('wss://mainnet.infura.io/ws/v3/YOUR-PROJECT-ID');

const erc20ABI = [
  {
    anonymous: false,
    inputs: [
      { indexed: true, name: 'from', type: 'address' },
      { indexed: true, name: 'to', type: 'address' },
      { indexed: false, name: 'value', type: 'uint256' }
    ],
    name: 'Transfer',
    type: 'event'
  }
];

async function listenEvents() {
  const tokenAddress = '0x...';
  const token = new web3.eth.Contract(erc20ABI, tokenAddress);
  
  // 1. 监听新事件
  token.events.Transfer({
    fromBlock: 'latest'
  })
  .on('data', (event) => {
    console.log('Transfer detected:', {
      from: event.returnValues.from,
      to: event.returnValues.to,
      value: web3.utils.fromWei(event.returnValues.value, 'ether'),
      blockNumber: event.blockNumber,
      transactionHash: event.transactionHash
    });
  })
  .on('error', (error) => {
    console.error('Event error:', error);
  });
  
  // 2. 监听特定地址
  const specificAddress = '0x...';
  token.events.Transfer({
    filter: { to: specificAddress },
    fromBlock: 'latest'
  })
  .on('data', (event) => {
    console.log('Transfer to specific address:', event.returnValues);
  });
  
  // 3. 查询历史事件
  const pastEvents = await token.getPastEvents('Transfer', {
    fromBlock: (await web3.eth.getBlockNumber()) - 1000,
    toBlock: 'latest'
  });
  
  console.log(`Found ${pastEvents.length} past events`);
  
  pastEvents.slice(0, 10).forEach(event => {
    console.log({
      from: event.returnValues.from,
      to: event.returnValues.to,
      value: event.returnValues.value,
      block: event.blockNumber
    });
  });
  
  // 4. 过滤查询
  const filteredEvents = await token.getPastEvents('Transfer', {
    filter: { from: specificAddress },
    fromBlock: 0,
    toBlock: 'latest'
  });
  
  console.log(`Transfers from ${specificAddress}:`, filteredEvents.length);
}

listenEvents();
```

---

## Part 3: Ethers.js对比 (1.5小时)

### 3.1 API差异对比

```javascript
// ===== 初始化 =====
// Ethers.js
const { ethers } = require('ethers');
const provider = new ethers.JsonRpcProvider(RPC_URL);
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);

// Web3.js
const { Web3 } = require('web3');
const web3 = new Web3(RPC_URL);
const account = web3.eth.accounts.privateKeyToAccount(PRIVATE_KEY);
web3.eth.accounts.wallet.add(account);

// ===== 单位转换 =====
// Ethers.js
const weiEthers = ethers.parseEther('1.0');
const ethEthers = ethers.formatEther(weiEthers);

// Web3.js
const weiWeb3 = web3.utils.toWei('1.0', 'ether');
const ethWeb3 = web3.utils.fromWei(weiWeb3, 'ether');

// ===== 查询余额 =====
// Ethers.js
const balanceEthers = await provider.getBalance(address);

// Web3.js
const balanceWeb3 = await web3.eth.getBalance(address);

// ===== 合约实例化 =====
// Ethers.js
const contractEthers = new ethers.Contract(address, abi, wallet);

// Web3.js
const contractWeb3 = new web3.eth.Contract(abi, address);

// ===== 读取合约 =====
// Ethers.js
const nameEthers = await contractEthers.name();

// Web3.js
const nameWeb3 = await contractWeb3.methods.name().call();

// ===== 写入合约 =====
// Ethers.js
const txEthers = await contractEthers.transfer(to, amount);
const receiptEthers = await txEthers.wait();

// Web3.js
const receiptWeb3 = await contractWeb3.methods.transfer(to, amount).send({
  from: account.address,
  gas: 100000
});

// ===== 事件监听 =====
// Ethers.js
contractEthers.on('Transfer', (from, to, value) => {
  console.log('Transfer:', { from, to, value });
});

// Web3.js
contractWeb3.events.Transfer({
  fromBlock: 'latest'
})
.on('data', (event) => {
  const { from, to, value } = event.returnValues;
  console.log('Transfer:', { from, to, value });
});

// ===== BigNumber处理 =====
// Ethers.js (内置BigInt支持)
const bnEthers = ethers.parseEther('1.5');
const sumEthers = bnEthers + ethers.parseEther('2.5');

// Web3.js (使用BN.js)
const bnWeb3 = web3.utils.toBN(web3.utils.toWei('1.5', 'ether'));
const bn2Web3 = web3.utils.toBN(web3.utils.toWei('2.5', 'ether'));
const sumWeb3 = bnWeb3.add(bn2Web3);
```

### 3.2 性能对比

```javascript
async function performanceComparison() {
  const { Web3 } = require('web3');
  const { ethers } = require('ethers');
  
  const RPC_URL = 'https://mainnet.infura.io/v3/YOUR-PROJECT-ID';
  const tokenAddress = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48'; // USDC
  
  const abi = [
    'function balanceOf(address) view returns (uint256)'
  ];
  
  // Web3.js测试
  console.time('Web3.js - 100 calls');
  const web3 = new Web3(RPC_URL);
  const contractWeb3 = new web3.eth.Contract([{
    constant: true,
    inputs: [{ name: '_owner', type: 'address' }],
    name: 'balanceOf',
    outputs: [{ name: 'balance', type: 'uint256' }],
    type: 'function'
  }], tokenAddress);
  
  const address = '0x...';
  for (let i = 0; i < 100; i++) {
    await contractWeb3.methods.balanceOf(address).call();
  }
  console.timeEnd('Web3.js - 100 calls');
  
  // Ethers.js测试
  console.time('Ethers.js - 100 calls');
  const provider = new ethers.JsonRpcProvider(RPC_URL);
  const contractEthers = new ethers.Contract(tokenAddress, abi, provider);
  
  for (let i = 0; i < 100; i++) {
    await contractEthers.balanceOf(address);
  }
  console.timeEnd('Ethers.js - 100 calls');
}

// 典型结果：
// Web3.js - 100 calls: ~8-10秒
// Ethers.js - 100 calls: ~7-9秒
// (实际性能取决于网络和RPC节点)
```

---

## Part 4: 实战案例 (1.5小时)

### 4.1 代币转账工具

```javascript
const { Web3 } = require('web3');
require('dotenv').config();

const web3 = new Web3(process.env.RPC_URL);

const erc20ABI = [
  {
    constant: true,
    inputs: [{ name: '_owner', type: 'address' }],
    name: 'balanceOf',
    outputs: [{ name: 'balance', type: 'uint256' }],
    type: 'function'
  },
  {
    constant: false,
    inputs: [
      { name: '_to', type: 'address' },
      { name: '_value', type: 'uint256' }
    ],
    name: 'transfer',
    outputs: [{ name: '', type: 'bool' }],
    type: 'function'
  },
  {
    constant: true,
    inputs: [],
    name: 'decimals',
    outputs: [{ name: '', type: 'uint8' }],
    type: 'function'
  }
];

class TokenTransfer {
  constructor(tokenAddress, privateKey) {
    this.token = new web3.eth.Contract(erc20ABI, tokenAddress);
    this.account = web3.eth.accounts.privateKeyToAccount(privateKey);
    web3.eth.accounts.wallet.add(this.account);
  }
  
  async getBalance(address) {
    const balance = await this.token.methods.balanceOf(address).call();
    const decimals = await this.token.methods.decimals().call();
    return {
      raw: balance,
      formatted: (Number(balance) / Math.pow(10, Number(decimals))).toFixed(decimals)
    };
  }
  
  async transfer(to, amount) {
    const decimals = await this.token.methods.decimals().call();
    const value = web3.utils.toBN(amount).mul(
      web3.utils.toBN(10).pow(web3.utils.toBN(decimals))
    );
    
    // 检查余额
    const balance = await this.token.methods.balanceOf(this.account.address).call();
    if (web3.utils.toBN(balance).lt(value)) {
      throw new Error('Insufficient balance');
    }
    
    // 估算Gas
    const gasEstimate = await this.token.methods.transfer(to, value.toString()).estimateGas({
      from: this.account.address
    });
    
    // 发送交易
    const receipt = await this.token.methods.transfer(to, value.toString()).send({
      from: this.account.address,
      gas: Math.floor(Number(gasEstimate) * 1.2), // +20% buffer
      gasPrice: await web3.eth.getGasPrice()
    });
    
    return {
      hash: receipt.transactionHash,
      blockNumber: receipt.blockNumber,
      gasUsed: receipt.gasUsed,
      status: receipt.status
    };
  }
  
  async batchTransfer(recipients) {
    const results = [];
    
    for (const { address, amount } of recipients) {
      try {
        const result = await this.transfer(address, amount);
        results.push({ address, success: true, ...result });
      } catch (error) {
        results.push({ address, success: false, error: error.message });
      }
    }
    
    return results;
  }
}

// 使用示例
async function main() {
  const tokenAddress = '0x...';
  const privateKey = process.env.PRIVATE_KEY;
  
  const transferTool = new TokenTransfer(tokenAddress, privateKey);
  
  // 查询余额
  const balance = await transferTool.getBalance(transferTool.account.address);
  console.log('Balance:', balance.formatted);
  
  // 单笔转账
  const receipt = await transferTool.transfer('0x...', '10');
  console.log('Transfer successful:', receipt.hash);
  
  // 批量转账
  const recipients = [
    { address: '0x...', amount: '5' },
    { address: '0x...', amount: '10' },
    { address: '0x...', amount: '15' }
  ];
  
  const results = await transferTool.batchTransfer(recipients);
  results.forEach(r => {
    console.log(`${r.address}: ${r.success ? 'Success' : 'Failed'}`);
  });
}

main();
```

### 4.2 区块监控

```javascript
const { Web3 } = require('web3');
const web3 = new Web3(new Web3.providers.WebsocketProvider(
  'wss://mainnet.infura.io/ws/v3/YOUR-PROJECT-ID'
));

class BlockMonitor {
  constructor() {
    this.handlers = [];
  }
  
  onNewBlock(handler) {
    this.handlers.push(handler);
  }
  
  async start() {
    console.log('Starting block monitor...');
    
    // 订阅新区块
    const subscription = await web3.eth.subscribe('newBlockHeaders');
    
    subscription.on('data', async (blockHeader) => {
      console.log('New block:', blockHeader.number);
      
      // 获取完整区块信息
      const block = await web3.eth.getBlock(blockHeader.number, true);
      
      const blockInfo = {
        number: block.number,
        hash: block.hash,
        timestamp: block.timestamp,
        transactions: block.transactions.length,
        gasUsed: block.gasUsed,
        gasLimit: block.gasLimit,
        baseFeePerGas: block.baseFeePerGas,
        miner: block.miner
      };
      
      // 调用所有处理函数
      for (const handler of this.handlers) {
        try {
          await handler(blockInfo, block.transactions);
        } catch (error) {
          console.error('Handler error:', error);
        }
      }
    });
    
    subscription.on('error', (error) => {
      console.error('Subscription error:', error);
    });
  }
}

// 使用示例
const monitor = new BlockMonitor();

// 添加处理函数
monitor.onNewBlock(async (blockInfo, transactions) => {
  console.log('Block info:', blockInfo);
  
  // 查找大额交易
  for (const tx of transactions) {
    if (tx.value && web3.utils.fromWei(tx.value, 'ether') > '10') {
      console.log('Large transaction:', {
        hash: tx.hash,
        from: tx.from,
        to: tx.to,
        value: web3.utils.fromWei(tx.value, 'ether')
      });
    }
  }
});

monitor.start();
```

---

## Part 5: 迁移指南 (1小时)

### 5.1 从Web3.js迁移到Ethers.js

```javascript
// 迁移映射表
const migrationMap = {
  // Provider
  'new Web3(url)': 'new ethers.JsonRpcProvider(url)',
  'new Web3.providers.WebsocketProvider(url)': 'new ethers.WebSocketProvider(url)',
  
  // Account
  'web3.eth.accounts.create()': 'ethers.Wallet.createRandom()',
  'web3.eth.accounts.privateKeyToAccount(key)': 'new ethers.Wallet(key)',
  'web3.eth.accounts.wallet.add(account)': '// Not needed in ethers',
  
  // Units
  'web3.utils.toWei(value, "ether")': 'ethers.parseEther(value)',
  'web3.utils.fromWei(value, "ether")': 'ethers.formatEther(value)',
  
  // Contract
  'new web3.eth.Contract(abi, address)': 'new ethers.Contract(address, abi, provider)',
  'contract.methods.func().call()': 'contract.func()',
  'contract.methods.func().send({from})': 'contract.func()',
  
  // Events
  'contract.events.Event({fromBlock})': 'contract.on("Event", handler)',
  'contract.getPastEvents("Event", {fromBlock})': 'contract.queryFilter("Event", fromBlock)',
  
  // Utils
  'web3.utils.sha3(text)': 'ethers.id(text)',
  'web3.utils.isAddress(addr)': 'ethers.isAddress(addr)',
  'web3.utils.toChecksumAddress(addr)': 'ethers.getAddress(addr)'
};

// 完整迁移示例
// Web3.js代码
async function web3Example() {
  const { Web3 } = require('web3');
  const web3 = new Web3('https://mainnet.infura.io/v3/YOUR-PROJECT-ID');
  
  const tokenAddress = '0x...';
  const abi = [...];
  const token = new web3.eth.Contract(abi, tokenAddress);
  
  const balance = await token.methods.balanceOf('0x...').call();
  console.log('Balance:', web3.utils.fromWei(balance, 'ether'));
  
  const account = web3.eth.accounts.privateKeyToAccount('0x...');
  web3.eth.accounts.wallet.add(account);
  
  const receipt = await token.methods.transfer('0x...', web3.utils.toWei('1', 'ether')).send({
    from: account.address
  });
  
  console.log('Hash:', receipt.transactionHash);
}

// 迁移后的Ethers.js代码
async function ethersExample() {
  const { ethers } = require('ethers');
  const provider = new ethers.JsonRpcProvider('https://mainnet.infura.io/v3/YOUR-PROJECT-ID');
  
  const tokenAddress = '0x...';
  const abi = [...];
  const token = new ethers.Contract(tokenAddress, abi, provider);
  
  const balance = await token.balanceOf('0x...');
  console.log('Balance:', ethers.formatEther(balance));
  
  const wallet = new ethers.Wallet('0x...', provider);
  const tokenWithSigner = token.connect(wallet);
  
  const tx = await tokenWithSigner.transfer('0x...', ethers.parseEther('1'));
  const receipt = await tx.wait();
  
  console.log('Hash:', receipt.hash);
}
```

### 5.2 选型建议

```markdown
## Web3.js vs Ethers.js 选型指南

### 选择Web3.js的场景：
1. **现有项目使用Web3.js** - 保持一致性
2. **需要更多底层控制** - Web3.js提供更多底层API
3. **团队熟悉Web3.js** - 降低学习成本
4. **需要特定功能** - 某些特性Web3.js支持更好

### 选择Ethers.js的场景：
1. **新项目** - 推荐使用Ethers.js
2. **更简洁的API** - Ethers.js API更现代
3. **更小的包体积** - Ethers.js更轻量
4. **TypeScript项目** - Ethers.js类型支持更好
5. **更好的文档** - Ethers.js文档更完善

### 功能对比：

| 功能 | Web3.js | Ethers.js |
|-----|---------|-----------|
| 包大小 | ~300KB | ~120KB |
| TypeScript | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 文档 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 社区 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 学习曲线 | 中等 | 较低 |
| API设计 | 传统 | 现代 |
| 性能 | 良好 | 优秀 |

### 最佳实践：
1. **新项目推荐Ethers.js**
2. **两者可以共存** - 不同模块使用不同库
3. **逐步迁移** - 老项目可以逐步迁移到Ethers.js
4. **根据需求选择** - 没有绝对的好坏
```

---

## 📝 今日作业

### 作业1: 完整DApp后端

使用Web3.js开发：
1. 代币信息查询
2. 转账功能
3. 历史记录
4. 事件监听

### 作业2: 迁移练习

将一个Web3.js项目迁移到Ethers.js：
1. 分析差异
2. 重写代码
3. 测试功能
4. 性能对比

### 作业3: 库选型报告

撰写技术选型报告：
1. 对比分析
2. 适用场景
3. 迁移成本
4. 推荐方案

---

## ✅ 检查清单

- [ ] 掌握Web3.js基础
- [ ] 理解与Ethers.js差异
- [ ] 完成实战案例
- [ ] 理解迁移方法
- [ ] 能够做技术选型

---

## 📅 周末预告

周末综合项目：
- 自动化测试完整项目
- 集成测试框架
- CI/CD配置
- 报告生成

**🎉 完成Day 5！周末项目见！**
