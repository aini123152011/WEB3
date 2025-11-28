# Week 6 - Day 2: 前端集成基础 - 下

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 掌握Context API
- ✅ 理解全局状态管理
- ✅ 实现主题切换
- ✅ 学习错误边界
- ✅ 掌握React Router

---

## Part 1: Context API (2小时)

### 1.1 基础用法

```javascript
// contexts/WalletContext.js
import React, { createContext, useState, useContext, useEffect, useCallback } from 'react';
import { ethers } from 'ethers';

// 创建Context
const WalletContext = createContext(undefined);

// Provider组件
export const WalletProvider = ({ children }) => {
  const [account, setAccount] = useState(null);
  const [provider, setProvider] = useState(null);
  const [signer, setSigner] = useState(null);
  const [chainId, setChainId] = useState(null);
  const [balance, setBalance] = useState('0');
  const [isConnecting, setIsConnecting] = useState(false);
  const [error, setError] = useState(null);
  
  // 连接钱包
  const connectWallet = useCallback(async () => {
    if (!window.ethereum) {
      setError('Please install MetaMask!');
      return;
    }
    
    try {
      setIsConnecting(true);
      setError(null);
      
      const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
      });
      
      const ethersProvider = new ethers.BrowserProvider(window.ethereum);
      const ethersSigner = await ethersProvider.getSigner();
      const network = await ethersProvider.getNetwork();
      
      setAccount(accounts[0]);
      setProvider(ethersProvider);
      setSigner(ethersSigner);
      setChainId(Number(network.chainId));
      
      // 获取余额
      const bal = await ethersProvider.getBalance(accounts[0]);
      setBalance(ethers.formatEther(bal));
      
      // 保存连接状态
      localStorage.setItem('walletConnected', 'true');
      
    } catch (err) {
      setError(err.message);
      console.error('Connect error:', err);
    } finally {
      setIsConnecting(false);
    }
  }, []);
  
  // 断开连接
  const disconnectWallet = useCallback(() => {
    setAccount(null);
    setProvider(null);
    setSigner(null);
    setChainId(null);
    setBalance('0');
    localStorage.removeItem('walletConnected');
  }, []);
  
  // 切换网络
  const switchNetwork = useCallback(async (targetChainId) => {
    try {
      await window.ethereum.request({
        method: 'wallet_switchEthereumChain',
        params: [{ chainId: `0x${targetChainId.toString(16)}` }]
      });
    } catch (error) {
      // 如果网络不存在，尝试添加
      if (error.code === 4902) {
        const networkConfig = getNetworkConfig(targetChainId);
        await window.ethereum.request({
          method: 'wallet_addEthereumChain',
          params: [networkConfig]
        });
      }
      throw error;
    }
  }, []);
  
  // 监听账户和网络变化
  useEffect(() => {
    if (!window.ethereum) return;
    
    const handleAccountsChanged = (accounts) => {
      if (accounts.length === 0) {
        disconnectWallet();
      } else {
        setAccount(accounts[0]);
        // 重新获取余额
        if (provider) {
          provider.getBalance(accounts[0]).then(bal => {
            setBalance(ethers.formatEther(bal));
          });
        }
      }
    };
    
    const handleChainChanged = (chainIdHex) => {
      setChainId(parseInt(chainIdHex, 16));
      window.location.reload();
    };
    
    window.ethereum.on('accountsChanged', handleAccountsChanged);
    window.ethereum.on('chainChanged', handleChainChanged);
    
    return () => {
      window.ethereum.removeListener('accountsChanged', handleAccountsChanged);
      window.ethereum.removeListener('chainChanged', handleChainChanged);
    };
  }, [provider, disconnectWallet]);
  
  // 自动连接
  useEffect(() => {
    const wasConnected = localStorage.getItem('walletConnected');
    if (wasConnected === 'true' && window.ethereum) {
      window.ethereum.request({ method: 'eth_accounts' })
        .then(accounts => {
          if (accounts.length > 0) {
            connectWallet();
          }
        });
    }
  }, [connectWallet]);
  
  const value = {
    account,
    provider,
    signer,
    chainId,
    balance,
    isConnecting,
    error,
    connectWallet,
    disconnectWallet,
    switchNetwork,
    isConnected: !!account
  };
  
  return (
    <WalletContext.Provider value={value}>
      {children}
    </WalletContext.Provider>
  );
};

// 自定义Hook使用Context
export const useWalletContext = () => {
  const context = useContext(WalletContext);
  if (context === undefined) {
    throw new Error('useWalletContext must be used within WalletProvider');
  }
  return context;
};

// 网络配置
function getNetworkConfig(chainId) {
  const configs = {
    137: { // Polygon
      chainId: '0x89',
      chainName: 'Polygon Mainnet',
      nativeCurrency: {
        name: 'MATIC',
        symbol: 'MATIC',
        decimals: 18
      },
      rpcUrls: ['https://polygon-rpc.com/'],
      blockExplorerUrls: ['https://polygonscan.com/']
    },
    // 可以添加更多网络
  };
  return configs[chainId];
}
```

