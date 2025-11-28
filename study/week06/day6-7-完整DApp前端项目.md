# Week 6 - Day 6-7: 完整DApp前端项目

**学习日期**: ___________
**预计用时**: 12-14小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 项目目标

构建一个完整的多链DeFi Dashboard：
- ✅ 集成多种钱包（MetaMask, WalletConnect）
- ✅ 支持多链切换（ETH, BSC, Polygon）
- ✅ 实时显示代币余额和价格
- ✅ 实现代币转账功能
- ✅ 完整的UI/UX设计
- ✅ 响应式移动端适配

---

## Part 1: 项目初始化 (2小时)

### 1.1 创建项目

使用Vite创建React项目：

```bash
npm create vite@latest defi-dashboard -- --template react
cd defi-dashboard
npm install
```

### 1.2 安装依赖

```bash
# Web3相关
npm install ethers @web3modal/ethers @web3modal/react

# 状态管理
npm install zustand

# 路由
npm install react-router-dom

# UI工具
npm install clsx tailwind-merge
```

### 1.3 目录结构

```
src/
├── components/         # UI组件
│   ├── Button/
│   ├── Card/
│   ├── Modal/
│   └── Layout/
├── config/            # 配置文件
│   ├── chains.js
│   ├── contracts.js
│   └── web3modal.js
├── contexts/          # React Contexts
│   └── ThemeContext.jsx
├── hooks/             # 自定义Hooks
│   ├── useWallet.js
│   ├── useToken.js
│   └── usePrice.js
├── pages/             # 页面组件
│   ├── Dashboard/
│   ├── Transfer/
│   └── Settings/
├── stores/            # Zustand Stores
│   └── useStore.js
├── utils/             # 工具函数
│   ├── format.js
│   └── network.js
└── App.jsx
```

---

## Part 2: 核心配置与状态管理 (2小时)

### 2.1 链配置

```javascript
// src/config/chains.js
export const chains = [
  {
    chainId: 1,
    name: 'Ethereum',
    currency: 'ETH',
    explorerUrl: 'https://etherscan.io',
    rpcUrl: 'https://cloudflare-eth.com',
    icon: '/icons/eth.svg'
  },
  {
    chainId: 137,
    name: 'Polygon',
    currency: 'MATIC',
    explorerUrl: 'https://polygonscan.com',
    rpcUrl: 'https://polygon-rpc.com',
    icon: '/icons/matic.svg'
  },
  {
    chainId: 56,
    name: 'BSC',
    currency: 'BNB',
    explorerUrl: 'https://bscscan.com',
    rpcUrl: 'https://bsc-dataseed.binance.org/',
    icon: '/icons/bnb.svg'
  }
];

export const getChain = (chainId) => chains.find(c => c.chainId === chainId);
```

### 2.2 全局状态 Store

使用Zustand管理应用状态。

```javascript
// src/stores/useStore.js
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useStore = create(
  persist(
    (set) => ({
      // 主题设置
      theme: 'light',
      toggleTheme: () => set((state) => ({ 
        theme: state.theme === 'light' ? 'dark' : 'light' 
      })),
      
      // 交易历史
      transactions: [],
      addTransaction: (tx) => set((state) => ({ 
        transactions: [tx, ...state.transactions] 
      })),
      
      // 用户偏好
      settings: {
        currency: 'USD',
        language: 'en'
      },
      updateSettings: (newSettings) => set((state) => ({
        settings: { ...state.settings, ...newSettings }
      }))
    }),
    {
      name: 'defi-dashboard-storage'
    }
  )
);
```

---

## Part 3: 钱包集成模块 (2小时)

### 3.1 钱包Hook封装

