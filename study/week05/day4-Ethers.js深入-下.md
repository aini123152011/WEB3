# Week 5 - Day 4: Ethers.js深入 - 下

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 学习目标

- ✅ 深入Contract操作
- ✅ 掌握事件监听
- ✅ 学习过滤器使用
- ✅ 理解批量操作
- ✅ 掌握ENS解析
- ✅ 学习错误处理

---

## Part 1: Contract深入操作 (2小时)

### 1.1 Contract实例化

```javascript
const { ethers } = require("ethers");

async function contractInstantiation() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
  
  // 1. 通过ABI和地址创建
  const tokenAddress = "0x...";
  const tokenABI = [
    "function name() view returns (string)",
    "function symbol() view returns (string)",
    "function totalSupply() view returns (uint256)",
    "function balanceOf(address) view returns (uint256)",
    "function transfer(address to, uint256 amount) returns (bool)",
    "event Transfer(address indexed from, address indexed to, uint256 value)"
  ];
  
  // 只读合约
  const tokenRead = new ethers.Contract(tokenAddress, tokenABI, provider);
  
  // 可写合约
  const tokenWrite = new ethers.Contract(tokenAddress, tokenABI, wallet);
  
  // 2. 从部署中创建
  const TokenFactory = new ethers.ContractFactory(
    tokenABI,
    "0x60806040...", // bytecode
    wallet
  );
  
  const newToken = await TokenFactory.deploy("MyToken", "MTK");
  await newToken.waitForDeployment();
  
  console.log("New token deployed at:", await newToken.getAddress());
  
  // 3. 附加到已部署合约
  const attachedToken = TokenFactory.attach(tokenAddress);
  
  // 4. 使用接口
  const iface = new ethers.Interface(tokenABI);
  console.log("Interface fragments:", iface.fragments.length);
}
```

### 1.2 读取合约数据

```javascript
async function readContractData() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  
  // ERC20合约示例
  const usdcAddress = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48";
  const erc20ABI = [
    "function name() view returns (string)",
    "function symbol() view returns (string)",
    "function decimals() view returns (uint8)",
    "function totalSupply() view returns (uint256)",
    "function balanceOf(address) view returns (uint256)"
  ];
  
  const usdc = new ethers.Contract(usdcAddress, erc20ABI, provider);
  
  // 读取基本信息
  const [name, symbol, decimals, totalSupply] = await Promise.all([
    usdc.name(),
    usdc.symbol(),
    usdc.decimals(),
    usdc.totalSupply()
  ]);
  
  console.log("Token Info:", {
    name,
    symbol,
    decimals,
    totalSupply: ethers.formatUnits(totalSupply, decimals)
  });
  
  // 查询余额
  const holderAddress = "0x...";
  const balance = await usdc.balanceOf(holderAddress);
  console.log("Balance:", ethers.formatUnits(balance, decimals));
  
  // 批量查询
  const addresses = [
    "0x...",
    "0x...",
    "0x..."
  ];
  
  const balances = await Promise.all(
    addresses.map(addr => usdc.balanceOf(addr))
  );
  
  balances.forEach((balance, i) => {
    console.log(`${addresses[i]}: ${ethers.formatUnits(balance, decimals)}`);
  });
}
```

### 1.3 写入合约数据

```javascript
async function writeContractData() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
  
  const tokenAddress = "0x...";
  const tokenABI = [
    "function transfer(address to, uint256 amount) returns (bool)",
    "function approve(address spender, uint256 amount) returns (bool)",
    "function transferFrom(address from, address to, uint256 amount) returns (bool)"
  ];
  
  const token = new ethers.Contract(tokenAddress, tokenABI, wallet);
  
  // 1. 简单转账
  const tx1 = await token.transfer(
    "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    ethers.parseEther("100")
  );
  console.log("Transaction hash:", tx1.hash);
  await tx1.wait();
  console.log("Transaction confirmed");
  
  // 2. 带Gas估算
  const gasEstimate = await token.transfer.estimateGas(
    "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    ethers.parseEther("100")
  );
  
  const tx2 = await token.transfer(
    "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    ethers.parseEther("100"),
    {
      gasLimit: gasEstimate * 120n / 100n, // +20% buffer
      maxFeePerGas: ethers.parseUnits("50", "gwei"),
      maxPriorityFeePerGas: ethers.parseUnits("2", "gwei")
    }
  );
  
  await tx2.wait();
  
  // 3. 静态调用（不发送交易）
  const result = await token.transfer.staticCall(
    "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    ethers.parseEther("100")
  );
  console.log("Static call result:", result);
  
  // 4. 填充交易（获取完整交易对象）
  const populatedTx = await token.transfer.populateTransaction(
    "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    ethers.parseEther("100")
  );
  console.log("Populated transaction:", populatedTx);
}
```

