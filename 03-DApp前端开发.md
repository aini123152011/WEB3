# 第三章：DApp前端开发 🎨

## 3.1 WEB3前端技术栈

### 完整技术架构

```
┌─────────────────────────────────────────┐
│        前端框架层                        │
│   React / Vue / Next.js / Nuxt.js       │
└──────────────┬──────────────────────────┘
               │
┌──────────────┴──────────────────────────┐
│        WEB3交互层                        │
│   Web3.js / Ethers.js / Viem            │
└──────────────┬──────────────────────────┘
               │
┌──────────────┴──────────────────────────┐
│        钱包连接层                        │
│   MetaMask / WalletConnect / RainbowKit │
└──────────────┬──────────────────────────┘
               │
┌──────────────┴──────────────────────────┐
│        区块链RPC层                       │
│   Infura / Alchemy / QuickNode          │
└──────────────┬──────────────────────────┘
               │
┌──────────────┴──────────────────────────┐
│        区块链网络                        │
│   Ethereum / BSC / Polygon              │
└─────────────────────────────────────────┘
```

### 核心库对比

| 特性           | Web3.js  | Ethers.js | Viem     |
| -------------- | -------- | --------- | -------- |
| **大小**       | ~1.1MB   | ~116KB    | ~60KB    |
| **性能**       | 中等     | 良好      | 优秀     |
| **TypeScript** | 部分支持 | 完全支持  | 完全支持 |
| **学习曲线**   | 陡峭     | 平缓      | 平缓     |
| **社区**       | 最大     | 快速增长  | 新兴     |
| **推荐度**     | ⭐⭐⭐      | ⭐⭐⭐⭐⭐     | ⭐⭐⭐⭐     |

**推荐选择**：Ethers.js（平衡了性能、易用性和生态）

## 3.2 Ethers.js 完全指南

### 安装和基本设置

```bash
# 安装 ethers.js
npm install ethers

# 或使用 yarn
yarn add ethers
```

### 3.2.1 连接到网络

```javascript
import { ethers } from 'ethers';

// 方式1: 连接到MetaMask等浏览器钱包
const provider = new ethers.BrowserProvider(window.ethereum);

// 方式2: 使用RPC URL连接
const provider = new ethers.JsonRpcProvider('https://eth-sepolia.g.alchemy.com/v2/YOUR-API-KEY');

// 方式3: 使用Infura
const provider = new ethers.InfuraProvider('sepolia', 'YOUR-INFURA-PROJECT-ID');

// 获取网络信息
const network = await provider.getNetwork();
console.log('Network:', network.name);
console.log('Chain ID:', network.chainId);

// 获取当前区块号
const blockNumber = await provider.getBlockNumber();
console.log('Current block:', blockNumber);
```

### 3.2.2 钱包管理

```javascript
// 1. 从私钥创建钱包
const wallet = new ethers.Wallet('0xYOUR_PRIVATE_KEY', provider);

// 2. 生成随机钱包
const randomWallet = ethers.Wallet.createRandom();
console.log('Address:', randomWallet.address);
console.log('Mnemonic:', randomWallet.mnemonic.phrase);
console.log('Private Key:', randomWallet.privateKey);

// 3. 从助记词恢复钱包
const mnemonic = 'test test test test test test test test test test test junk';
const walletFromMnemonic = ethers.Wallet.fromPhrase(mnemonic);

// 4. 连接钱包到provider
const connectedWallet = wallet.connect(provider);

// 5. 获取钱包余额
const balance = await provider.getBalance(wallet.address);
console.log('Balance:', ethers.formatEther(balance), 'ETH');
```

### 3.2.3 发送交易

```javascript
// 1. 简单的ETH转账
async function sendEther(to, amount) {
  const tx = await wallet.sendTransaction({
    to: to,
    value: ethers.parseEther(amount)  // 转换ETH到Wei
  });
  
  console.log('Transaction hash:', tx.hash);
  
  // 等待交易确认
  const receipt = await tx.wait();
  console.log('Transaction confirmed in block:', receipt.blockNumber);
  
  return receipt;
}

// 使用示例
await sendEther('0xRecipientAddress', '0.1');

// 2. 带数据的交易
const tx = await wallet.sendTransaction({
  to: '0xContractAddress',
  value: ethers.parseEther('0.01'),
  gasLimit: 100000,
  data: '0x...'  // 合约调用数据
});

// 3. 估算Gas
const gasEstimate = await provider.estimateGas({
  to: '0xAddress',
  value: ethers.parseEther('0.1')
});
console.log('Estimated gas:', gasEstimate.toString());
```

### 3.2.4 与智能合约交互

