# Week 5 - Day 3: Ethers.js深入 - 上

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 深入理解Provider
- ✅ 掌握Signer管理
- ✅ 学习钱包操作
- ✅ 理解网络配置
- ✅ 掌握查询方法
- ✅ 学习工具函数

---

## Part 1: Provider深入 (2小时)

### 1.1 Provider类型

```javascript
const { ethers } = require("ethers");

// 1. JsonRpcProvider - 连接到JSON-RPC节点
const provider = new ethers.JsonRpcProvider("https://eth-mainnet.alchemyapi.io/v2/YOUR-API-KEY");

// 2. WebSocketProvider - WebSocket连接
const wsProvider = new ethers.WebSocketProvider("wss://eth-mainnet.ws.alchemyapi.io/v2/YOUR-API-KEY");

// 3. BrowserProvider - MetaMask等浏览器钱包
if (typeof window !== 'undefined' && window.ethereum) {
  const browserProvider = new ethers.BrowserProvider(window.ethereum);
}

// 4. FallbackProvider - 多个Provider故障转移
const fallbackProvider = new ethers.FallbackProvider([
  new ethers.JsonRpcProvider("https://mainnet.infura.io/v3/YOUR-KEY"),
  new ethers.JsonRpcProvider("https://eth-mainnet.alchemyapi.io/v2/YOUR-KEY")
]);

// 5. IpcProvider - IPC连接（Node.js）
// const ipcProvider = new ethers.IpcProvider("/path/to/geth.ipc");
```

### 1.2 Provider基础操作

```javascript
async function providerBasics() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  
  // 获取网络信息
  const network = await provider.getNetwork();
  console.log("Network:", {
    name: network.name,
    chainId: network.chainId.toString()
  });
  
  // 获取区块号
  const blockNumber = await provider.getBlockNumber();
  console.log("Current block:", blockNumber);
  
  // 获取Gas价格
  const feeData = await provider.getFeeData();
  console.log("Fee Data:", {
    gasPrice: ethers.formatUnits(feeData.gasPrice, "gwei"),
    maxFeePerGas: ethers.formatUnits(feeData.maxFeePerGas, "gwei"),
    maxPriorityFeePerGas: ethers.formatUnits(feeData.maxPriorityFeePerGas, "gwei")
  });
  
  // 获取账户余额
  const address = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb";
  const balance = await provider.getBalance(address);
  console.log("Balance:", ethers.formatEther(balance), "ETH");
  
  // 获取交易数量（nonce）
  const txCount = await provider.getTransactionCount(address);
  console.log("Transaction count:", txCount);
  
  // 获取代码（检查是否是合约）
  const code = await provider.getCode(address);
  const isContract = code !== "0x";
  console.log("Is contract:", isContract);
}
```

### 1.3 区块查询

```javascript
async function blockQueries() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  
  // 获取最新区块
  const latestBlock = await provider.getBlock("latest");
  console.log("Latest block:", {
    number: latestBlock.number,
    hash: latestBlock.hash,
    timestamp: latestBlock.timestamp,
    miner: latestBlock.miner,
    gasUsed: latestBlock.gasUsed.toString(),
    transactions: latestBlock.transactions.length
  });
  
  // 获取特定区块
  const block = await provider.getBlock(18000000);
  console.log("Block 18000000:", {
    number: block.number,
    hash: block.hash,
    parentHash: block.parentHash,
    nonce: block.nonce,
    difficulty: block.difficulty
  });
  
  // 获取区块中的所有交易
  const blockWithTxs = await provider.getBlock(18000000, true);
  console.log("Transactions in block:", blockWithTxs.transactions.length);
  
  // 遍历交易
  for (const tx of blockWithTxs.transactions.slice(0, 3)) {
    console.log("Transaction:", {
      hash: tx.hash,
      from: tx.from,
      to: tx.to,
      value: ethers.formatEther(tx.value)
    });
  }
}
```

### 1.4 交易查询