---

## Part 2: 事件监听 (2小时)

### 2.1 基础事件监听

```javascript
async function basicEventListening() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  
  const tokenAddress = "0x...";
  const tokenABI = [
    "event Transfer(address indexed from, address indexed to, uint256 value)",
    "event Approval(address indexed owner, address indexed spender, uint256 value)"
  ];
  
  const token = new ethers.Contract(tokenAddress, tokenABI, provider);
  
  // 1. 监听单个事件
  token.on("Transfer", (from, to, value, event) => {
    console.log("Transfer detected:", {
      from,
      to,
      value: ethers.formatEther(value),
      blockNumber: event.log.blockNumber,
      transactionHash: event.log.transactionHash
    });
  });
  
  // 2. 监听一次
  token.once("Transfer", (from, to, value) => {
    console.log("First transfer:", ethers.formatEther(value));
  });
  
  // 3. 监听特定地址的转账
  const filter = token.filters.Transfer(null, "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb");
  
  token.on(filter, (from, to, value) => {
    console.log(`Transfer to specific address: ${ethers.formatEther(value)}`);
  });
  
  // 4. 移除监听器
  setTimeout(() => {
    token.removeAllListeners("Transfer");
    console.log("All Transfer listeners removed");
  }, 60000); // 1分钟后移除
}
```

### 2.2 过滤和查询历史事件

```javascript
async function queryHistoricalEvents() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  
  const tokenAddress = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"; // USDC
  const tokenABI = [
    "event Transfer(address indexed from, address indexed to, uint256 value)"
  ];
  
  const token = new ethers.Contract(tokenAddress, tokenABI, provider);
  
  // 1. 查询所有转账
  const currentBlock = await provider.getBlockNumber();
  const fromBlock = currentBlock - 1000; // 最近1000个区块
  
  const allTransfers = await token.queryFilter(
    "Transfer",
    fromBlock,
    currentBlock
  );
  
  console.log(`Found ${allTransfers.length} transfers`);
  
  // 2. 查询特定地址的转账
  const specificAddress = "0x...";
  const filter = token.filters.Transfer(specificAddress, null);
  
  const transfersFrom = await token.queryFilter(filter, fromBlock, currentBlock);
  console.log(`Transfers from ${specificAddress}: ${transfersFrom.length}`);
  
  // 3. 查询转入特定地址
  const filterTo = token.filters.Transfer(null, specificAddress);
  const transfersTo = await token.queryFilter(filterTo, fromBlock, currentBlock);
  console.log(`Transfers to ${specificAddress}: ${transfersTo.length}`);
  
  // 4. 处理事件数据
  for (const event of allTransfers.slice(0, 10)) {
    console.log({
      from: event.args.from,
      to: event.args.to,
      value: ethers.formatUnits(event.args.value, 6),
      block: event.blockNumber,
      tx: event.transactionHash
    });
  }
  
  // 5. 分批查询（避免RPC限制）
  async function queryEventsBatch(fromBlock, toBlock, batchSize = 1000) {
    const events = [];
    
    for (let block = fromBlock; block <= toBlock; block += batchSize) {
      const endBlock = Math.min(block + batchSize - 1, toBlock);
      const batchEvents = await token.queryFilter("Transfer", block, endBlock);
      events.push(...batchEvents);
      console.log(`Queried blocks ${block} to ${endBlock}: ${batchEvents.length} events`);
    }
    
    return events;
  }
  
  const allEvents = await queryEventsBatch(currentBlock - 10000, currentBlock);
  console.log(`Total events: ${allEvents.length}`);
}
```

### 2.3 实时事件监控