```javascript
// 合约ABI（从编译后的合约中获取）
const contractABI = [
  "function balanceOf(address owner) view returns (uint256)",
  "function transfer(address to, uint256 amount) returns (bool)",
  "event Transfer(address indexed from, address indexed to, uint256 amount)"
];

const contractAddress = '0xYourContractAddress';

// 创建合约实例
const contract = new ethers.Contract(contractAddress, contractABI, provider);

// 只读调用（不需要Gas）
const balance = await contract.balanceOf('0xUserAddress');
console.log('Balance:', ethers.formatUnits(balance, 18));

// 写入调用（需要签名和Gas）
const contractWithSigner = contract.connect(wallet);
const tx = await contractWithSigner.transfer('0xRecipient', ethers.parseUnits('100', 18));
await tx.wait();

// 监听事件
contract.on('Transfer', (from, to, amount, event) => {
  console.log(`Transfer from ${from} to ${to}: ${ethers.formatUnits(amount, 18)}`);
});

// 查询历史事件
const filter = contract.filters.Transfer(null, '0xMyAddress');
const events = await contract.queryFilter(filter, -10000);  // 最近10000个区块
events.forEach(event => {
  console.log('Event:', event.args);
});
```

### 3.2.5 完整的ERC-20代币交互示例

```javascript
// ERC-20 标准接口
const ERC20_ABI = [
  "function name() view returns (string)",
  "function symbol() view returns (string)",
  "function decimals() view returns (uint8)",
  "function totalSupply() view returns (uint256)",
  "function balanceOf(address) view returns (uint256)",
  "function transfer(address to, uint256 amount) returns (bool)",
  "function approve(address spender, uint256 amount) returns (bool)",
  "function allowance(address owner, address spender) view returns (uint256)",
  "function transferFrom(address from, address to, uint256 amount) returns (bool)",
  "event Transfer(address indexed from, address indexed to, uint256 value)",
  "event Approval(address indexed owner, address indexed spender, uint256 value)"
];

class TokenManager {
  constructor(tokenAddress, signerOrProvider) {
    this.contract = new ethers.Contract(tokenAddress, ERC20_ABI, signerOrProvider);
  }
  
  // 获取代币信息
  async getTokenInfo() {
    const [name, symbol, decimals, totalSupply] = await Promise.all([
      this.contract.name(),
      this.contract.symbol(),
      this.contract.decimals(),
      this.contract.totalSupply()
    ]);
    
    return {
      name,
      symbol,
      decimals,
      totalSupply: ethers.formatUnits(totalSupply, decimals)
    };
  }
  
  // 获取余额
  async getBalance(address) {
    const balance = await this.contract.balanceOf(address);
    const decimals = await this.contract.decimals();
    return ethers.formatUnits(balance, decimals);
  }
  
  // 转账
  async transfer(to, amount) {
    const decimals = await this.contract.decimals();
    const tx = await this.contract.transfer(to, ethers.parseUnits(amount, decimals));
    return await tx.wait();
  }
  
  // 授权
  async approve(spender, amount) {
    const decimals = await this.contract.decimals();
    const tx = await this.contract.approve(spender, ethers.parseUnits(amount, decimals));
    return await tx.wait();
  }
  
  // 查询授权额度
  async getAllowance(owner, spender) {
    const allowance = await this.contract.allowance(owner, spender);
    const decimals = await this.contract.decimals();
    return ethers.formatUnits(allowance, decimals);
  }
}

// 使用示例
const tokenManager = new TokenManager('0xTokenAddress', wallet);
const info = await tokenManager.getTokenInfo();
console.log('Token Info:', info);
```

## 3.3 React集成示例

### 3.3.1 创建WEB3上下文

```typescript
// contexts/Web3Context.tsx
import React, { createContext, useContext, useState, useEffect } from 'react';
import { ethers } from 'ethers';

interface Web3ContextType {
  provider: ethers.BrowserProvider | null;
  signer: ethers.Signer | null;
  account: string | null;
  chainId: number | null;
  connect: () => Promise<void>;
  disconnect: () => void;
}

const Web3Context = createContext<Web3ContextType | undefined>(undefined);

export function Web3Provider({ children }: { children: React.ReactNode }) {
  const [provider, setProvider] = useState<ethers.BrowserProvider | null>(null);
  const [signer, setSigner] = useState<ethers.Signer | null>(null);
  const [account, setAccount] = useState<string | null>(null);
  const [chainId, setChainId] = useState<number | null>(null);
  
  // 连接钱包
  const connect = async () => {
    if (typeof window.ethereum === 'undefined') {
      alert('Please install MetaMask!');
      return;
    }
    
    try {
      const provider = new ethers.BrowserProvider(window.ethereum);
      const accounts = await provider.send('eth_requestAccounts', []);
      const signer = await provider.getSigner();
      const network = await provider.getNetwork();
      
      setProvider(provider);
      setSigner(signer);
      setAccount(accounts[0]);
      setChainId(Number(network.chainId));
    } catch (error) {
      console.error('Error connecting wallet:', error);
    }
  };
  
  // 断开连接
  const disconnect = () => {
    setProvider(null);
    setSigner(null);
    setAccount(null);
    setChainId(null);
  };
  
  // 监听账户变化
  useEffect(() => {
    if (window.ethereum) {
      window.ethereum.on('accountsChanged', (accounts: string[]) => {
        if (accounts.length === 0) {
          disconnect();
        } else {
          setAccount(accounts[0]);
        }
      });
      
      window.ethereum.on('chainChanged', () => {
        window.location.reload();
      });
    }
    
    return () => {
      if (window.ethereum) {
        window.ethereum.removeAllListeners();
      }
    };
  }, []);
  
  return (
    <Web3Context.Provider value={{ provider, signer, account, chainId, connect, disconnect }}>
      {children}
    </Web3Context.Provider>
  );
}

export function useWeb3() {
  const context = useContext(Web3Context);
  if (!context) {
    throw new Error('useWeb3 must be used within Web3Provider');
  }
  return context;
}
```

