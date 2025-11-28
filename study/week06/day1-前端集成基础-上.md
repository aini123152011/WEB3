# Week 6 - Day 1: 前端集成基础 - 上

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 理解Web3前端架构
- ✅ 掌握React基础
- ✅ 学习状态管理
- ✅ 理解Hooks使用
- ✅ 搭建项目基础

---

## Part 1: Web3前端架构 (1.5小时)

### 1.1 架构概览

```
┌─────────────────────────────────────┐
│          User Interface             │
│   (React Components + UI Library)   │
└──────────────┬──────────────────────┘
               │
┌──────────────┴──────────────────────┐
│      State Management Layer         │
│   (Context API / Redux / Zustand)   │
└──────────────┬──────────────────────┘
               │
┌──────────────┴──────────────────────┐
│     Web3 Integration Layer          │
│  (ethers.js / web3.js / wagmi)      │
└──────────────┬──────────────────────┘
               │
┌──────────────┴──────────────────────┐
│       Wallet Connection             │
│  (MetaMask / WalletConnect / etc)   │
└──────────────┬──────────────────────┘
               │
┌──────────────┴──────────────────────┐
│        Blockchain Network           │
│    (Ethereum / Polygon / BSC)       │
└─────────────────────────────────────┘
```

### 1.2 技术栈选择

```markdown
## 推荐技术栈

### 前端框架
- **React** ⭐⭐⭐⭐⭐ (首选)
- Next.js (服务端渲染)
- Vue.js (替代方案)

### Web3库
- **ethers.js** ⭐⭐⭐⭐⭐ (推荐)
- web3.js (传统选择)
- wagmi (React hooks封装)

### 状态管理
- Context API (小型项目)
- **Zustand** ⭐⭐⭐⭐⭐ (中型项目)
- Redux Toolkit (大型项目)

### UI库
- **Material-UI** ⭐⭐⭐⭐⭐
- Ant Design
- Chakra UI (Web3专用)
- Tailwind CSS

### 钱包集成
- **RainbowKit** ⭐⭐⭐⭐⭐ (推荐)
- Web3Modal
- ConnectKit
```

---

## Part 2: React基础回顾 (2小时)

### 2.1 组件基础

```javascript
// 函数组件
import React from 'react';

// 基础函数组件
function Welcome(props) {
  return <h1>Hello, {props.name}!</h1>;
}

// 箭头函数组件（推荐）
const WelcomeArrow = ({ name }) => {
  return <h1>Hello, {name}!</h1>;
};

// 带Props解构的组件
const UserCard = ({ username, address, balance }) => {
  return (
    <div className="user-card">
      <h2>{username}</h2>
      <p>Address: {address}</p>
      <p>Balance: {balance} ETH</p>
    </div>
  );
};

// 带默认Props的组件
const Button = ({ 
  children, 
  variant = 'primary',
  onClick,
  disabled = false 
}) => {
  return (
    <button 
      className={`btn btn-${variant}`}
      onClick={onClick}
      disabled={disabled}
    >
      {children}
    </button>
  );
};

// 使用组件
function App() {
  return (
    <div>
      <Welcome name="Alice" />
      <UserCard 
        username="Bob"
        address="0x..."
        balance="1.5"
      />
      <Button onClick={() => alert('Clicked!')}>
        Connect Wallet
      </Button>
    </div>
  );
}

export default App;
```

### 2.2 JSX语法

```javascript
import React from 'react';

const JsxExamples = () => {
  const user = {
    name: 'Alice',
    address: '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb'
  };
  
  const isConnected = true;
  const balance = 1.5;
  
  // 条件渲染
  const renderStatus = () => {
    if (isConnected) {
      return <span className="status-online">Connected</span>;
    }
    return <span className="status-offline">Disconnected</span>;
  };
  
  return (
    <div className="container">
      {/* 1. 表达式插值 */}
      <h1>Welcome, {user.name}!</h1>
      <p>Address: {user.address.slice(0, 6)}...{user.address.slice(-4)}</p>
      
      {/* 2. 条件渲染 - 三元运算符 */}
      <div>
        {isConnected ? (
          <p>Balance: {balance} ETH</p>
        ) : (
          <p>Please connect your wallet</p>
        )}
      </div>
      
      {/* 3. 条件渲染 - && 操作符 */}
      {isConnected && <p>You are connected!</p>}
      
      {/* 4. 列表渲染 */}
      <ul>
        {['ETH', 'USDC', 'DAI'].map((token, index) => (
          <li key={index}>{token}</li>
        ))}
      </ul>
      
      {/* 5. 样式对象 */}
      <div style={{ 
        backgroundColor: '#f0f0f0', 
        padding: '20px',
        borderRadius: '8px'
      }}>
        <p>Styled content</p>
      </div>
      
      {/* 6. 类名动态绑定 */}
      <button className={`btn ${isConnected ? 'btn-success' : 'btn-primary'}`}>
        {isConnected ? 'Disconnect' : 'Connect'}
      </button>
      
      {/* 7. 函数调用 */}
      {renderStatus()}
    </div>
  );
};

export default JsxExamples;
```