```javascript
async function realTimeEventMonitoring() {
  const provider = new ethers.WebSocketProvider(process.env.WSS_URL);
  
  const tokenAddress = "0x...";
  const tokenABI = [
    "event Transfer(address indexed from, address indexed to, uint256 value)",
    "event Approval(address indexed owner, address indexed spender, uint256 value)"
  ];
  
  const token = new ethers.Contract(tokenAddress, tokenABI, provider);
  
  // 监控大额转账
  const threshold = ethers.parseEther("1000000"); // 100万代币
  
  token.on("Transfer", async (from, to, value, event) => {
    if (value >= threshold) {
      console.log("🚨 Large transfer detected!");
      console.log({
        from,
        to,
        value: ethers.formatEther(value),
        tx: event.log.transactionHash
      });
      
      // 获取交易详情
      const tx = await provider.getTransaction(event.log.transactionHash);
      console.log("Gas price:", ethers.formatUnits(tx.gasPrice, "gwei"));
      
      // 发送通知（示例）
      await sendAlert({
        type: "LARGE_TRANSFER",
        amount: ethers.formatEther(value),
        tx: event.log.transactionHash
      });
    }
  });
  
  // 错误处理
  provider.on("error", (error) => {
    console.error("Provider error:", error);
  });
  
  // 保持连接
  console.log("Monitoring started...");
}

async function sendAlert(data) {
  // 实现通知逻辑（如发送邮件、Telegram等）
  console.log("Alert sent:", data);
}
```

---

## Part 3: 高级过滤器 (1.5小时)

### 3.1 复杂过滤器

```javascript
async function advancedFilters() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  
  const dexAddress = "0x...";
  const dexABI = [
    "event Swap(address indexed sender, uint256 amount0In, uint256 amount1In, uint256 amount0Out, uint256 amount1Out, address indexed to)",
    "event Mint(address indexed sender, uint256 amount0, uint256 amount1)",
    "event Burn(address indexed sender, uint256 amount0, uint256 amount1, address indexed to)"
  ];
  
  const dex = new ethers.Contract(dexAddress, dexABI, provider);
  
  // 1. 多条件过滤
  const userAddress = "0x...";
  
  // 查询特定用户的所有交换
  const swapFilter = dex.filters.Swap(userAddress);
  const swaps = await dex.queryFilter(swapFilter, -10000);
  
  // 查询特定用户的流动性操作
  const mintFilter = dex.filters.Mint(userAddress);
  const burnFilter = dex.filters.Burn(userAddress);
  
  const [mints, burns] = await Promise.all([
    dex.queryFilter(mintFilter, -10000),
    dex.queryFilter(burnFilter, -10000)
  ]);
  
  console.log("User activity:", {
    swaps: swaps.length,
    mints: mints.length,
    burns: burns.length
  });
  
  // 2. OR过滤器（监听多个事件）
  const topics = [
    dex.interface.getEvent("Swap").topicHash,
    dex.interface.getEvent("Mint").topicHash,
    dex.interface.getEvent("Burn").topicHash
  ];
  
  provider.on({
    address: dexAddress,
    topics: [topics]
  }, (log) => {
    const parsed = dex.interface.parseLog(log);
    console.log("Event:", parsed.name);
  });
  
  // 3. 自定义主题过滤
  const customFilter = {
    address: dexAddress,
    topics: [
      dex.interface.getEvent("Swap").topicHash,
      ethers.zeroPadValue(userAddress, 32) // indexed sender
    ]
  };
  
  const customEvents = await provider.getLogs({
    ...customFilter,
    fromBlock: -10000,
    toBlock: "latest"
  });
  
  console.log("Custom filter results:", customEvents.length);
}
```

### 3.2 事件解码