```javascript
async function transactionQueries() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  
  const txHash = "0x5c504ed432cb51138bcf09aa5e8a410dd4a1e204ef84bfed1be16dfba1b22060";
  
  // 获取交易详情
  const tx = await provider.getTransaction(txHash);
  console.log("Transaction:", {
    hash: tx.hash,
    from: tx.from,
    to: tx.to,
    value: ethers.formatEther(tx.value),
    gasLimit: tx.gasLimit.toString(),
    gasPrice: ethers.formatUnits(tx.gasPrice, "gwei"),
    nonce: tx.nonce,
    data: tx.data,
    chainId: tx.chainId
  });
  
  // 获取交易回执
  const receipt = await provider.getTransactionReceipt(txHash);
  if (receipt) {
    console.log("Receipt:", {
      status: receipt.status, // 1 = success, 0 = failed
      blockNumber: receipt.blockNumber,
      blockHash: receipt.blockHash,
      gasUsed: receipt.gasUsed.toString(),
      cumulativeGasUsed: receipt.cumulativeGasUsed.toString(),
      logs: receipt.logs.length,
      contractAddress: receipt.contractAddress
    });
    
    // 解析日志
    for (const log of receipt.logs) {
      console.log("Log:", {
        address: log.address,
        topics: log.topics,
        data: log.data
      });
    }
  }
}
```

---

## Part 2: Signer深入 (2小时)

### 2.1 创建Signer

```javascript
async function createSigners() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  
  // 1. 从私钥创建
  const privateKey = "0x0123456789012345678901234567890123456789012345678901234567890123";
  const wallet1 = new ethers.Wallet(privateKey, provider);
  console.log("Wallet 1:", wallet1.address);
  
  // 2. 从助记词创建
  const mnemonic = "test test test test test test test test test test test junk";
  const wallet2 = ethers.Wallet.fromPhrase(mnemonic, provider);
  console.log("Wallet 2:", wallet2.address);
  
  // 3. 创建随机钱包
  const randomWallet = ethers.Wallet.createRandom(provider);
  console.log("Random wallet:", {
    address: randomWallet.address,
    mnemonic: randomWallet.mnemonic.phrase,
    privateKey: randomWallet.privateKey
  });
  
  // 4. 从加密JSON创建
  const json = await wallet1.encrypt("password123");
  const wallet3 = await ethers.Wallet.fromEncryptedJson(json, "password123", provider);
  console.log("Wallet 3:", wallet3.address);
  
  // 5. HD钱包派生
  const hdNode = ethers.HDNodeWallet.fromPhrase(mnemonic);
  const derivedWallet1 = hdNode.derivePath("m/44'/60'/0'/0/0");
  const derivedWallet2 = hdNode.derivePath("m/44'/60'/0'/0/1");
  console.log("Derived wallets:", derivedWallet1.address, derivedWallet2.address);
}
```

### 2.2 签名操作

```javascript
async function signingOperations() {
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY);
  
  // 1. 签名消息
  const message = "Hello, Ethereum!";
  const signature = await wallet.signMessage(message);
  console.log("Signature:", signature);
  
  // 验证签名
  const recoveredAddress = ethers.verifyMessage(message, signature);
  console.log("Recovered address:", recoveredAddress);
  console.log("Match:", recoveredAddress === wallet.address);
  
  // 2. 签名交易
  const tx = {
    to: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    value: ethers.parseEther("0.1"),
    gasLimit: 21000,
    gasPrice: ethers.parseUnits("20", "gwei"),
    nonce: 0,
    chainId: 1
  };
  
  const signedTx = await wallet.signTransaction(tx);
  console.log("Signed transaction:", signedTx);
  
  // 3. EIP-712类型化数据签名
  const domain = {
    name: "MyDApp",
    version: "1",
    chainId: 1,
    verifyingContract: "0x1234567890123456789012345678901234567890"
  };
  
  const types = {
    Person: [
      { name: "name", type: "string" },
      { name: "wallet", type: "address" }
    ],
    Mail: [
      { name: "from", type: "Person" },
      { name: "to", type: "Person" },
      { name: "contents", type: "string" }
    ]
  };
  
  const value = {
    from: {
      name: "Alice",
      wallet: "0x1111111111111111111111111111111111111111"
    },
    to: {
      name: "Bob",
      wallet: "0x2222222222222222222222222222222222222222"
    },
    contents: "Hello, Bob!"
  };
  
  const typedSignature = await wallet.signTypedData(domain, types, value);
  console.log("Typed data signature:", typedSignature);
}
```