### 1.2 使用Context

```javascript
// App.js
import React from 'react';
import { WalletProvider } from './contexts/WalletContext';
import Header from './components/Header';
import Dashboard from './components/Dashboard';

function App() {
  return (
    <WalletProvider>
      <div className="app">
        <Header />
        <Dashboard />
      </div>
    </WalletProvider>
  );
}

export default App;

// components/Header.js
import React from 'react';
import { useWalletContext } from '../contexts/WalletContext';

const Header = () => {
  const {
    account,
    chainId,
    balance,
    isConnecting,
    isConnected,
    connectWallet,
    disconnectWallet,
    switchNetwork
  } = useWalletContext();
  
  const networks = {
    1: 'Ethereum',
    5: 'Goerli',
    11155111: 'Sepolia',
    137: 'Polygon'
  };
  
  return (
    <header className="header">
      <div className="logo">My DApp</div>
      
      <div className="wallet-info">
        {!isConnected ? (
          <button 
            onClick={connectWallet}
            disabled={isConnecting}
            className="btn-connect"
          >
            {isConnecting ? 'Connecting...' : 'Connect Wallet'}
          </button>
        ) : (
          <div className="connected">
            <div className="network">
              Network: {networks[chainId] || chainId}
              <button onClick={() => switchNetwork(137)}>
                Switch to Polygon
              </button>
            </div>
            <div className="balance">
              {parseFloat(balance).toFixed(4)} ETH
            </div>
            <div className="account">
              {account.slice(0, 6)}...{account.slice(-4)}
            </div>
            <button onClick={disconnectWallet}>
              Disconnect
            </button>
          </div>
        )}
      </div>
    </header>
  );
};

export default Header;
```

---

## Part 2: 主题系统 (1.5小时)

### 2.1 主题Context

```javascript
// contexts/ThemeContext.js
import React, { createContext, useState, useContext, useEffect } from 'react';

const ThemeContext = createContext(undefined);

const themes = {
  light: {
    background: '#ffffff',
    surface: '#f5f5f5',
    primary: '#1976d2',
    secondary: '#dc004e',
    text: '#000000',
    textSecondary: '#666666',
    border: '#e0e0e0',
    success: '#4caf50',
    error: '#f44336',
    warning: '#ff9800'
  },
  dark: {
    background: '#121212',
    surface: '#1e1e1e',
    primary: '#90caf9',
    secondary: '#f48fb1',
    text: '#ffffff',
    textSecondary: '#b0b0b0',
    border: '#333333',
    success: '#66bb6a',
    error: '#ef5350',
    warning: '#ffa726'
  }
};

export const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState(() => {
    const saved = localStorage.getItem('theme');
    return saved || 'light';
  });
  
  useEffect(() => {
    localStorage.setItem('theme', theme);
    
    // 应用CSS变量
    const colors = themes[theme];
    Object.entries(colors).forEach(([key, value]) => {
      document.documentElement.style.setProperty(`--color-${key}`, value);
    });
  }, [theme]);
  
  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  };
  
  const value = {
    theme,
    themes: themes[theme],
    toggleTheme,
    isDark: theme === 'dark'
  };
  
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
};

export const useTheme = () => {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
};
```

### 2.2 主题切换组件