```javascript
async function eventDecoding() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  
  const contractAddress = "0x...";
  const contractABI = [
    "event ComplexEvent(address indexed user, uint256[] amounts, string data, (address token, uint256 amount)[] transfers)"
  ];
  
  const contract = new ethers.Contract(contractAddress, contractABI, provider);
  const iface = contract.interface;
  
  // 获取原始日志
  const logs = await provider.getLogs({
    address: contractAddress,
    fromBlock: -1000,
    toBlock: "latest"
  });
  
  // 解码日志
  for (const log of logs) {
    try {
      const parsed = iface.parseLog(log);
      
      if (parsed) {
        console.log("Event name:", parsed.name);
        console.log("Arguments:", parsed.args);
        
        // 访问特定参数
        if (parsed.name === "ComplexEvent") {
          console.log("User:", parsed.args.user);
          console.log("Amounts:", parsed.args.amounts.map(a => a.toString()));
          console.log("Data:", parsed.args.data);
          console.log("Transfers:", parsed.args.transfers);
        }
      }
    } catch (error) {
      console.log("Could not parse log:", error.message);
    }
  }
  
  // 手动解码
  const eventSignature = "Transfer(address,address,uint256)";
  const eventTopic = ethers.id(eventSignature);
  
  const transferLogs = logs.filter(log => log.topics[0] === eventTopic);
  
  for (const log of transferLogs) {
    const from = ethers.getAddress(ethers.dataSlice(log.topics[1], 12));
    const to = ethers.getAddress(ethers.dataSlice(log.topics[2], 12));
    const value = ethers.getBigInt(log.data);
    
    console.log("Transfer:", { from, to, value: value.toString() });
  }
}
```

---

## Part 4: 批量操作与优化 (1.5小时)

### 4.1 Multicall批量查询

```javascript
async function multicallBatchQuery() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  
  // Multicall3合约地址（多链通用）
  const multicallAddress = "0xcA11bde05977b3631167028862bE2a173976CA11";
  const multicallABI = [
    "function aggregate3(tuple(address target, bool allowFailure, bytes callData)[] calls) view returns (tuple(bool success, bytes returnData)[] returnData)"
  ];
  
  const multicall = new ethers.Contract(multicallAddress, multicallABI, provider);
  
  // 准备多个调用
  const tokenAddress = "0x...";
  const tokenABI = [
    "function balanceOf(address) view returns (uint256)",
    "function decimals() view returns (uint8)"
  ];
  
  const token = new ethers.Contract(tokenAddress, tokenABI, provider);
  const addresses = [
    "0x...",
    "0x...",
    "0x..."
  ];
  
  // 构建调用数据
  const calls = addresses.map(addr => ({
    target: tokenAddress,
    allowFailure: true,
    callData: token.interface.encodeFunctionData("balanceOf", [addr])
  }));
  
  // 执行批量调用
  const results = await multicall.aggregate3(calls);
  
  // 解析结果
  const decimals = await token.decimals();
  
  results.forEach((result, i) => {
    if (result.success) {
      const balance = token.interface.decodeFunctionResult(
        "balanceOf",
        result.returnData
      )[0];
      
      console.log(`${addresses[i]}: ${ethers.formatUnits(balance, decimals)}`);
    } else {
      console.log(`${addresses[i]}: Failed`);
    }
  });
}
```

### 4.2 批量交易

```javascript
async function batchTransactions() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
  
  const tokenAddress = "0x...";
  const tokenABI = [
    "function transfer(address to, uint256 amount) returns (bool)"
  ];
  
  const token = new ethers.Contract(tokenAddress, tokenABI, wallet);
  
  const recipients = [
    { address: "0x...", amount: ethers.parseEther("10") },
    { address: "0x...", amount: ethers.parseEther("20") },
    { address: "0x...", amount: ethers.parseEther("30") }
  ];
  
  // 方法1: 顺序发送
  console.log("Sequential transactions:");
  for (const { address, amount } of recipients) {
    const tx = await token.transfer(address, amount);
    console.log(`Sent to ${address}: ${tx.hash}`);
    await tx.wait();
    console.log("Confirmed");
  }
  
  // 方法2: 并行发送（不等待确认）
  console.log("Parallel transactions:");
  const txPromises = recipients.map(({ address, amount }, i) =>
    token.transfer(address, amount, {
      nonce: await wallet.getNonce() + i // 手动设置nonce
    })
  );
  
  const txs = await Promise.all(txPromises);
  txs.forEach(tx => console.log("Transaction:", tx.hash));
  
  // 等待所有确认
  const receipts = await Promise.all(txs.map(tx => tx.wait()));
  console.log("All confirmed:", receipts.every(r => r.status === 1));
  
  // 方法3: 使用批量转账合约
  const batchTransferABI = [
    "function batchTransfer(address[] recipients, uint256[] amounts)"
  ];
  
  const batchContract = new ethers.Contract("0x...", batchTransferABI, wallet);
  
  const tx = await batchContract.batchTransfer(
    recipients.map(r => r.address),
    recipients.map(r => r.amount)
  );
  
  await tx.wait();
  console.log("Batch transfer completed");
}
```