---

## Part 3: React Hooks (2小时)

### 3.1 useState - 状态管理

```javascript
import React, { useState } from 'react';

const WalletConnection = () => {
  // 基础状态
  const [isConnected, setIsConnected] = useState(false);
  const [account, setAccount] = useState('');
  const [balance, setBalance] = useState('0');
  
  // 对象状态
  const [userInfo, setUserInfo] = useState({
    address: '',
    balance: '0',
    network: ''
  });
  
  // 数组状态
  const [transactions, setTransactions] = useState([]);
  
  // 连接钱包
  const connectWallet = async () => {
    if (window.ethereum) {
      try {
        // 请求账户
        const accounts = await window.ethereum.request({
          method: 'eth_requestAccounts'
        });
        
        setAccount(accounts[0]);
        setIsConnected(true);
        
        // 获取余额
        const balanceHex = await window.ethereum.request({
          method: 'eth_getBalance',
          params: [accounts[0], 'latest']
        });
        
        const balanceEth = parseInt(balanceHex, 16) / 1e18;
        setBalance(balanceEth.toFixed(4));
        
      } catch (error) {
        console.error('Connection error:', error);
      }
    } else {
      alert('Please install MetaMask!');
    }
  };
  
  // 断开连接
  const disconnectWallet = () => {
    setAccount('');
    setBalance('0');
    setIsConnected(false);
  };
  
  // 更新用户信息（对象状态）
  const updateUserInfo = (key, value) => {
    setUserInfo(prev => ({
      ...prev,
      [key]: value
    }));
  };
  
  // 添加交易（数组状态）
  const addTransaction = (tx) => {
    setTransactions(prev => [...prev, tx]);
  };
  
  return (
    <div className="wallet-connection">
      {!isConnected ? (
        <button onClick={connectWallet}>Connect Wallet</button>
      ) : (
        <div>
          <p>Address: {account.slice(0, 6)}...{account.slice(-4)}</p>
          <p>Balance: {balance} ETH</p>
          <button onClick={disconnectWallet}>Disconnect</button>
        </div>
      )}
    </div>
  );
};

export default WalletConnection;
```

### 3.2 useEffect - 副作用处理

```javascript
import React, { useState, useEffect } from 'react';
import { ethers } from 'ethers';

const TokenBalance = ({ tokenAddress, userAddress }) => {
  const [balance, setBalance] = useState('0');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  // Effect 1: 初始加载
  useEffect(() => {
    console.log('Component mounted');
    
    // 清理函数
    return () => {
      console.log('Component will unmount');
    };
  }, []); // 空依赖数组 = 只在mount时执行
  
  // Effect 2: 监听地址变化
  useEffect(() => {
    if (!tokenAddress || !userAddress) return;
    
    fetchBalance();
  }, [tokenAddress, userAddress]); // 依赖数组
  
  // Effect 3: 设置定时器
  useEffect(() => {
    // 每10秒更新一次余额
    const interval = setInterval(() => {
      fetchBalance();
    }, 10000);
    
    // 清理定时器
    return () => clearInterval(interval);
  }, [tokenAddress, userAddress]);
  
  // Effect 4: 监听区块链事件
  useEffect(() => {
    if (!window.ethereum) return;
    
    const provider = new ethers.BrowserProvider(window.ethereum);
    
    // 监听账户变化
    const handleAccountsChanged = (accounts) => {
      console.log('Accounts changed:', accounts);
    };
    
    // 监听链变化
    const handleChainChanged = (chainId) => {
      console.log('Chain changed:', chainId);
      window.location.reload();
    };
    
    window.ethereum.on('accountsChanged', handleAccountsChanged);
    window.ethereum.on('chainChanged', handleChainChanged);
    
    // 清理监听器
    return () => {
      window.ethereum.removeListener('accountsChanged', handleAccountsChanged);
      window.ethereum.removeListener('chainChanged', handleChainChanged);
    };
  }, []);
  
  // 获取余额
  const fetchBalance = async () => {
    try {
      setLoading(true);
      setError(null);
      
      const provider = new ethers.BrowserProvider(window.ethereum);
      const contract = new ethers.Contract(
        tokenAddress,
        ['function balanceOf(address) view returns (uint256)'],
        provider
      );
      
      const bal = await contract.balanceOf(userAddress);
      setBalance(ethers.formatEther(bal));
      
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };
  
  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  
  return (
    <div>
      <p>Token Balance: {balance}</p>
    </div>
  );
};

export default TokenBalance;
```