### 2.3 发送交易

```javascript
async function sendTransactions() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
  
  // 1. 发送ETH
  const tx1 = await wallet.sendTransaction({
    to: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    value: ethers.parseEther("0.001")
  });
  
  console.log("Transaction sent:", tx1.hash);
  
  // 等待确认
  const receipt1 = await tx1.wait();
  console.log("Transaction confirmed:", receipt1.status === 1);
  
  // 2. 指定Gas参数
  const tx2 = await wallet.sendTransaction({
    to: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    value: ethers.parseEther("0.001"),
    gasLimit: 21000,
    maxFeePerGas: ethers.parseUnits("50", "gwei"),
    maxPriorityFeePerGas: ethers.parseUnits("2", "gwei")
  });
  
  await tx2.wait();
  
  // 3. 发送数据
  const tx3 = await wallet.sendTransaction({
    to: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    data: "0x1234567890"
  });
  
  await tx3.wait();
  
  // 4. 部署合约
  const abi = ["constructor(string memory name, string memory symbol)"];
  const bytecode = "0x608060405234801561001057600080fd5b50...";
  
  const factory = new ethers.ContractFactory(abi, bytecode, wallet);
  const contract = await factory.deploy("MyToken", "MTK");
  await contract.waitForDeployment();
  
  console.log("Contract deployed at:", await contract.getAddress());
}
```

---

## Part 3: 钱包管理 (1.5小时)

### 3.1 钱包加密与存储

```javascript
async function walletEncryption() {
  // 创建钱包
  const wallet = ethers.Wallet.createRandom();
  console.log("Address:", wallet.address);
  console.log("Private key:", wallet.privateKey);
  console.log("Mnemonic:", wallet.mnemonic.phrase);
  
  // 加密钱包（使用密码）
  const password = "MySecurePassword123!";
  
  console.log("Encrypting wallet...");
  const encryptedJson = await wallet.encrypt(password);
  console.log("Encrypted JSON:", encryptedJson);
  
  // 保存到文件
  const fs = require('fs');
  fs.writeFileSync('wallet.json', encryptedJson);
  
  // 从加密JSON恢复
  console.log("Decrypting wallet...");
  const restoredWallet = await ethers.Wallet.fromEncryptedJson(
    encryptedJson,
    password
  );
  
  console.log("Restored address:", restoredWallet.address);
  console.log("Addresses match:", wallet.address === restoredWallet.address);
  
  // 使用自定义加密选项
  const customOptions = {
    scrypt: {
      N: 1024,  // CPU/内存成本
      r: 8,
      p: 1
    }
  };
  
  const fastEncrypted = await wallet.encrypt(password, customOptions);
  console.log("Fast encrypted JSON length:", fastEncrypted.length);
}
```

### 3.2 HD钱包派生

