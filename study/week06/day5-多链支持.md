# Week 6 - Day 5: 多链支持

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 理解EIP-3085标准
- ✅ 实现多链配置管理
- ✅ 掌握自动网络切换
- ✅ 学习多链数据聚合
- ✅ 优化多链用户体验

---

## Part 1: 多链配置管理 (1.5小时)

### 1.1 链配置标准

根据EIP-3085，我们需要定义标准的链参数。

```javascript
// config/chains.js

export const CHAIN_IDS = {
  ETHEREUM: 1,
  GOERLI: 5,
  SEPOLIA: 11155111,
  BSC: 56,
  BSC_TESTNET: 97,
  POLYGON: 137,
  MUMBAI: 80001,
  ARBITRUM: 42161,
  OPTIMISM: 42161,
  AVALANCHE: 43114
};

export const CHAINS = {
  [CHAIN_IDS.ETHEREUM]: {
    chainId: '0x1',
    chainName: 'Ethereum Mainnet',
    nativeCurrency: {
      name: 'Ether',
      symbol: 'ETH',
      decimals: 18
    },
    rpcUrls: ['https://mainnet.infura.io/v3/YOUR_PROJECT_ID'],
    blockExplorerUrls: ['https://etherscan.io'],
    iconUrl: '/icons/eth.png'
  },
  [CHAIN_IDS.SEPOLIA]: {
    chainId: '0xaa36a7',
    chainName: 'Sepolia Testnet',
    nativeCurrency: {
      name: 'Sepolia Ether',
      symbol: 'SEP',
      decimals: 18
    },
    rpcUrls: ['https://sepolia.infura.io/v3/YOUR_PROJECT_ID'],
    blockExplorerUrls: ['https://sepolia.etherscan.io'],
    iconUrl: '/icons/eth.png',
    isTestnet: true
  },
  [CHAIN_IDS.BSC]: {
    chainId: '0x38',
    chainName: 'Binance Smart Chain',
    nativeCurrency: {
      name: 'BNB',
      symbol: 'BNB',
      decimals: 18
    },
    rpcUrls: ['https://bsc-dataseed.binance.org/'],
    blockExplorerUrls: ['https://bscscan.com'],
    iconUrl: '/icons/bnb.png'
  },
  [CHAIN_IDS.POLYGON]: {
    chainId: '0x89',
    chainName: 'Polygon Mainnet',
    nativeCurrency: {
      name: 'MATIC',
      symbol: 'MATIC',
      decimals: 18
    },
    rpcUrls: ['https://polygon-rpc.com/'],
    blockExplorerUrls: ['https://polygonscan.com'],
    iconUrl: '/icons/matic.png'
  }
};

export const getChainConfig = (chainId) => {
  const config = CHAINS[chainId];
  if (!config) {
    throw new Error(`Chain ${chainId} not supported`);
  }
  return config;
};
```

### 1.2 链状态管理

```javascript
// hooks/useChain.js
import { useState, useEffect, useCallback } from 'react';
import { CHAINS, CHAIN_IDS } from '../config/chains';

export const useChain = (provider) => {
  const [currentChainId, setCurrentChainId] = useState(null);
  const [isSupported, setIsSupported] = useState(true);
  const [chainConfig, setChainConfig] = useState(null);
  
  // 更新链状态
  const updateChainState = useCallback((chainId) => {
    const id = Number(chainId);
    setCurrentChainId(id);
    
    const config = CHAINS[id];
    setChainConfig(config || null);
    setIsSupported(!!config);
  }, []);
  
  // 初始化和监听
  useEffect(() => {
    if (!provider) return;
    
    const init = async () => {
      const network = await provider.getNetwork();
      updateChainState(network.chainId);
    };
    
    init();
    
    // 如果是MetaMask provider，可以监听事件
    if (provider.provider?.on) {
      const handleChainChanged = (chainId) => {
        updateChainState(chainId);
        window.location.reload();
      };
      
      provider.provider.on('chainChanged', handleChainChanged);
      
      return () => {
        provider.provider.removeListener('chainChanged', handleChainChanged);
      };
    }
  }, [provider, updateChainState]);
  
  return {
    chainId: currentChainId,
    isSupported,
    config: chainConfig,
    supportedChains: Object.values(CHAINS)
  };
};
```