### 3.3.2 钱包连接组件

```typescript
// components/WalletConnect.tsx
import React from 'react';
import { useWeb3 } from '../contexts/Web3Context';

export function WalletConnect() {
  const { account, chainId, connect, disconnect } = useWeb3();
  
  const formatAddress = (address: string) => {
    return `${address.slice(0, 6)}...${address.slice(-4)}`;
  };
  
  const getNetworkName = (chainId: number) => {
    const networks: Record<number, string> = {
      1: 'Ethereum',
      5: 'Goerli',
      11155111: 'Sepolia',
      56: 'BSC',
      137: 'Polygon'
    };
    return networks[chainId] || 'Unknown';
  };
  
  return (
    <div className="wallet-connect">
      {!account ? (
        <button onClick={connect} className="connect-button">
          Connect Wallet
        </button>
      ) : (
        <div className="wallet-info">
          <span className="network-badge">
            {chainId && getNetworkName(chainId)}
          </span>
          <span className="address">{formatAddress(account)}</span>
          <button onClick={disconnect} className="disconnect-button">
            Disconnect
          </button>
        </div>
      )}
    </div>
  );
}
```

### 3.3.3 代币余额查询组件

```typescript
// components/TokenBalance.tsx
import React, { useState, useEffect } from 'react';
import { ethers } from 'ethers';
import { useWeb3 } from '../contexts/Web3Context';

const ERC20_ABI = [
  "function balanceOf(address) view returns (uint256)",
  "function decimals() view returns (uint8)",
  "function symbol() view returns (string)"
];

export function TokenBalance({ tokenAddress }: { tokenAddress: string }) {
  const { provider, account } = useWeb3();
  const [balance, setBalance] = useState<string>('0');
  const [symbol, setSymbol] = useState<string>('');
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    if (!provider || !account || !tokenAddress) return;
    
    const fetchBalance = async () => {
      setLoading(true);
      try {
        const contract = new ethers.Contract(tokenAddress, ERC20_ABI, provider);
        const [balance, decimals, symbol] = await Promise.all([
          contract.balanceOf(account),
          contract.decimals(),
          contract.symbol()
        ]);
        
        setBalance(ethers.formatUnits(balance, decimals));
        setSymbol(symbol);
      } catch (error) {
        console.error('Error fetching balance:', error);
      } finally {
        setLoading(false);
      }
    };
    
    fetchBalance();
    
    // 每15秒刷新一次
    const interval = setInterval(fetchBalance, 15000);
    return () => clearInterval(interval);
  }, [provider, account, tokenAddress]);
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <div className="token-balance">
      <h3>Token Balance</h3>
      <p>{balance} {symbol}</p>
    </div>
  );
}
```

### 3.3.4 代币转账组件