```javascript
async function hdWalletDerivation() {
  const mnemonic = ethers.Wallet.createRandom().mnemonic.phrase;
  console.log("Mnemonic:", mnemonic);
  
  // 创建HD节点
  const hdNode = ethers.HDNodeWallet.fromPhrase(mnemonic);
  
  // 派生账户（BIP-44标准）
  // m/44'/60'/0'/0/index
  const accounts = [];
  for (let i = 0; i < 5; i++) {
    const path = `m/44'/60'/0'/0/${i}`;
    const derived = hdNode.derivePath(path);
    accounts.push({
      index: i,
      path: path,
      address: derived.address,
      privateKey: derived.privateKey
    });
  }
  
  console.log("Derived accounts:");
  accounts.forEach(acc => {
    console.log(`  ${acc.index}: ${acc.address}`);
  });
  
  // 派生不同的账户类型
  // 外部账户（接收）
  const external0 = hdNode.derivePath("m/44'/60'/0'/0/0");
  // 内部账户（找零）
  const internal0 = hdNode.derivePath("m/44'/60'/0'/1/0");
  
  console.log("External account:", external0.address);
  console.log("Internal account:", internal0.address);
  
  // 派生不同币种
  // Bitcoin: m/44'/0'/0'/0/0
  // Ethereum: m/44'/60'/0'/0/0
  // Testnet: m/44'/1'/0'/0/0
}
```

### 3.3 多签钱包操作

```javascript
async function multiSigOperations() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  
  // 创建多个签名者
  const signer1 = new ethers.Wallet(process.env.PRIVATE_KEY1, provider);
  const signer2 = new ethers.Wallet(process.env.PRIVATE_KEY2, provider);
  const signer3 = new ethers.Wallet(process.env.PRIVATE_KEY3, provider);
  
  // 连接到多签钱包合约
  const multiSigAddress = "0x...";
  const multiSigABI = [
    "function submitTransaction(address to, uint256 value, bytes data) returns (uint256)",
    "function confirmTransaction(uint256 txId)",
    "function executeTransaction(uint256 txId)",
    "function getTransactionCount() view returns (uint256)"
  ];
  
  const multiSig = new ethers.Contract(multiSigAddress, multiSigABI, provider);
  
  // Signer1提交交易
  const tx = await multiSig.connect(signer1).submitTransaction(
    "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    ethers.parseEther("1.0"),
    "0x"
  );
  
  const receipt = await tx.wait();
  const txId = receipt.logs[0].topics[1]; // 假设第一个日志包含txId
  
  // Signer2确认
  await multiSig.connect(signer2).confirmTransaction(txId);
  
  // Signer3执行（达到阈值后）
  await multiSig.connect(signer3).executeTransaction(txId);
  
  console.log("Multi-sig transaction executed");
}
```

---

## Part 4: 网络配置 (1小时)

### 4.1 网络切换

```javascript
async function networkSwitching() {
  // 定义多个网络配置
  const networks = {
    mainnet: {
      name: "Ethereum Mainnet",
      rpc: process.env.MAINNET_RPC_URL,
      chainId: 1
    },
    goerli: {
      name: "Goerli Testnet",
      rpc: process.env.GOERLI_RPC_URL,
      chainId: 5
    },
    sepolia: {
      name: "Sepolia Testnet",
      rpc: process.env.SEPOLIA_RPC_URL,
      chainId: 11155111
    },
    polygon: {
      name: "Polygon Mainnet",
      rpc: "https://polygon-rpc.com",
      chainId: 137
    }
  };
  
  // 创建Providers
  const providers = {};
  for (const [key, config] of Object.entries(networks)) {
    providers[key] = new ethers.JsonRpcProvider(config.rpc);
  }
  
  // 查询不同网络的状态
  for (const [key, provider] of Object.entries(providers)) {
    try {
      const network = await provider.getNetwork();
      const blockNumber = await provider.getBlockNumber();
      const feeData = await provider.getFeeData();
      
      console.log(`${networks[key].name}:`, {
        chainId: network.chainId.toString(),
        blockNumber: blockNumber,
        gasPrice: ethers.formatUnits(feeData.gasPrice, "gwei")
      });
    } catch (error) {
      console.error(`Error with ${key}:`, error.message);
    }
  }
}
```

### 4.2 自定义网络

```javascript
async function customNetwork() {
  // 配置自定义网络
  const customNetworkConfig = {
    name: "My Custom Network",
    chainId: 1337,
    ensAddress: null
  };
  
  const provider = new ethers.JsonRpcProvider(
    "http://localhost:8545",
    customNetworkConfig
  );
  
  // 验证网络
  const network = await provider.getNetwork();
  console.log("Connected to:", network.name, "Chain ID:", network.chainId);
  
  // 配置Layer 2网络
  const arbitrumProvider = new ethers.JsonRpcProvider(
    "https://arb1.arbitrum.io/rpc",
    {
      name: "Arbitrum One",
      chainId: 42161
    }
  );
  
  const optimismProvider = new ethers.JsonRpcProvider(
    "https://mainnet.optimism.io",
    {
      name: "Optimism",
      chainId: 10
    }
  );
}
```

---

## Part 5: 工具函数 (1.5小时)

### 5.1 单位转换

```javascript
function unitConversions() {
  // Wei <-> Ether
  const wei = ethers.parseEther("1.5");
  console.log("1.5 ETH in wei:", wei.toString());
  
  const ether = ethers.formatEther(wei);
  console.log("Wei to ETH:", ether);
  
  // 其他单位
  const gwei = ethers.parseUnits("20", "gwei");
  console.log("20 Gwei in wei:", gwei.toString());
  
  const gweiFormatted = ethers.formatUnits(gwei, "gwei");
  console.log("Wei to Gwei:", gweiFormatted);
  
  // 自定义精度（如ERC20代币）
  const usdcAmount = ethers.parseUnits("100.5", 6); // USDC有6位小数
  console.log("100.5 USDC:", usdcAmount.toString());
  
  const usdcFormatted = ethers.formatUnits(usdcAmount, 6);
  console.log("Format USDC:", usdcFormatted);
  
  // BigInt运算
  const value1 = ethers.parseEther("1.5");
  const value2 = ethers.parseEther("2.3");
  const sum = value1 + value2;
  console.log("Sum:", ethers.formatEther(sum), "ETH");
}
```

### 5.2 地址操作

```javascript
function addressOperations() {
  // 地址验证
  const address = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb";
  console.log("Is valid:", ethers.isAddress(address));
  
  // Checksum地址
  const checksumAddress = ethers.getAddress(address.toLowerCase());
  console.log("Checksum address:", checksumAddress);
  
  // 从公钥计算地址
  const publicKey = "0x04..."; // 完整公钥
  const computedAddress = ethers.computeAddress(publicKey);
  console.log("Computed address:", computedAddress);
  
  // 合约地址预测
  const deployerAddress = "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb";
  const nonce = 5;
  const contractAddress = ethers.getCreateAddress({
    from: deployerAddress,
    nonce: nonce
  });
  console.log("Contract address (CREATE):", contractAddress);
  
  // CREATE2地址预测
  const salt = ethers.id("my-salt");
  const initCodeHash = ethers.keccak256("0x60806040...");
  const create2Address = ethers.getCreate2Address(
    deployerAddress,
    salt,
    initCodeHash
  );
  console.log("Contract address (CREATE2):", create2Address);
}
```

### 5.3 哈希与编码

```javascript
function hashingAndEncoding() {
  // Keccak256
  const hash1 = ethers.keccak256(ethers.toUtf8Bytes("Hello, World!"));
  console.log("Keccak256:", hash1);
  
  // ID（字符串到bytes32）
  const id = ethers.id("Transfer(address,address,uint256)");
  console.log("Function signature hash:", id);
  
  // Soliditykeccak256
  const packed = ethers.solidityPackedKeccak256(
    ["address", "uint256"],
    ["0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb", 123]
  );
  console.log("Solidity packed hash:", packed);
  
  // ABI编码
  const abiCoder = ethers.AbiCoder.defaultAbiCoder();
  const encoded = abiCoder.encode(
    ["address", "uint256", "string"],
    ["0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb", 123, "Hello"]
  );
  console.log("ABI encoded:", encoded);
  
  // ABI解码
  const decoded = abiCoder.decode(
    ["address", "uint256", "string"],
    encoded
  );
  console.log("ABI decoded:", decoded);
  
  // Base64编码
  const base64 = ethers.encodeBase64(ethers.toUtf8Bytes("Hello"));
  console.log("Base64:", base64);
  
  const decodedBase64 = ethers.decodeBase64(base64);
  console.log("Decoded:", ethers.toUtf8String(decodedBase64));
}
```

---

## 📝 今日作业

### 作业1: Provider完整应用

实现Provider工具类：
1. 网络信息查询
2. 区块数据获取
3. 交易追踪
4. Gas费用监控

### 作业2: 钱包管理系统

创建钱包管理工具：
1. 创建/导入钱包
2. 加密存储
3. HD派生
4. 签名验证

### 作业3: 多网络DApp

开发多网络支持：
1. 网络切换
2. 状态同步
3. 交易发送
4. 错误处理

---

## ✅ 检查清单

- [ ] 理解Provider类型
- [ ] 掌握Signer操作
- [ ] 会钱包管理
- [ ] 理解网络配置
- [ ] 掌握工具函数

---

## 📅 明日预告

明天学习Ethers.js深入 - 下：
- Contract深入
- 事件监听
- 过滤器使用
- 批量操作

**🎉 完成Day 3！明天见！**