---

## Part 2: 网络切换 (1.5小时)

### 2.1 自动切换逻辑

```javascript
// utils/network.js
import { getChainConfig } from '../config/chains';

export const switchNetwork = async (provider, chainId) => {
  if (!provider.send) {
    throw new Error('Provider does not support network switching');
  }
  
  const config = getChainConfig(chainId);
  const hexChainId = `0x${Number(chainId).toString(16)}`;
  
  try {
    // 尝试切换网络
    await provider.send('wallet_switchEthereumChain', [
      { chainId: hexChainId }
    ]);
  } catch (error) {
    // 如果网络不存在(错误代码 4902)，则尝试添加
    if (error.code === 4902) {
      try {
        await provider.send('wallet_addEthereumChain', [
          {
            chainId: hexChainId,
            chainName: config.chainName,
            nativeCurrency: config.nativeCurrency,
            rpcUrls: config.rpcUrls,
            blockExplorerUrls: config.blockExplorerUrls
          }
        ]);
      } catch (addError) {
        console.error('Failed to add network:', addError);
        throw addError;
      }
    } else {
      console.error('Failed to switch network:', error);
      throw error;
    }
  }
};
```

### 2.2 网络切换组件

```javascript
// components/NetworkSelector.js
import React, { useState } from 'react';
import { useChain } from '../hooks/useChain';
import { switchNetwork } from '../utils/network';
import './NetworkSelector.css';

const NetworkSelector = ({ provider }) => {
  const { chainId, supportedChains, isSupported } = useChain(provider);
  const [isSwitching, setIsSwitching] = useState(false);
  const [error, setError] = useState(null);
  
  const handleSwitch = async (targetChainId) => {
    if (targetChainId === chainId) return;
    
    setIsSwitching(true);
    setError(null);
    
    try {
      await switchNetwork(provider, targetChainId);
    } catch (err) {
      setError(err.message);
    } finally {
      setIsSwitching(false);
    }
  };
  
  return (
    <div className="network-selector">
      {!isSupported && (
        <div className="network-warning">
          Wrong Network! Please switch to a supported network.
        </div>
      )}
      
      {error && <div className="network-error">{error}</div>}
      
      <div className="chain-list">
        {supportedChains.map((chain) => (
          <button
            key={chain.chainId}
            className={`chain-item ${
              parseInt(chain.chainId, 16) === chainId ? 'active' : ''
            }`}
            onClick={() => handleSwitch(parseInt(chain.chainId, 16))}
            disabled={isSwitching}
          >
            <img 
              src={chain.iconUrl} 
              alt={chain.chainName} 
              className="chain-icon"
            />
            <span className="chain-name">{chain.chainName}</span>
            {isSwitching && parseInt(chain.chainId, 16) !== chainId && (
              <span className="loading-spinner" />
            )}
          </button>
        ))}
      </div>
    </div>
  );
};

export default NetworkSelector;
```

---

## Part 3: 多链数据聚合 (2小时)

### 3.1 Multicall配置

为了在多链上高效查询数据，我们使用Multicall。

```javascript
// config/multicall.js

export const MULTICALL_ADDRESSES = {
  1: '0xcA11bde05977b3631167028862bE2a173976CA11', // Mainnet
  5: '0xcA11bde05977b3631167028862bE2a173976CA11', // Goerli
  56: '0xcA11bde05977b3631167028862bE2a173976CA11', // BSC
  137: '0xcA11bde05977b3631167028862bE2a173976CA11' // Polygon
};

export const MULTICALL_ABI = [
  'function aggregate(tuple(address target, bytes callData)[] calls) view returns (uint256 blockNumber, bytes[] returnData)',
  'function getEthBalance(address addr) view returns (uint256 balance)'
];
```