```javascript
// components/ThemeToggle.js
import React from 'react';
import { useTheme } from '../contexts/ThemeContext';
import './ThemeToggle.css';

const ThemeToggle = () => {
  const { theme, toggleTheme } = useTheme();
  
  return (
    <button 
      className="theme-toggle"
      onClick={toggleTheme}
      aria-label="Toggle theme"
    >
      {theme === 'light' ? '🌙' : '☀️'}
    </button>
  );
};

export default ThemeToggle;

// CSS (ThemeToggle.css)
.theme-toggle {
  width: 50px;
  height: 50px;
  border-radius: 50%;
  border: 2px solid var(--color-border);
  background: var(--color-surface);
  cursor: pointer;
  font-size: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
}

.theme-toggle:hover {
  transform: scale(1.1);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
}
```

### 2.3 全局样式

```css
/* styles/global.css */
:root {
  /* 颜色会通过ThemeContext动态设置 */
  --color-background: #ffffff;
  --color-surface: #f5f5f5;
  --color-primary: #1976d2;
  --color-secondary: #dc004e;
  --color-text: #000000;
  --color-text-secondary: #666666;
  --color-border: #e0e0e0;
  --color-success: #4caf50;
  --color-error: #f44336;
  --color-warning: #ff9800;
  
  /* 间距 */
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;
  
  /* 圆角 */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-full: 9999px;
  
  /* 阴影 */
  --shadow-sm: 0 1px 3px rgba(0, 0, 0, 0.1);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.1);
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen',
    'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue',
    sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  background-color: var(--color-background);
  color: var(--color-text);
  transition: background-color 0.3s ease, color 0.3s ease;
}

.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* 按钮样式 */
.btn {
  padding: var(--spacing-md) var(--spacing-lg);
  border-radius: var(--radius-md);
  border: none;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.2s ease;
}

.btn-primary {
  background: var(--color-primary);
  color: white;
}

.btn-primary:hover {
  opacity: 0.9;
  transform: translateY(-1px);
  box-shadow: var(--shadow-md);
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
}

/* 卡片样式 */
.card {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-lg);
  padding: var(--spacing-lg);
  box-shadow: var(--shadow-sm);
}

/* 输入框样式 */
.input {
  width: 100%;
  padding: var(--spacing-md);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  background: var(--color-background);
  color: var(--color-text);
  font-size: 14px;
}

.input:focus {
  outline: none;
  border-color: var(--color-primary);
  box-shadow: 0 0 0 2px rgba(25, 118, 210, 0.1);
}
```

---

## Part 3: 错误边界 (1小时)

### 3.1 错误边界组件

```javascript
// components/ErrorBoundary.js
import React from 'react';

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
      errorInfo: null
    };
  }
  
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }
  
  componentDidCatch(error, errorInfo) {
    // 记录错误到日志服务
    console.error('Error caught by boundary:', error, errorInfo);
    
    this.setState({
      error,
      errorInfo
    });
    
    // 可以发送到错误追踪服务
    // logErrorToService(error, errorInfo);
  }
  
  render() {
    if (this.state.hasError) {
      return (
        <div className="error-boundary">
          <div className="error-content">
            <h1>😞 Oops! Something went wrong</h1>
            <p>We're sorry for the inconvenience. Please try refreshing the page.</p>
            
            {process.env.NODE_ENV === 'development' && (
              <details style={{ whiteSpace: 'pre-wrap', marginTop: '20px' }}>
                <summary>Error Details (Dev Only)</summary>
                <p>{this.state.error && this.state.error.toString()}</p>
                <p>{this.state.errorInfo && this.state.errorInfo.componentStack}</p>
              </details>
            )}
            
            <button 
              onClick={() => window.location.reload()}
              className="btn-primary"
            >
              Refresh Page
            </button>
          </div>
        </div>
      );
    }
    
    return this.props.children;
  }
}

export default ErrorBoundary;
```

### 3.2 错误处理Hook