### 3.3 useCallback & useMemo - 性能优化

```javascript
import React, { useState, useCallback, useMemo } from 'react';

const TransactionList = () => {
  const [transactions, setTransactions] = useState([]);
  const [filter, setFilter] = useState('all'); // 'all', 'sent', 'received'
  const [sortBy, setSortBy] = useState('date'); // 'date', 'amount'
  
  // useCallback: 缓存函数，避免子组件不必要的重渲染
  const addTransaction = useCallback((tx) => {
    setTransactions(prev => [...prev, {
      ...tx,
      id: Date.now(),
      timestamp: Date.now()
    }]);
  }, []);
  
  const removeTransaction = useCallback((id) => {
    setTransactions(prev => prev.filter(tx => tx.id !== id));
  }, []);
  
  // useMemo: 缓存计算结果
  const filteredTransactions = useMemo(() => {
    let filtered = transactions;
    
    // 过滤
    if (filter !== 'all') {
      filtered = filtered.filter(tx => tx.type === filter);
    }
    
    // 排序
    filtered.sort((a, b) => {
      if (sortBy === 'date') {
        return b.timestamp - a.timestamp;
      } else {
        return b.amount - a.amount;
      }
    });
    
    console.log('Filtered transactions calculated');
    return filtered;
  }, [transactions, filter, sortBy]);
  
  // 统计数据（使用useMemo缓存）
  const stats = useMemo(() => {
    const totalSent = transactions
      .filter(tx => tx.type === 'sent')
      .reduce((sum, tx) => sum + tx.amount, 0);
      
    const totalReceived = transactions
      .filter(tx => tx.type === 'received')
      .reduce((sum, tx) => sum + tx.amount, 0);
    
    return {
      totalSent,
      totalReceived,
      netBalance: totalReceived - totalSent
    };
  }, [transactions]);
  
  return (
    <div>
      <div className="stats">
        <p>Sent: {stats.totalSent} ETH</p>
        <p>Received: {stats.totalReceived} ETH</p>
        <p>Net: {stats.netBalance} ETH</p>
      </div>
      
      <div className="filters">
        <select value={filter} onChange={(e) => setFilter(e.target.value)}>
          <option value="all">All</option>
          <option value="sent">Sent</option>
          <option value="received">Received</option>
        </select>
        
        <select value={sortBy} onChange={(e) => setSortBy(e.target.value)}>
          <option value="date">Sort by Date</option>
          <option value="amount">Sort by Amount</option>
        </select>
      </div>
      
      <ul>
        {filteredTransactions.map(tx => (
          <li key={tx.id}>
            <span>{tx.type}</span>
            <span>{tx.amount} ETH</span>
            <button onClick={() => removeTransaction(tx.id)}>Remove</button>
          </li>
        ))}
      </ul>
      
      <button onClick={() => addTransaction({
        type: 'sent',
        amount: Math.random() * 10
      })}>
        Add Transaction
      </button>
    </div>
  );
};

export default TransactionList;
```

### 3.4 自定义Hooks