```javascript
// src/hooks/useWallet.js
import { useWeb3Modal, useWeb3ModalAccount, useWeb3ModalProvider } from '@web3modal/ethers/react';
import { ethers } from 'ethers';
import { useState, useEffect } from 'react';

export const useWallet = () => {
  const { open, close } = useWeb3Modal();
  const { address, chainId, isConnected } = useWeb3ModalAccount();
  const { walletProvider } = useWeb3ModalProvider();
  
  const [provider, setProvider] = useState(null);
  const [signer, setSigner] = useState(null);
  const [balance, setBalance] = useState('0');
  
  // 初始化Provider
  useEffect(() => {
    if (walletProvider) {
      const ethersProvider = new ethers.BrowserProvider(walletProvider);
      setProvider(ethersProvider);
      
      ethersProvider.getSigner().then(setSigner);
    } else {
      setProvider(null);
      setSigner(null);
    }
  }, [walletProvider]);
  
  // 获取余额
  useEffect(() => {
    if (provider && address) {
      provider.getBalance(address).then(bal => {
        setBalance(ethers.formatEther(bal));
      });
      
      // 监听区块更新余额
      const handleBlock = () => {
        provider.getBalance(address).then(bal => {
          setBalance(ethers.formatEther(bal));
        });
      };
      
      provider.on('block', handleBlock);
      return () => provider.off('block', handleBlock);
    }
  }, [provider, address, chainId]);
  
  return {
    open,
    close,
    address,
    chainId,
    isConnected,
    provider,
    signer,
    balance
  };
};
```

### 3.2 连接按钮组件

```javascript
// src/components/ConnectButton.jsx
import React from 'react';
import { useWallet } from '../hooks/useWallet';
import { formatAddress } from '../utils/format';

export const ConnectButton = () => {
  const { open, isConnected, address, balance } = useWallet();
  
  if (isConnected) {
    return (
      <button 
        onClick={() => open()}
        className="flex items-center gap-2 px-4 py-2 bg-blue-100 text-blue-600 rounded-lg hover:bg-blue-200 transition-colors"
      >
        <span className="font-bold">{parseFloat(balance).toFixed(4)} ETH</span>
        <span className="px-2 py-1 bg-white rounded-md text-sm">
          {formatAddress(address)}
        </span>
      </button>
    );
  }
  
  return (
    <button 
      onClick={() => open()}
      className="px-6 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors font-semibold"
    >
      Connect Wallet
    </button>
  );
};
```

---

## Part 4: 代币与余额模块 (2小时)

### 4.1 代币列表配置

```javascript
// src/config/tokens.js
export const tokens = {
  1: [ // Ethereum
    { symbol: 'USDT', address: '0xdAC17F958D2ee523a2206206994597C13D831ec7', decimals: 6 },
    { symbol: 'USDC', address: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', decimals: 6 },
    { symbol: 'DAI', address: '0x6B175474E89094C44Da98b954EedeAC495271d0F', decimals: 18 }
  ],
  137: [ // Polygon
    { symbol: 'USDT', address: '0xc2132D05D31c914a87C6611C10748AEb04B58e8F', decimals: 6 },
    { symbol: 'USDC', address: '0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174', decimals: 6 }
  ]
};
```

### 4.2 代币余额Hook

```javascript
// src/hooks/useTokenBalances.js
import { useState, useEffect } from 'react';
import { ethers } from 'ethers';
import { tokens } from '../config/tokens';

const ERC20_ABI = [
  'function balanceOf(address) view returns (uint256)',
  'function decimals() view returns (uint8)'
];

export const useTokenBalances = (address, chainId, provider) => {
  const [balances, setBalances] = useState([]);
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    if (!address || !chainId || !provider) return;
    
    const fetchBalances = async () => {
      setLoading(true);
      const chainTokens = tokens[chainId] || [];
      
      try {
        const results = await Promise.all(chainTokens.map(async (token) => {
          const contract = new ethers.Contract(token.address, ERC20_ABI, provider);
          const balance = await contract.balanceOf(address);
          
          return {
            ...token,
            balance: ethers.formatUnits(balance, token.decimals)
          };
        }));
        
        setBalances(results);
      } catch (error) {
        console.error('Failed to fetch token balances:', error);
      } finally {
        setLoading(false);
      }
    };
    
    fetchBalances();
    const interval = setInterval(fetchBalances, 15000);
    
    return () => clearInterval(interval);
  }, [address, chainId, provider]);
  
  return { balances, loading };
};
```