```javascript
// hooks/useErrorHandler.js
import { useState, useCallback } from 'react';

export const useErrorHandler = () => {
  const [error, setError] = useState(null);
  
  const handleError = useCallback((err) => {
    console.error('Error:', err);
    
    // 格式化错误消息
    let message = 'An error occurred';
    
    if (err.code === 4001) {
      message = 'Transaction rejected by user';
    } else if (err.code === -32603) {
      message = 'Internal JSON-RPC error';
    } else if (err.message) {
      message = err.message;
    }
    
    setError(message);
    
    // 自动清除错误
    setTimeout(() => setError(null), 5000);
  }, []);
  
  const clearError = useCallback(() => {
    setError(null);
  }, []);
  
  return { error, handleError, clearError };
};

// 使用示例
import React from 'react';
import { useErrorHandler } from '../hooks/useErrorHandler';
import { useWalletContext } from '../contexts/WalletContext';

const TransferForm = () => {
  const { signer } = useWalletContext();
  const { error, handleError, clearError } = useErrorHandler();
  const [to, setTo] = useState('');
  const [amount, setAmount] = useState('');
  const [loading, setLoading] = useState(false);
  
  const handleTransfer = async () => {
    try {
      clearError();
      setLoading(true);
      
      const tx = await signer.sendTransaction({
        to,
        value: ethers.parseEther(amount)
      });
      
      await tx.wait();
      alert('Transfer successful!');
      
    } catch (err) {
      handleError(err);
    } finally {
      setLoading(false);
    }
  };
  
  return (
    <div className="transfer-form">
      {error && (
        <div className="error-message">
          {error}
          <button onClick={clearError}>×</button>
        </div>
      )}
      
      <input
        value={to}
        onChange={(e) => setTo(e.target.value)}
        placeholder="Recipient address"
      />
      
      <input
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        placeholder="Amount (ETH)"
      />
      
      <button onClick={handleTransfer} disabled={loading}>
        {loading ? 'Sending...' : 'Send'}
      </button>
    </div>
  );
};
```

---

## Part 4: React Router (1.5小时)

### 4.1 路由配置

```javascript
// App.js
import React from 'react';
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { WalletProvider } from './contexts/WalletContext';
import { ThemeProvider } from './contexts/ThemeContext';
import ErrorBoundary from './components/ErrorBoundary';
import Layout from './components/Layout';
import Home from './pages/Home';
import Dashboard from './pages/Dashboard';
import Swap from './pages/Swap';
import Pool from './pages/Pool';
import Stake from './pages/Stake';
import NotFound from './pages/NotFound';

function App() {
  return (
    <ErrorBoundary>
      <ThemeProvider>
        <WalletProvider>
          <BrowserRouter>
            <Routes>
              <Route path="/" element={<Layout />}>
                <Route index element={<Home />} />
                <Route path="dashboard" element={<Dashboard />} />
                <Route path="swap" element={<Swap />} />
                <Route path="pool" element={<Pool />} />
                <Route path="stake" element={<Stake />} />
                <Route path="404" element={<NotFound />} />
                <Route path="*" element={<Navigate to="/404" replace />} />
              </Route>
            </Routes>
          </BrowserRouter>
        </WalletProvider>
      </ThemeProvider>
    </ErrorBoundary>
  );
}

export default App;
```

### 4.2 Layout组件

```javascript
// components/Layout.js
import React from 'react';
import { Outlet, Link, useLocation } from 'react-router-dom';
import { useWalletContext } from '../contexts/WalletContext';
import ThemeToggle from './ThemeToggle';
import './Layout.css';

const Layout = () => {
  const location = useLocation();
  const { account, isConnected, connectWallet } = useWalletContext();
  
  const navItems = [
    { path: '/', label: 'Home' },
    { path: '/dashboard', label: 'Dashboard' },
    { path: '/swap', label: 'Swap' },
    { path: '/pool', label: 'Pool' },
    { path: '/stake', label: 'Stake' }
  ];
  
  return (
    <div className="layout">
      <nav className="navbar">
        <div className="nav-brand">
          <Link to="/">🚀 DeFi App</Link>
        </div>
        
        <div className="nav-links">
          {navItems.map(item => (
            <Link
              key={item.path}
              to={item.path}
              className={location.pathname === item.path ? 'active' : ''}
            >
              {item.label}
            </Link>
          ))}
        </div>
        
        <div className="nav-actions">
          <ThemeToggle />
          
          {!isConnected ? (
            <button onClick={connectWallet} className="btn-connect">
              Connect Wallet
            </button>
          ) : (
            <div className="wallet-badge">
              {account.slice(0, 6)}...{account.slice(-4)}
            </div>
          )}
        </div>
      </nav>
      
      <main className="main-content">
        <Outlet />
      </main>
      
      <footer className="footer">
        <p>© 2024 DeFi App. All rights reserved.</p>
      </footer>
    </div>
  );
};

export default Layout;
```

### 4.3 受保护的路由