```typescript
// components/TokenTransfer.tsx
import React, { useState } from 'react';
import { ethers } from 'ethers';
import { useWeb3 } from '../contexts/Web3Context';

const ERC20_ABI = [
  "function transfer(address to, uint256 amount) returns (bool)",
  "function decimals() view returns (uint8)"
];

export function TokenTransfer({ tokenAddress }: { tokenAddress: string }) {
  const { signer } = useWeb3();
  const [recipient, setRecipient] = useState('');
  const [amount, setAmount] = useState('');
  const [loading, setLoading] = useState(false);
  const [txHash, setTxHash] = useState('');
  
  const handleTransfer = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!signer || !recipient || !amount) return;
    
    setLoading(true);
    setTxHash('');
    
    try {
      const contract = new ethers.Contract(tokenAddress, ERC20_ABI, signer);
      const decimals = await contract.decimals();
      
      const tx = await contract.transfer(
        recipient,
        ethers.parseUnits(amount, decimals)
      );
      
      setTxHash(tx.hash);
      await tx.wait();
      
      alert('Transfer successful!');
      setRecipient('');
      setAmount('');
    } catch (error: any) {
      console.error('Transfer error:', error);
      alert(error.message || 'Transfer failed');
    } finally {
      setLoading(false);
    }
  };
  
  return (
    <div className="token-transfer">
      <h3>Send Tokens</h3>
      <form onSubmit={handleTransfer}>
        <input
          type="text"
          placeholder="Recipient Address"
          value={recipient}
          onChange={(e) => setRecipient(e.target.value)}
          required
        />
        <input
          type="number"
          placeholder="Amount"
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
          step="0.000001"
          min="0"
          required
        />
        <button type="submit" disabled={loading}>
          {loading ? 'Sending...' : 'Send'}
        </button>
      </form>
      {txHash && (
        <div className="tx-info">
          <p>Transaction Hash:</p>
          <a 
            href={`https://etherscan.io/tx/${txHash}`}
            target="_blank"
            rel="noopener noreferrer"
          >
            {txHash}
          </a>
        </div>
      )}
    </div>
  );
}
```

## 3.4 常用钱包集成方案

### 3.4.1 RainbowKit集成（推荐）

```bash
npm install @rainbow-me/rainbowkit wagmi viem@2.x @tanstack/react-query
```

```typescript
// app.tsx
import '@rainbow-me/rainbowkit/styles.css';
import { getDefaultConfig, RainbowKitProvider } from '@rainbow-me/rainbowkit';
import { WagmiProvider } from 'wagmi';
import { mainnet, polygon, optimism, arbitrum } from 'wagmi/chains';
import { QueryClientProvider, QueryClient } from '@tanstack/react-query';

const config = getDefaultConfig({
  appName: 'My DApp',
  projectId: 'YOUR_WALLETCONNECT_PROJECT_ID',
  chains: [mainnet, polygon, optimism, arbitrum],
});

const queryClient = new QueryClient();

function App() {
  return (
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        <RainbowKitProvider>
          {/* Your App */}
        </RainbowKitProvider>
      </QueryClientProvider>
    </WagmiProvider>
  );
}
```

### 3.4.2 WalletConnect集成

```typescript
import { WalletConnect } from '@walletconnect/web3-provider';

const provider = new WalletConnect({
  infuraId: 'YOUR_INFURA_ID',
  qrcode: true,
});

await provider.enable();
const ethersProvider = new ethers.BrowserProvider(provider);
```

## 3.5 实用工具函数

```typescript
// utils/web3Utils.ts

// 格式化地址
export function formatAddress(address: string, length = 4): string {
  return `${address.slice(0, length + 2)}...${address.slice(-length)}`;
}

// 格式化金额
export function formatAmount(amount: string, decimals = 4): string {
  const num = parseFloat(amount);
  return num.toFixed(decimals);
}

// 验证地址
export function isValidAddress(address: string): boolean {
  return ethers.isAddress(address);
}

// 格式化交易哈希
export function getEtherscanLink(
  chainId: number,
  hash: string,
  type: 'tx' | 'address' = 'tx'
): string {
  const networks: Record<number, string> = {
    1: 'etherscan.io',
    5: 'goerli.etherscan.io',
    11155111: 'sepolia.etherscan.io',
    56: 'bscscan.com',
    137: 'polygonscan.com',
  };
  
  const baseUrl = networks[chainId] || 'etherscan.io';
  return `https://${baseUrl}/${type}/${hash}`;
}

// 等待交易确认
export async function waitForTransaction(
  provider: ethers.Provider,
  txHash: string,
  confirmations = 1
): Promise<ethers.TransactionReceipt> {
  console.log(`Waiting for transaction ${txHash}...`);
  const receipt = await provider.waitForTransaction(txHash, confirmations);
  console.log(`Transaction confirmed in block ${receipt.blockNumber}`);
  return receipt;
}

// 错误处理
export function handleError(error: any): string {
  if (error.code === 'ACTION_REJECTED') {
    return 'Transaction rejected by user';
  }
  if (error.code === 'INSUFFICIENT_FUNDS') {
    return 'Insufficient funds for transaction';
  }
  if (error.code === 'NETWORK_ERROR') {
    return 'Network error. Please check your connection';
  }
  return error.message || 'Unknown error occurred';
}
```

---

## 📚 本章小结

DApp前端开发需要掌握Web3库的使用、钱包集成、合约交互等关键技能。Ethers.js提供了强大而易用的API，配合React等现代框架，可以快速构建用户友好的去中心化应用。

**下一章预告**：我们将学习智能合约的测试与部署流程 🚀

---

> 💡 **最佳实践**：始终在测试网上充分测试，处理所有可能的错误情况，提供良好的用户反馈。