```javascript
// hooks/useWallet.js
import { useState, useEffect, useCallback } from 'react';
import { ethers } from 'ethers';

export const useWallet = () => {
  const [account, setAccount] = useState(null);
  const [provider, setProvider] = useState(null);
  const [signer, setSigner] = useState(null);
  const [chainId, setChainId] = useState(null);
  const [balance, setBalance] = useState('0');
  const [isConnecting, setIsConnecting] = useState(false);
  const [error, setError] = useState(null);
  
  // 连接钱包
  const connect = useCallback(async () => {
    if (!window.ethereum) {
      setError('Please install MetaMask!');
      return;
    }
    
    try {
      setIsConnecting(true);
      setError(null);
      
      // 请求账户
      const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
      });
      
      // 创建provider和signer
      const ethersProvider = new ethers.BrowserProvider(window.ethereum);
      const ethersSigner = await ethersProvider.getSigner();
      const network = await ethersProvider.getNetwork();
      
      setProvider(ethersProvider);
      setSigner(ethersSigner);
      setAccount(accounts[0]);
      setChainId(Number(network.chainId));
      
      // 获取余额
      const bal = await ethersProvider.getBalance(accounts[0]);
      setBalance(ethers.formatEther(bal));
      
    } catch (err) {
      setError(err.message);
    } finally {
      setIsConnecting(false);
    }
  }, []);
  
  // 断开连接
  const disconnect = useCallback(() => {
    setAccount(null);
    setProvider(null);
    setSigner(null);
    setChainId(null);
    setBalance('0');
  }, []);
  
  // 监听账户和网络变化
  useEffect(() => {
    if (!window.ethereum) return;
    
    const handleAccountsChanged = (accounts) => {
      if (accounts.length === 0) {
        disconnect();
      } else {
        setAccount(accounts[0]);
      }
    };
    
    const handleChainChanged = () => {
      window.location.reload();
    };
    
    window.ethereum.on('accountsChanged', handleAccountsChanged);
    window.ethereum.on('chainChanged', handleChainChanged);
    
    return () => {
      window.ethereum.removeListener('accountsChanged', handleAccountsChanged);
      window.ethereum.removeListener('chainChanged', handleChainChanged);
    };
  }, [disconnect]);
  
  // 自动连接（如果之前连接过）
  useEffect(() => {
    const checkConnection = async () => {
      if (!window.ethereum) return;
      
      try {
        const accounts = await window.ethereum.request({
          method: 'eth_accounts'
        });
        
        if (accounts.length > 0) {
          await connect();
        }
      } catch (err) {
        console.error('Auto-connect failed:', err);
      }
    };
    
    checkConnection();
  }, [connect]);
  
  return {
    account,
    provider,
    signer,
    chainId,
    balance,
    isConnecting,
    error,
    connect,
    disconnect,
    isConnected: !!account
  };
};

// hooks/useContract.js
export const useContract = (address, abi) => {
  const { provider, signer } = useWallet();
  const [contract, setContract] = useState(null);
  
  useEffect(() => {
    if (!address || !abi || !provider) return;
    
    const contractInstance = new ethers.Contract(
      address,
      abi,
      signer || provider
    );
    
    setContract(contractInstance);
  }, [address, abi, provider, signer]);
  
  return contract;
};

// hooks/useBalance.js
export const useBalance = (tokenAddress) => {
  const { account, provider } = useWallet();
  const [balance, setBalance] = useState('0');
  const [loading, setLoading] = useState(false);
  
  const fetchBalance = useCallback(async () => {
    if (!account || !provider) return;
    
    try {
      setLoading(true);
      
      if (!tokenAddress) {
        // ETH余额
        const bal = await provider.getBalance(account);
        setBalance(ethers.formatEther(bal));
      } else {
        // ERC20余额
        const contract = new ethers.Contract(
          tokenAddress,
          ['function balanceOf(address) view returns (uint256)'],
          provider
        );
        const bal = await contract.balanceOf(account);
        setBalance(ethers.formatEther(bal));
      }
    } catch (err) {
      console.error('Balance fetch failed:', err);
    } finally {
      setLoading(false);
    }
  }, [account, provider, tokenAddress]);
  
  useEffect(() => {
    fetchBalance();
    
    // 定期更新
    const interval = setInterval(fetchBalance, 10000);
    return () => clearInterval(interval);
  }, [fetchBalance]);
  
  return { balance, loading, refetch: fetchBalance };
};

// 使用自定义Hooks
import React from 'react';
import { useWallet, useBalance } from './hooks';

const WalletInfo = () => {
  const { 
    account, 
    chainId, 
    isConnecting, 
    error, 
    connect, 
    disconnect, 
    isConnected 
  } = useWallet();
  
  const { balance } = useBalance();
  
  if (error) {
    return <div className="error">{error}</div>;
  }
  
  if (!isConnected) {
    return (
      <button onClick={connect} disabled={isConnecting}>
        {isConnecting ? 'Connecting...' : 'Connect Wallet'}
      </button>
    );
  }
  
  return (
    <div className="wallet-info">
      <p>Account: {account.slice(0, 6)}...{account.slice(-4)}</p>
      <p>Chain ID: {chainId}</p>
      <p>Balance: {balance} ETH</p>
      <button onClick={disconnect}>Disconnect</button>
    </div>
  );
};

export default WalletInfo;
```

---

## 📝 今日作业

### 作业1: 钱包连接组件

创建完整的钱包连接组件：
1. 连接/断开功能
2. 显示账户信息
3. 监听账户变化
4. 错误处理

### 作业2: 自定义Hooks

开发实用的自定义Hooks：
1. useWallet
2. useBalance
3. useContract
4. useTransaction

### 作业3: 交易列表

实现交易列表组件：
1. 过滤功能
2. 排序功能
3. 性能优化
4. 数据缓存

---

## ✅ 检查清单

- [ ] 理解Web3前端架构
- [ ] 掌握React基础语法
- [ ] 熟练使用React Hooks
- [ ] 能创建自定义Hooks
- [ ] 理解性能优化

---

## 📅 明日预告

明天继续前端集成基础：
- Context API使用
- 全局状态管理
- 主题切换
- 错误边界

**🎉 完成Day 1！继续加油！**
