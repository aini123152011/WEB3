# Week 6 - Day 4: WalletConnect实现

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 理解WalletConnect协议
- ✅ 集成WalletConnect v2
- ✅ 掌握多钱包支持
- ✅ 实现二维码连接
- ✅ 适配移动端钱包

---

## Part 1: WalletConnect基础 (1.5小时)

### 1.1 协议原理

WalletConnect是一个开放协议，用于在DApp和钱包之间建立通信。它使用桥接服务器在两个客户端之间中继有效载荷。

**主要特点**：
- 端到端加密
- 二维码扫描连接
- 跨设备支持
- 广泛的钱包生态

**工作流程**：
1. DApp生成连接URI（包含密钥）
2. DApp显示二维码
3. 钱包扫描二维码
4. 建立加密通道
5. DApp发送请求，钱包签名

### 1.2 准备工作

首先需要注册WalletConnect Cloud获取Project ID：
1. 访问 [WalletConnect Cloud](https://cloud.walletconnect.com/)
2. 注册账号
3. 创建新项目
4. 获取Project ID

### 1.3 安装依赖

```bash
npm install @web3modal/ethers ethers
# 或者如果你使用React
npm install @web3modal/react @web3modal/core ethers
```

---

## Part 2: 使用Web3Modal集成 (2小时)

Web3Modal是WalletConnect官方推荐的集成库，它不仅支持WalletConnect，还支持Injected (MetaMask)、Coinbase Wallet等。

### 2.1 基础配置

```javascript
// config/web3modal.js
import { createWeb3Modal, defaultConfig } from '@web3modal/ethers/react';

// 1. 获取Project ID
const projectId = 'YOUR_PROJECT_ID';

// 2. 配置网络
const mainnet = {
  chainId: 1,
  name: 'Ethereum',
  currency: 'ETH',
  explorerUrl: 'https://etherscan.io',
  rpcUrl: 'https://cloudflare-eth.com'
};

const sepolia = {
  chainId: 11155111,
  name: 'Sepolia',
  currency: 'SEP',
  explorerUrl: 'https://sepolia.etherscan.io',
  rpcUrl: 'https://rpc.sepolia.org'
};

const polygon = {
  chainId: 137,
  name: 'Polygon',
  currency: 'MATIC',
  explorerUrl: 'https://polygonscan.com',
  rpcUrl: 'https://polygon-rpc.com'
};

// 3. 创建元数据
const metadata = {
  name: 'My Web3 DApp',
  description: 'My Web3 DApp Description',
  url: 'https://mywebsite.com', // 你的网站域名
  icons: ['https://avatars.mywebsite.com/']
};

// 4. 初始化Web3Modal
createWeb3Modal({
  ethersConfig: defaultConfig({ metadata }),
  chains: [mainnet, sepolia, polygon],
  projectId,
  enableAnalytics: true, // 可选
  themeMode: 'light', // 'light' | 'dark'
  themeVariables: {
    '--w3m-font-family': 'Roboto, sans-serif',
    '--w3m-accent': '#1976d2'
  }
});

export const web3ModalConfig = {
  projectId,
  chains: [mainnet, sepolia, polygon]
};
```

### 2.2 React集成

```javascript
// App.js
import React from 'react';
import './config/web3modal'; // 引入配置以初始化

export default function App() {
  return (
    <div className="app">
      {/* 你的应用组件 */}
      <Header />
      <Main />
    </div>
  );
}

// components/ConnectButton.js
import React from 'react';

export default function ConnectButton() {
  return (
    <>
      {/* 使用官方组件 */}
      <w3m-button />
      
      {/* 或者自定义按钮触发网络切换 */}
      {/* <w3m-network-button /> */}
    </>
  );
}
```

### 2.3 自定义Hook

```javascript
// hooks/useWeb3Modal.js
import { 
  useWeb3Modal, 
  useWeb3ModalAccount, 
  useWeb3ModalProvider,
  useWeb3ModalError
} from '@web3modal/ethers/react';
import { ethers } from 'ethers';
import { useState, useEffect } from 'react';

export const useWalletConnect = () => {
  const { open, close } = useWeb3Modal();
  const { address, chainId, isConnected } = useWeb3ModalAccount();
  const { walletProvider } = useWeb3ModalProvider();
  const { error } = useWeb3ModalError();
  
  const [ethersProvider, setEthersProvider] = useState(null);
  const [signer, setSigner] = useState(null);
  
  useEffect(() => {
    if (walletProvider) {
      const provider = new ethers.BrowserProvider(walletProvider);
      setEthersProvider(provider);
      
      provider.getSigner().then(signer => {
        setSigner(signer);
      });
    } else {
      setEthersProvider(null);
      setSigner(null);
    }
  }, [walletProvider]);
  
  return {
    open,
    close,
    address,
    chainId,
    isConnected,
    provider: ethersProvider,
    signer,
    error
  };
};
```

---

## Part 3: 独立WalletConnect集成 (2小时)

如果你不想使用Web3Modal，可以直接使用 `@walletconnect/ethereum-provider`。

### 3.1 Provider设置

```javascript
// utils/walletConnect.js
import { EthereumProvider } from '@walletconnect/ethereum-provider';

export const createWalletConnectProvider = async () => {
  try {
    const provider = await EthereumProvider.init({
      projectId: 'YOUR_PROJECT_ID', // 必填
      chains: [1], // 必填：主链ID
      optionalChains: [11155111, 137], // 可选链
      showQrModal: true, // 是否显示官方二维码模态框
      methods: ['eth_sendTransaction', 'personal_sign'], // 必填
      events: ['chainChanged', 'accountsChanged'], // 必填
      metadata: {
        name: 'My DApp',
        description: 'My DApp description',
        url: 'https://my-dapp.com',
        icons: ['https://my-dapp.com/logo.png']
      }
    });
    
    return provider;
  } catch (error) {
    console.error('WalletConnect init failed:', error);
    throw error;
  }
};
```

### 3.2 连接逻辑

```javascript
// hooks/useStandaloneWalletConnect.js
import { useState, useEffect, useCallback } from 'react';
import { ethers } from 'ethers';
import { createWalletConnectProvider } from '../utils/walletConnect';

export const useStandaloneWalletConnect = () => {
  const [wcProvider, setWcProvider] = useState(null);
  const [provider, setProvider] = useState(null);
  const [account, setAccount] = useState(null);
  const [chainId, setChainId] = useState(null);
  const [isConnecting, setIsConnecting] = useState(false);
  
  // 初始化
  useEffect(() => {
    const init = async () => {
      const _wcProvider = await createWalletConnectProvider();
      setWcProvider(_wcProvider);
      
      // 检查是否已经连接
      if (_wcProvider.accounts.length > 0) {
        handleAccountsChanged(_wcProvider.accounts);
        handleChainChanged(_wcProvider.chainId);
      }
      
      // 设置事件监听
      _wcProvider.on('accountsChanged', handleAccountsChanged);
      _wcProvider.on('chainChanged', handleChainChanged);
      _wcProvider.on('disconnect', handleDisconnect);
    };
    
    init();
    
    return () => {
      // 清理工作
      if (wcProvider) {
        wcProvider.removeListener('accountsChanged', handleAccountsChanged);
        wcProvider.removeListener('chainChanged', handleChainChanged);
        wcProvider.removeListener('disconnect', handleDisconnect);
      }
    };
  }, []);
  
  // 事件处理
  const handleAccountsChanged = (accounts) => {
    if (accounts.length > 0) {
      setAccount(accounts[0]);
      updateEthersProvider(wcProvider);
    } else {
      handleDisconnect();
    }
  };
  
  const handleChainChanged = (id) => {
    setChainId(Number(id));
    updateEthersProvider(wcProvider);
  };
  
  const handleDisconnect = () => {
    setAccount(null);
    setChainId(null);
    setProvider(null);
  };
  
  const updateEthersProvider = (walletProvider) => {
    if (walletProvider) {
      const ethersProvider = new ethers.BrowserProvider(walletProvider);
      setProvider(ethersProvider);
    }
  };
  
  // 连接功能
  const connect = async () => {
    if (!wcProvider) return;
    
    try {
      setIsConnecting(true);
      await wcProvider.connect();
    } catch (error) {
      console.error('Connection error:', error);
    } finally {
      setIsConnecting(false);
    }
  };
  
  // 断开功能
  const disconnect = async () => {
    if (!wcProvider) return;
    
    try {
      await wcProvider.disconnect();
    } catch (error) {
      console.error('Disconnect error:', error);
    }
  };
  
  return {
    account,
    chainId,
    provider,
    isConnecting,
    connect,
    disconnect,
    isConnected: !!account
  };
};
```

---

## Part 4: 统一钱包适配器 (1.5小时)

为了同时支持MetaMask和WalletConnect，我们需要创建一个统一的适配器模式。

### 4.1 适配器接口

```javascript
// adapters/WalletAdapter.js

class WalletAdapter {
  constructor() {
    if (this.constructor === WalletAdapter) {
      throw new Error("Abstract class cannot be instantiated");
    }
  }
  
  async connect() { throw new Error("Method 'connect' must be implemented"); }
  async disconnect() { throw new Error("Method 'disconnect' must be implemented"); }
  async getSigner() { throw new Error("Method 'getSigner' must be implemented"); }
  async getAccount() { throw new Error("Method 'getAccount' must be implemented"); }
  async getChainId() { throw new Error("Method 'getChainId' must be implemented"); }
}

export default WalletAdapter;
```

### 4.2 MetaMask适配器

```javascript
// adapters/MetaMaskAdapter.js
import { ethers } from 'ethers';
import WalletAdapter from './WalletAdapter';

class MetaMaskAdapter extends WalletAdapter {
  constructor() {
    super();
    this.provider = window.ethereum ? new ethers.BrowserProvider(window.ethereum) : null;
  }
  
  isAvailable() {
    return !!this.provider;
  }
  
  async connect() {
    if (!this.isAvailable()) throw new Error("MetaMask not installed");
    await this.provider.send("eth_requestAccounts", []);
    return this.getAccount();
  }
  
  async disconnect() {
    // MetaMask不能通过API断开，只能在UI上清理状态
    return true;
  }
  
  async getSigner() {
    if (!this.isAvailable()) return null;
    return await this.provider.getSigner();
  }
  
  async getAccount() {
    if (!this.isAvailable()) return null;
    const accounts = await this.provider.listAccounts();
    return accounts.length > 0 ? accounts[0].address : null;
  }
  
  async getChainId() {
    if (!this.isAvailable()) return null;
    const network = await this.provider.getNetwork();
    return Number(network.chainId);
  }
}

export default new MetaMaskAdapter();
```

### 4.3 WalletConnect适配器

```javascript
// adapters/WalletConnectAdapter.js
import { ethers } from 'ethers';
import { createWalletConnectProvider } from '../utils/walletConnect';
import WalletAdapter from './WalletAdapter';

class WalletConnectAdapter extends WalletAdapter {
  constructor() {
    super();
    this.wcProvider = null;
    this.ethersProvider = null;
    this.initPromise = this.init();
  }
  
  async init() {
    this.wcProvider = await createWalletConnectProvider();
    this.ethersProvider = new ethers.BrowserProvider(this.wcProvider);
  }
  
  async connect() {
    await this.initPromise;
    await this.wcProvider.connect();
    return this.getAccount();
  }
  
  async disconnect() {
    await this.initPromise;
    await this.wcProvider.disconnect();
  }
  
  async getSigner() {
    await this.initPromise;
    return await this.ethersProvider.getSigner();
  }
  
  async getAccount() {
    await this.initPromise;
    const accounts = this.wcProvider.accounts;
    return accounts.length > 0 ? accounts[0] : null;
  }
  
  async getChainId() {
    await this.initPromise;
    return this.wcProvider.chainId;
  }
}

export default new WalletConnectAdapter();
```

### 4.4 统一Hook

```javascript
// hooks/useUnifiedWallet.js
import { useState, useCallback } from 'react';
import MetaMaskAdapter from '../adapters/MetaMaskAdapter';
import WalletConnectAdapter from '../adapters/WalletConnectAdapter';

const ADAPTERS = {
  metamask: MetaMaskAdapter,
  walletconnect: WalletConnectAdapter
};

export const useUnifiedWallet = () => {
  const [activeAdapter, setActiveAdapter] = useState(null);
  const [account, setAccount] = useState(null);
  const [isConnecting, setIsConnecting] = useState(false);
  
  const connect = useCallback(async (walletType) => {
    const adapter = ADAPTERS[walletType];
    if (!adapter) throw new Error("Unknown wallet type");
    
    try {
      setIsConnecting(true);
      const acc = await adapter.connect();
      setActiveAdapter(adapter);
      setAccount(acc);
      localStorage.setItem('wallet_type', walletType);
    } catch (error) {
      console.error("Connect failed", error);
      throw error;
    } finally {
      setIsConnecting(false);
    }
  }, []);
  
  const disconnect = useCallback(async () => {
    if (activeAdapter) {
      await activeAdapter.disconnect();
      setActiveAdapter(null);
      setAccount(null);
      localStorage.removeItem('wallet_type');
    }
  }, [activeAdapter]);
  
  // 恢复连接
  // useEffect to check localStorage and reconnect...
  
  return {
    connect,
    disconnect,
    account,
    isConnecting,
    activeAdapter
  };
};
```

---

## Part 5: 移动端适配 (1小时)

### 5.1 移动端深度链接

当在移动设备上时，WalletConnect需要处理Deep Link跳转到钱包APP。

```javascript
// utils/mobile.js

export const isMobile = () => {
  return /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
    navigator.userAgent
  );
};

export const openWalletApp = (uri) => {
  if (isMobile()) {
    // 这里的uri是WalletConnect生成的连接字符串
    window.location.href = uri;
  }
};
```

### 5.2 响应式UI

```css
/* styles/MobileWallet.css */

.wallet-modal {
  position: fixed;
  bottom: 0;
  left: 0;
  width: 100%;
  background: white;
  border-top-left-radius: 20px;
  border-top-right-radius: 20px;
  padding: 20px;
  box-shadow: 0 -4px 12px rgba(0,0,0,0.1);
  transform: translateY(100%);
  transition: transform 0.3s ease;
}

.wallet-modal.open {
  transform: translateY(0);
}

.wallet-option {
  display: flex;
  align-items: center;
  padding: 15px;
  border-bottom: 1px solid #eee;
  cursor: pointer;
}

.wallet-option img {
  width: 40px;
  height: 40px;
  margin-right: 15px;
}

/* Desktop styles */
@media (min-width: 768px) {
  .wallet-modal {
    top: 50%;
    left: 50%;
    bottom: auto;
    width: 400px;
    border-radius: 12px;
    transform: translate(-50%, -50%) scale(0.9);
    opacity: 0;
    pointer-events: none;
  }
  
  .wallet-modal.open {
    transform: translate(-50%, -50%) scale(1);
    opacity: 1;
    pointer-events: auto;
  }
}
```

---

## 📝 今日作业

### 作业1: 集成Web3Modal

使用`@web3modal/ethers`库：
1. 配置Project ID
2. 添加主网和测试网
3. 实现自定义连接按钮
4. 自定义主题样式

### 作业2: 统一钱包管理器

实现一个WalletManager组件：
1. 支持MetaMask和WalletConnect切换
2. 记住用户的钱包选择
3. 统一的账户和签名接口
4. 优雅的错误处理

### 作业3: 移动端优化

优化移动端体验：
1. 检测移动设备
2. 适配移动端UI布局
3. 处理移动端钱包跳转
4. 测试在手机浏览器中的表现

---

## ✅ 检查清单

- [ ] 注册WalletConnect Project ID
- [ ] 成功集成Web3Modal
- [ ] 理解WalletConnect协议原理
- [ ] 实现多钱包适配器
- [ ] 移动端测试通过

---

## 📅 明日预告

明天学习多链支持：
- EIP-3085: 添加自定义链
- 自动切换网络
- 多链状态管理
- 跨链数据聚合

**🎉 完成Day 4！前端集成越来越完善了！**