### 3.2 跨链数据Hook

```javascript
// hooks/useMultiChainData.js
import { useState, useEffect } from 'react';
import { ethers } from 'ethers';
import { CHAINS, CHAIN_IDS } from '../config/chains';
import { MULTICALL_ADDRESSES, MULTICALL_ABI } from '../config/multicall';

export const useMultiChainBalances = (account) => {
  const [balances, setBalances] = useState({});
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    if (!account) return;
    
    const fetchBalances = async () => {
      setLoading(true);
      const results = {};
      
      try {
        // 并行请求所有支持链的余额
        const promises = Object.entries(CHAINS).map(async ([chainId, config]) => {
          try {
            const provider = new ethers.JsonRpcProvider(config.rpcUrls[0]);
            const multicallAddress = MULTICALL_ADDRESSES[chainId];
            
            if (multicallAddress) {
              const multicall = new ethers.Contract(
                multicallAddress,
                MULTICALL_ABI,
                provider
              );
              const balance = await multicall.getEthBalance(account);
              return { chainId, balance: ethers.formatEther(balance) };
            } else {
              // 回退到普通RPC调用
              const balance = await provider.getBalance(account);
              return { chainId, balance: ethers.formatEther(balance) };
            }
          } catch (err) {
            console.error(`Failed to fetch balance for chain ${chainId}:`, err);
            return { chainId, balance: '0', error: true };
          }
        });
        
        const balanceData = await Promise.all(promises);
        
        balanceData.forEach(({ chainId, balance }) => {
          results[chainId] = balance;
        });
        
        setBalances(results);
      } catch (error) {
        console.error('Multi-chain fetch error:', error);
      } finally {
        setLoading(false);
      }
    };
    
    fetchBalances();
    const interval = setInterval(fetchBalances, 15000); // 每15秒更新
    
    return () => clearInterval(interval);
  }, [account]);
  
  return { balances, loading };
};
```

---

## Part 4: 合约地址管理 (1小时)

### 4.1 地址映射

管理不同链上的合约地址。

```javascript
// config/contracts.js

export const CONTRACTS = {
  USDC: {
    [CHAIN_IDS.ETHEREUM]: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
    [CHAIN_IDS.BSC]: '0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d',
    [CHAIN_IDS.POLYGON]: '0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174'
  },
  UNISWAP_ROUTER: {
    [CHAIN_IDS.ETHEREUM]: '0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D',
    // 其他链可能使用不同的DEX
    [CHAIN_IDS.BSC]: '0x10ED43C718714eb63d5aA57B78B54704E256024E', // PancakeSwap
    [CHAIN_IDS.POLYGON]: '0xa5E0829CaCEd8fFDD4De3c43696c57F7D7A678ff' // QuickSwap
  }
};

export const getContractAddress = (name, chainId) => {
  const addresses = CONTRACTS[name];
  if (!addresses) throw new Error(`Contract ${name} not configured`);
  
  const address = addresses[chainId];
  if (!address) throw new Error(`Contract ${name} not available on chain ${chainId}`);
  
  return address;
};
```

### 4.2 自动合约实例化

```javascript
// hooks/useContract.js
import { useMemo } from 'react';
import { ethers } from 'ethers';
import { useChain } from './useChain';
import { getContractAddress } from '../config/contracts';

export const useTokenContract = (tokenName, provider) => {
  const { chainId } = useChain(provider);
  
  return useMemo(() => {
    if (!chainId || !provider) return null;
    
    try {
      const address = getContractAddress(tokenName, chainId);
      const abi = [
        'function balanceOf(address) view returns (uint256)',
        'function decimals() view returns (uint8)',
        'function symbol() view returns (string)',
        'function transfer(address to, uint256 amount) returns (bool)'
      ];
      
      return new ethers.Contract(address, abi, provider.getSigner ? provider.getSigner() : provider);
    } catch (error) {
      console.warn(`Token ${tokenName} not available on current chain`);
      return null;
    }
  }, [tokenName, chainId, provider]);
};
```