---

## Part 5: ENS与特殊功能 (1小时)

### 5.1 ENS域名解析

```javascript
async function ensOperations() {
  const provider = new ethers.JsonRpcProvider(process.env.MAINNET_RPC_URL);
  
  // 1. 解析ENS到地址
  const address = await provider.resolveName("vitalik.eth");
  console.log("vitalik.eth resolves to:", address);
  
  // 2. 反向解析（地址到ENS）
  const name = await provider.lookupAddress(address);
  console.log("Address resolves to:", name);
  
  // 3. 获取ENS记录
  const resolver = await provider.getResolver("vitalik.eth");
  
  if (resolver) {
    // 获取文本记录
    const avatar = await resolver.getAvatar();
    const email = await resolver.getText("email");
    const twitter = await resolver.getText("com.twitter");
    
    console.log("ENS Records:", {
      avatar: avatar?.url,
      email,
      twitter
    });
    
    // 获取内容哈希（IPFS等）
    const contentHash = await resolver.getContentHash();
    console.log("Content hash:", contentHash);
  }
  
  // 4. 批量ENS解析
  const names = ["vitalik.eth", "nick.eth", "brantly.eth"];
  const addresses = await Promise.all(
    names.map(name => provider.resolveName(name))
  );
  
  names.forEach((name, i) => {
    console.log(`${name} -> ${addresses[i]}`);
  });
}
```

### 5.2 错误处理

```javascript
async function errorHandling() {
  const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
  
  const tokenAddress = "0x...";
  const tokenABI = [
    "function transfer(address to, uint256 amount) returns (bool)",
    "error InsufficientBalance(uint256 available, uint256 required)"
  ];
  
  const token = new ethers.Contract(tokenAddress, tokenABI, wallet);
  
  try {
    const tx = await token.transfer(
      "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
      ethers.parseEther("1000000")
    );
    await tx.wait();
  } catch (error) {
    // 处理不同类型的错误
    if (error.code === "INSUFFICIENT_FUNDS") {
      console.error("Insufficient ETH for gas");
    } else if (error.code === "NONCE_EXPIRED") {
      console.error("Nonce已被使用");
    } else if (error.code === "REPLACEMENT_UNDERPRICED") {
      console.error("Gas价格过低");
    } else if (error.code === "UNPREDICTABLE_GAS_LIMIT") {
      console.error("无法估算Gas（交易可能失败）");
    } else if (error.data) {
      // 解码自定义错误
      try {
        const decodedError = token.interface.parseError(error.data);
        console.error("Custom error:", decodedError.name);
        console.error("Arguments:", decodedError.args);
      } catch {
        console.error("Unknown error:", error.message);
      }
    } else {
      console.error("Error:", error.message);
    }
  }
  
  // 重试逻辑
  async function sendWithRetry(fn, maxRetries = 3) {
    for (let i = 0; i < maxRetries; i++) {
      try {
        return await fn();
      } catch (error) {
        if (i === maxRetries - 1) throw error;
        console.log(`Retry ${i + 1}/${maxRetries}`);
        await new Promise(resolve => setTimeout(resolve, 2000));
      }
    }
  }
  
  const result = await sendWithRetry(() =>
    token.transfer("0x...", ethers.parseEther("1"))
  );
}
```

---

## 📝 今日作业

### 作业1: 事件监控系统

创建实时事件监控：
1. 监听多个合约
2. 过滤重要事件
3. 实时通知
4. 历史数据分析

### 作业2: 批量操作工具

开发批量操作工具：
1. Multicall集成
2. 批量查询
3. 批量交易
4. Gas优化

### 作业3: ENS集成

实现ENS完整支持：
1. 域名解析
2. 反向解析
3. 记录读取
4. 用户友好显示

---

## ✅ 检查清单

- [ ] 掌握Contract操作
- [ ] 理解事件监听
- [ ] 会使用过滤器
- [ ] 掌握批量操作
- [ ] 理解ENS解析

---

## 📅 明日预告

明天学习Web3.js实战：
- Web3.js基础
- 与Ethers.js对比
- 实战案例
- 迁移指南

**🎉 完成Day 4！明天见！**