---

## Part 5: 转账功能模块 (2小时)

### 5.1 转账组件

```javascript
// src/components/TransferCard.jsx
import React, { useState } from 'react';
import { ethers } from 'ethers';
import { useWallet } from '../hooks/useWallet';
import { useNotification } from '../contexts/NotificationContext';

const ERC20_ABI = ['function transfer(address to, uint256 amount) returns (bool)'];

export const TransferCard = () => {
  const { signer, chainId } = useWallet();
  const { showNotification } = useNotification();
  
  const [to, setTo] = useState('');
  const [amount, setAmount] = useState('');
  const [token, setToken] = useState('ETH'); // 'ETH' or token address
  const [loading, setLoading] = useState(false);
  
  const handleTransfer = async (e) => {
    e.preventDefault();
    if (!signer) return;
    
    setLoading(true);
    try {
      let tx;
      
      if (token === 'ETH') {
        // 发送原生代币
        tx = await signer.sendTransaction({
          to,
          value: ethers.parseEther(amount)
        });
      } else {
        // 发送ERC20
        const contract = new ethers.Contract(token, ERC20_ABI, signer);
        // 注意：这里假设是18位精度，实际应从配置获取
        tx = await contract.transfer(to, ethers.parseEther(amount));
      }
      
      showNotification('Transaction sent: ' + tx.hash, 'info');
      await tx.wait();
      showNotification('Transaction confirmed!', 'success');
      
      // 重置表单
      setTo('');
      setAmount('');
      
    } catch (error) {
      console.error(error);
      showNotification(error.message, 'error');
    } finally {
      setLoading(false);
    }
  };
  
  return (
    <div className="bg-white p-6 rounded-xl shadow-sm border border-gray-100">
      <h2 className="text-xl font-bold mb-4">Transfer</h2>
      
      <form onSubmit={handleTransfer} className="space-y-4">
        <div>
          <label className="block text-sm text-gray-600 mb-1">Token</label>
          <select 
            value={token}
            onChange={(e) => setToken(e.target.value)}
            className="w-full p-2 border rounded-lg"
          >
            <option value="ETH">Native Token (ETH/MATIC/BNB)</option>
            {/* 映射当前链的代币列表 */}
          </select>
        </div>
        
        <div>
          <label className="block text-sm text-gray-600 mb-1">Recipient</label>
          <input
            type="text"
            value={to}
            onChange={(e) => setTo(e.target.value)}
            placeholder="0x..."
            className="w-full p-2 border rounded-lg"
            required
          />
        </div>
        
        <div>
          <label className="block text-sm text-gray-600 mb-1">Amount</label>
          <input
            type="number"
            step="0.000001"
            value={amount}
            onChange={(e) => setAmount(e.target.value)}
            placeholder="0.0"
            className="w-full p-2 border rounded-lg"
            required
          />
        </div>
        
        <button
          type="submit"
          disabled={loading || !signer}
          className="w-full py-3 bg-blue-600 text-white rounded-lg font-semibold disabled:opacity-50"
        >
          {loading ? 'Confirming...' : 'Send'}
        </button>
      </form>
    </div>
  );
};
```

---

## Part 6: 页面整合 (2小时)

### 6.1 Dashboard页面