---

## Part 5: 跨链组件封装 (1小时)

### 5.1 跨链按钮

一个智能按钮，如果不在正确的链上，先提示切换网络。

```javascript
// components/ChainAwareButton.js
import React from 'react';
import { useChain } from '../hooks/useChain';
import { switchNetwork } from '../utils/network';

const ChainAwareButton = ({ 
  targetChainId, 
  provider, 
  onClick, 
  children,
  ...props 
}) => {
  const { chainId } = useChain(provider);
  const isCorrectChain = chainId === targetChainId;
  
  const handleClick = async (e) => {
    if (!isCorrectChain) {
      try {
        await switchNetwork(provider, targetChainId);
      } catch (error) {
        console.error('Switch failed:', error);
      }
    } else {
      onClick && onClick(e);
    }
  };
  
  return (
    <button 
      onClick={handleClick}
      className={!isCorrectChain ? 'btn-warning' : 'btn-primary'}
      {...props}
    >
      {!isCorrectChain ? 'Switch Network' : children}
    </button>
  );
};

export default ChainAwareButton;
```

### 5.2 综合示例

```javascript
// App.js
import React from 'react';
import NetworkSelector from './components/NetworkSelector';
import ChainAwareButton from './components/ChainAwareButton';
import { useMultiChainBalances } from './hooks/useMultiChainData';
import { CHAIN_IDS, CHAINS } from './config/chains';

const App = ({ provider, account }) => {
  const { balances, loading } = useMultiChainBalances(account);
  
  return (
    <div className="app">
      <header>
        <NetworkSelector provider={provider} />
      </header>
      
      <main>
        <section className="balances">
          <h2>Your Balances Across Chains</h2>
          {loading ? (
            <p>Loading balances...</p>
          ) : (
            <div className="balance-grid">
              {Object.entries(balances).map(([chainId, balance]) => (
                <div key={chainId} className="balance-card">
                  <img 
                    src={CHAINS[chainId].iconUrl} 
                    alt={CHAINS[chainId].chainName} 
                  />
                  <div className="balance-info">
                    <h3>{CHAINS[chainId].chainName}</h3>
                    <p>{parseFloat(balance).toFixed(4)} {CHAINS[chainId].nativeCurrency.symbol}</p>
                  </div>
                  
                  <ChainAwareButton
                    targetChainId={Number(chainId)}
                    provider={provider}
                    onClick={() => alert('Ready to transact!')}
                  >
                    Transact
                  </ChainAwareButton>
                </div>
              ))}
            </div>
          )}
        </section>
      </main>
    </div>
  );
};

export default App;
```

---

## 📝 今日作业

### 作业1: 完善配置

创建一个完整的配置文件：
1. 支持至少5条公链
2. 配置主流测试网
3. 包含常用代币地址
4. 包含多链RPC节点

### 作业2: 自动切换器

开发一个智能网络切换器：
1. 检测当前网络
2. 提示错误网络
3. 一键切换
4. 自动添加新网络配置

### 作业3: 多链资产看板

实现一个多链资产展示页面：
1. 并行查询多链余额
2. 聚合显示总资产
3. 按链过滤显示
4. 实时更新数据

---

## ✅ 检查清单

- [ ] 配置EIP-3085参数
- [ ] 实现自动添加网络
- [ ] 完成多链数据聚合
- [ ] 封装跨链交互组件
- [ ] 优化切换体验

---

## 📅 明日预告

周末将进行完整项目实战：
- 整合Week 6所有知识
- 构建多链DApp前端
- 集成多种钱包
- 实现完整交互流程

**🎉 完成Day 5！准备好迎接周末挑战！**
