# Week 7 - Day 2: Web3React深入

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 学习目标

- ✅ 理解Web3React架构
- ✅ 掌握Connectors配置
- ✅ 实现多钱包连接
- ✅ 学习Web3React Hooks
- ✅ 处理连接错误

---

## Part 1: Web3React基础 (1.5小时)

### 1.1 安装与配置

```bash
npm install @web3-react/core @web3-react/injected-connector @web3-react/walletconnect-connector @web3-react/walletlink-connector
npm install ethers
```

### 1.2 Provider配置

```javascript
// App.js
import { Web3ReactProvider } from '@web3-react/core';
import { ethers } from 'ethers';

function getLibrary(provider) {
  const library = new ethers.BrowserProvider(provider);
  library.pollingInterval = 12000;
  return library;
}

export default function App() {
  return (
    <Web3ReactProvider getLibrary={getLibrary}>
      <MyComponent />
    </Web3ReactProvider>
  );
}
```

---

## Part 2: Connectors配置 (2小时)

### 2.1 连接器定义

```javascript
// connectors.js
import { InjectedConnector } from '@web3-react/injected-connector';
import { WalletConnectConnector } from '@web3-react/walletconnect-connector';
import { WalletLinkConnector } from '@web3-react/walletlink-connector';

const RPC_URLS = {
  1: 'https://mainnet.infura.io/v3/YOUR_PROJECT_ID',
  4: 'https://rinkeby.infura.io/v3/YOUR_PROJECT_ID'
};

// MetaMask
export const injected = new InjectedConnector({
  supportedChainIds: [1, 3, 4, 5, 42]
});

// WalletConnect
export const walletconnect = new WalletConnectConnector({
  rpc: { 1: RPC_URLS[1] },
  bridge: 'https://bridge.walletconnect.org',
  qrcode: true
});

// Coinbase Wallet
export const walletlink = new WalletLinkConnector({
  url: RPC_URLS[1],
  appName: 'My Web3 App',
  supportedChainIds: [1, 3, 4, 5, 42]
});
```

### 2.2 连接管理器

```javascript
// hooks/useEagerConnect.js
import { useState, useEffect } from 'react';
import { useWeb3React } from '@web3-react/core';
import { injected } from '../connectors';

export function useEagerConnect() {
  const { activate, active } = useWeb3React();
  const [tried, setTried] = useState(false);

  useEffect(() => {
    injected.isAuthorized().then((isAuthorized) => {
      if (isAuthorized) {
        activate(injected, undefined, true).catch(() => {
          setTried(true);
        });
      } else {
        setTried(true);
      }
    });
  }, []); // intentionally only running on mount (make sure it's only mounted once :))

  // if the connection worked, wait until we get confirmation of that to flip the flag
  useEffect(() => {
    if (!tried && active) {
      setTried(true);
    }
  }, [tried, active]);

  return tried;
}
```

---

## Part 3: 连接状态管理 (2小时)

### 3.1 钱包连接组件

```javascript
import { useWeb3React } from '@web3-react/core';
import { injected, walletconnect, walletlink } from '../connectors';

const WalletConnector = () => {
  const { activate, deactivate, active, error } = useWeb3React();

  const handleConnect = async (connector) => {
    try {
      await activate(connector);
    } catch (ex) {
      console.log(ex);
    }
  };

  return (
    <div>
      <h3>Connect Wallet</h3>
      <button onClick={() => handleConnect(injected)}>MetaMask</button>
      <button onClick={() => handleConnect(walletconnect)}>WalletConnect</button>
      <button onClick={() => handleConnect(walletlink)}>Coinbase Wallet</button>
      
      {active && <button onClick={deactivate}>Disconnect</button>}
      {error && <p>Error: {error.message}</p>}
    </div>
  );
};
```

### 3.2 账户信息显示

```javascript
import { useWeb3React } from '@web3-react/core';
import { formatEther } from '@ethersproject/units';
import useSWR from 'swr';

const AccountInfo = () => {
  const { account, library, chainId } = useWeb3React();

  const { data: balance } = useSWR(['getBalance', account, 'latest'], {
    fetcher: (key, ...args) => library.getBalance(...args),
  });

  if (!account) return null;

  return (
    <div>
      <p>Account: {account}</p>
      <p>Chain ID: {chainId}</p>
      <p>Balance: {balance ? formatEther(balance) : '...'} ETH</p>
    </div>
  );
};
```

---

## Part 4: 错误处理与网络切换 (1.5小时)

### 4.1 错误处理

```javascript
// utils/getErrorMessage.js
import { UnsupportedChainIdError } from '@web3-react/core';
import {
  NoEthereumProviderError,
  UserRejectedRequestError as UserRejectedRequestErrorInjected
} from '@web3-react/injected-connector';

export function getErrorMessage(error) {
  if (error instanceof NoEthereumProviderError) {
    return 'No Ethereum browser extension detected, install MetaMask on desktop or visit from a dApp browser on mobile.';
  } else if (error instanceof UnsupportedChainIdError) {
    return "You're connected to an unsupported network.";
  } else if (error instanceof UserRejectedRequestErrorInjected) {
    return 'Please authorize this website to access your Ethereum account.';
  } else {
    console.error(error);
    return 'An unknown error occurred. Check the console for more details.';
  }
}
```

### 4.2 网络切换

```javascript
const switchNetwork = async (library, chainId) => {
  try {
    await library.provider.request({
      method: 'wallet_switchEthereumChain',
      params: [{ chainId: `0x${chainId.toString(16)}` }],
    });
  } catch (switchError) {
    // This error code indicates that the chain has not been added to MetaMask.
    if (switchError.code === 4902) {
      try {
        await library.provider.request({
          method: 'wallet_addEthereumChain',
          params: [
            {
              chainId: `0x${chainId.toString(16)}`,
              chainName: '...',
              rpcUrls: ['...'],
            },
          ],
        });
      } catch (addError) {
        // handle "add" error
      }
    }
    // handle other "switch" errors
  }
};
```

---

## 📝 今日作业

### 作业1: 实现多钱包连接模态框

创建一个美观的模态框，支持：
1. MetaMask
2. WalletConnect
3. Coinbase Wallet
4. 显示连接状态和错误信息

### 作业2: 自动连接与持久化

实现：
1. 页面刷新后自动重连
2. 记住上次使用的钱包
3. 处理账户切换和断开连接事件

### 作业3: 网络切换器

创建一个网络切换组件：
1. 显示当前网络
2. 列出支持的网络
3. 点击切换网络
4. 处理不支持的网络错误

---

## ✅ 检查清单

- [ ] 成功配置Web3ReactProvider
- [ ] 实现三种以上钱包连接
- [ ] 能够正确显示账户和余额
- [ ] 实现错误处理机制
- [ ] 完成网络切换功能

---

## 📅 明日预告

明天学习状态管理与数据流：
- Redux Toolkit集成
- Zustand使用
- React Query / SWR 数据获取
- 乐观UI更新

**🎉 完成Day 2！Web3集成更进一步！**