```javascript
// src/pages/Dashboard.jsx
import React from 'react';
import { useWallet } from '../hooks/useWallet';
import { useTokenBalances } from '../hooks/useTokenBalances';
import { ConnectButton } from '../components/ConnectButton';
import { TransferCard } from '../components/TransferCard';

export const Dashboard = () => {
  const { address, chainId, provider, isConnected } = useWallet();
  const { balances, loading } = useTokenBalances(address, chainId, provider);
  
  if (!isConnected) {
    return (
      <div className="flex flex-col items-center justify-center min-h-[60vh]">
        <h1 className="text-3xl font-bold mb-4">Welcome to DeFi Dashboard</h1>
        <p className="text-gray-500 mb-8">Connect your wallet to manage your assets</p>
        <ConnectButton />
      </div>
    );
  }
  
  return (
    <div className="container mx-auto px-4 py-8">
      <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
        {/* 左侧：资产列表 */}
        <div className="md:col-span-2 space-y-6">
          <div className="bg-white p-6 rounded-xl shadow-sm border border-gray-100">
            <h2 className="text-xl font-bold mb-4">Your Assets</h2>
            
            {loading ? (
              <div className="animate-pulse space-y-4">
                {[1, 2, 3].map(i => (
                  <div key={i} className="h-16 bg-gray-100 rounded-lg" />
                ))}
              </div>
            ) : (
              <div className="space-y-4">
                {balances.map(token => (
                  <div key={token.symbol} className="flex items-center justify-between p-4 bg-gray-50 rounded-lg">
                    <div className="flex items-center gap-3">
                      <div className="w-10 h-10 bg-blue-100 rounded-full flex items-center justify-center text-blue-600 font-bold">
                        {token.symbol[0]}
                      </div>
                      <div>
                        <div className="font-bold">{token.symbol}</div>
                        <div className="text-sm text-gray-500">Balance</div>
                      </div>
                    </div>
                    <div className="text-right">
                      <div className="font-bold text-lg">{parseFloat(token.balance).toFixed(4)}</div>
                      <div className="text-sm text-gray-500">
                        ≈ ${(parseFloat(token.balance) * 1.0).toFixed(2)}
                      </div>
                    </div>
                  </div>
                ))}
                
                {balances.length === 0 && (
                  <p className="text-center text-gray-500 py-8">No tokens found</p>
                )}
              </div>
            )}
          </div>
          
          {/* 交易历史组件 */}
          <TransactionHistory />
        </div>
        
        {/* 右侧：转账功能 */}
        <div className="space-y-6">
          <TransferCard />
          
          {/* 网络信息卡片 */}
          <div className="bg-blue-50 p-6 rounded-xl border border-blue-100">
            <h3 className="font-bold text-blue-900 mb-2">Network Info</h3>
            <div className="space-y-2 text-sm text-blue-800">
              <div className="flex justify-between">
                <span>Chain ID:</span>
                <span className="font-mono">{chainId}</span>
              </div>
              <div className="flex justify-between">
                <span>Block Number:</span>
                <span className="font-mono">Loading...</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
};
```

---

## Part 7: 部署与测试 (1.5小时)

### 7.1 环境变量

```env
# .env
VITE_PROJECT_ID=your_walletconnect_project_id
VITE_INFURA_KEY=your_infura_key
```

### 7.2 构建配置

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  define: {
    global: 'globalThis', // 解决Web3库的兼容性问题
  },
  resolve: {
    alias: {
      process: 'process/browser',
      util: 'util',
    },
  },
})
```

---

## 📝 项目总结

### 核心功能
1. **多链支持**: 自动识别并切换网络
2. **多钱包**: 支持MetaMask和WalletConnect
3. **资产管理**: 实时余额查询和估值
4. **交互功能**: 顺畅的转账体验

### 优化点
- 状态持久化 (Zustand + LocalStorage)
- 错误边界处理
- 响应式布局
- Loading骨架屏

---

## ✅ 检查清单

- [ ] 项目结构搭建完成
- [ ] 钱包连接功能正常
- [ ] 代币余额显示正确
- [ ] 转账功能测试通过
- [ ] 多链切换顺畅
- [ ] 移动端适配完成

---

## 📅 下周预告

下周进入React DApp深度开发：
- React Hooks深度应用
- 复杂状态管理
- UI组件库集成
- 实时数据更新策略

**🎉 恭喜完成Week 6！你已经掌握了DApp前端开发的核心技能！**