```javascript
// components/ProtectedRoute.js
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useWalletContext } from '../contexts/WalletContext';

const ProtectedRoute = ({ children }) => {
  const { isConnected } = useWalletContext();
  
  if (!isConnected) {
    return <Navigate to="/" replace />;
  }
  
  return children;
};

export default ProtectedRoute;

// 使用受保护路由
<Route 
  path="dashboard" 
  element={
    <ProtectedRoute>
      <Dashboard />
    </ProtectedRoute>
  } 
/>
```

---

## Part 5: 通知系统 (1小时)

### 5.1 通知Context

```javascript
// contexts/NotificationContext.js
import React, { createContext, useState, useContext, useCallback } from 'react';

const NotificationContext = createContext(undefined);

export const NotificationProvider = ({ children }) => {
  const [notifications, setNotifications] = useState([]);
  
  const addNotification = useCallback((message, type = 'info', duration = 5000) => {
    const id = Date.now();
    
    setNotifications(prev => [...prev, {
      id,
      message,
      type,
      timestamp: new Date()
    }]);
    
    // 自动移除
    if (duration > 0) {
      setTimeout(() => {
        removeNotification(id);
      }, duration);
    }
    
    return id;
  }, []);
  
  const removeNotification = useCallback((id) => {
    setNotifications(prev => prev.filter(n => n.id !== id));
  }, []);
  
  const success = useCallback((message) => {
    return addNotification(message, 'success');
  }, [addNotification]);
  
  const error = useCallback((message) => {
    return addNotification(message, 'error');
  }, [addNotification]);
  
  const warning = useCallback((message) => {
    return addNotification(message, 'warning');
  }, [addNotification]);
  
  const info = useCallback((message) => {
    return addNotification(message, 'info');
  }, [addNotification]);
  
  return (
    <NotificationContext.Provider value={{
      notifications,
      addNotification,
      removeNotification,
      success,
      error,
      warning,
      info
    }}>
      {children}
    </NotificationContext.Provider>
  );
};

export const useNotification = () => {
  const context = useContext(NotificationContext);
  if (!context) {
    throw new Error('useNotification must be used within NotificationProvider');
  }
  return context;
};
```

### 5.2 通知组件

```javascript
// components/NotificationContainer.js
import React from 'react';
import { useNotification } from '../contexts/NotificationContext';
import './Notification.css';

const NotificationContainer = () => {
  const { notifications, removeNotification } = useNotification();
  
  return (
    <div className="notification-container">
      {notifications.map(notification => (
        <div
          key={notification.id}
          className={`notification notification-${notification.type}`}
        >
          <div className="notification-content">
            <span className="notification-icon">
              {getIcon(notification.type)}
            </span>
            <span className="notification-message">
              {notification.message}
            </span>
          </div>
          <button
            className="notification-close"
            onClick={() => removeNotification(notification.id)}
          >
            ×
          </button>
        </div>
      ))}
    </div>
  );
};

function getIcon(type) {
  const icons = {
    success: '✓',
    error: '✗',
    warning: '⚠',
    info: 'ℹ'
  };
  return icons[type] || icons.info;
}

export default NotificationContainer;

// 使用通知
import { useNotification } from '../contexts/NotificationContext';

const MyComponent = () => {
  const { success, error } = useNotification();
  
  const handleAction = async () => {
    try {
      // 执行操作
      success('Operation completed successfully!');
    } catch (err) {
      error('Operation failed: ' + err.message);
    }
  };
  
  return <button onClick={handleAction}>Do Something</button>;
};
```

---

## 📝 今日作业

### 作业1: Context集成

实现完整的Context系统：
1. WalletContext
2. ThemeContext
3. NotificationContext
4. 多Context组合使用

### 作业2: 主题系统

开发完整主题系统：
1. 亮色/暗色主题
2. 平滑切换动画
3. 持久化保存
4. CSS变量应用

### 作业3: 路由应用

实现多页面路由：
1. 基础路由配置
2. 受保护路由
3. 404页面
4. 路由动画

---

## ✅ 检查清单

- [ ] 掌握Context API
- [ ] 实现全局状态管理
- [ ] 完成主题切换
- [ ] 理解错误边界
- [ ] 熟悉React Router

---

## 📅 明日预告

明天学习MetaMask集成：
- MetaMask API
- 钱包事件处理
- 交易签名
- 多链支持

**🎉 完成Day 2！继续努力！**
