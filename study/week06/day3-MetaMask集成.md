# Week 6 - Day 3: MetaMask集成

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 深入MetaMask API
- ✅ 掌握交易签名
- ✅ 理解消息签名
- ✅ 实现多链切换
- ✅ 处理钱包事件

---

## Part 1: MetaMask检测与连接 (1.5小时)

### 1.1 MetaMask检测

```javascript
// utils/metamask.js

/**
 * 检查MetaMask是否安装
 */
export const isMetaMaskInstalled = () => {
  return typeof window !== 'undefined' && 
         typeof window.ethereum !== 'undefined' && 
         window.ethereum.isMetaMask;
};

/**
 * 获取MetaMask版本
 */
export const getMetaMaskVersion = async () => {
  if (!isMetaMaskInstalled()) return null;
  
  try {
    const version = await window.ethereum.request({
      method: 'web3_clientVersion'
    });
    return version;
  } catch (error) {
    console.error('Failed to get MetaMask version:', error);
    return null;
  }
};

/**
 * 检查是否在移动端MetaMask浏览器中
 */
export const isMetaMaskMobile = () => {
  return typeof window !== 'undefined' && 
         window.ethereum && 
         window.ethereum.isMetaMask && 
         /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
           navigator.userAgent
         );
};

/**
 * 完整的环境检测
 */
export const detectWalletEnvironment = () => {
  const hasMetaMask = isMetaMaskInstalled();
  const isMobile = isMetaMaskMobile();
  
  // 检测其他钱包
  const hasCoinbase = window.ethereum?.isCoinbaseWallet;
  const hasBinance = window.BinanceChain !== undefined;
  
  return {
    hasMetaMask,
    isMobile,
    hasCoinbase,
    hasBinance,
    hasAnyWallet: hasMetaMask || hasCoinbase || hasBinance
  };
};
```

### 1.2 连接流程

```javascript
// hooks/useMetaMask.js
import { useState, useEffect, useCallback } from 'react';
import { ethers } from 'ethers';

export const useMetaMask = () => {
  const [account, setAccount] = useState(null);
  const [chainId, setChainId] = useState(null);
  const [provider, setProvider] = useState(null);
  const [signer, setSigner] = useState(null);
  const [isConnecting, setIsConnecting] = useState(false);
  const [error, setError] = useState(null);
  
  /**
   * 连接MetaMask
   */
  const connect = useCallback(async () => {
    if (!window.ethereum) {
      setError('MetaMask is not installed');
      // 引导用户安装
      window.open('https://metamask.io/download/', '_blank');
      return;
    }
    
    try {
      setIsConnecting(true);
      setError(null);
      
      // 请求账户访问
      const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
      });
      
      if (accounts.length === 0) {
        throw new Error('No accounts found');
      }
      
      // 创建provider和signer
      const ethersProvider = new ethers.BrowserProvider(window.ethereum);
      const ethersSigner = await ethersProvider.getSigner();
      
      // 获取网络信息
      const network = await ethersProvider.getNetwork();
      
      setAccount(accounts[0]);
      setChainId(Number(network.chainId));
      setProvider(ethersProvider);
      setSigner(ethersSigner);
      
      // 保存连接状态
      localStorage.setItem('metamask_connected', 'true');
      
      console.log('Connected to MetaMask:', {
        account: accounts[0],
        chainId: Number(network.chainId)
      });
      
    } catch (err) {
      console.error('Connection error:', err);
      
      // 处理不同的错误
      if (err.code === 4001) {
        setError('User rejected the connection request');
      } else if (err.code === -32002) {
        setError('Connection request already pending');
      } else {
        setError(err.message || 'Failed to connect to MetaMask');
      }
    } finally {
      setIsConnecting(false);
    }
  }, []);
  
  /**
   * 断开连接
   */
  const disconnect = useCallback(() => {
    setAccount(null);
    setChainId(null);
    setProvider(null);
    setSigner(null);
    localStorage.removeItem('metamask_connected');
  }, []);
  
  /**
   * 自动重连
   */
  useEffect(() => {
    const wasConnected = localStorage.getItem('metamask_connected');
    
    if (wasConnected === 'true' && window.ethereum) {
      // 检查是否已有连接的账户
      window.ethereum
        .request({ method: 'eth_accounts' })
        .then(accounts => {
          if (accounts.length > 0) {
            connect();
          }
        })
        .catch(console.error);
    }
  }, [connect]);
  
  return {
    account,
    chainId,
    provider,
    signer,
    isConnecting,
    error,
    isConnected: !!account,
    connect,
    disconnect
  };
};
```

---

## Part 2: 账户和网络管理 (1.5小时)

### 2.1 账户切换监听

```javascript
// hooks/useAccountListener.js
import { useEffect, useCallback } from 'react';

export const useAccountListener = (onAccountChange) => {
  const handleAccountsChanged = useCallback((accounts) => {
    console.log('Accounts changed:', accounts);
    
    if (accounts.length === 0) {
      // 用户断开了所有账户
      onAccountChange(null);
    } else if (accounts[0] !== onAccountChange.currentAccount) {
      // 用户切换了账户
      onAccountChange(accounts[0]);
    }
  }, [onAccountChange]);
  
  useEffect(() => {
    if (!window.ethereum) return;
    
    window.ethereum.on('accountsChanged', handleAccountsChanged);
    
    return () => {
      window.ethereum.removeListener('accountsChanged', handleAccountsChanged);
    };
  }, [handleAccountsChanged]);
};

// 使用示例
const MyComponent = () => {
  const [account, setAccount] = useState(null);
  
  useAccountListener(useCallback((newAccount) => {
    setAccount(newAccount);
    
    if (!newAccount) {
      // 处理断开连接
      console.log('User disconnected');
    } else {
      // 处理账户切换
      console.log('Switched to:', newAccount);
      // 重新加载用户数据
      loadUserData(newAccount);
    }
  }, []));
  
  return <div>Current account: {account}</div>;
};
```

### 2.2 网络管理

```javascript
// utils/networks.js

export const NETWORKS = {
  ETHEREUM_MAINNET: {
    chainId: '0x1',
    chainName: 'Ethereum Mainnet',
    nativeCurrency: {
      name: 'Ethereum',
      symbol: 'ETH',
      decimals: 18
    },
    rpcUrls: ['https://mainnet.infura.io/v3/YOUR-PROJECT-ID'],
    blockExplorerUrls: ['https://etherscan.io']
  },
  SEPOLIA: {
    chainId: '0xaa36a7',
    chainName: 'Sepolia Testnet',
    nativeCurrency: {
      name: 'Sepolia ETH',
      symbol: 'SEP',
      decimals: 18
    },
    rpcUrls: ['https://sepolia.infura.io/v3/YOUR-PROJECT-ID'],
    blockExplorerUrls: ['https://sepolia.etherscan.io']
  },
  POLYGON: {
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
  BSC: {
    chainId: '0x38',
    chainName: 'Binance Smart Chain',
    nativeCurrency: {
      name: 'BNB',
      symbol: 'BNB',
      decimals: 18
    },
    rpcUrls: ['https://bsc-dataseed1.binance.org/'],
    blockExplorerUrls: ['https://bscscan.com/']
  }
};

/**
 * 切换网络
 */
export const switchNetwork = async (chainId) => {
  if (!window.ethereum) {
    throw new Error('MetaMask is not installed');
  }
  
  try {
    await window.ethereum.request({
      method: 'wallet_switchEthereumChain',
      params: [{ chainId }]
    });
  } catch (error) {
    // 如果网络不存在，尝试添加
    if (error.code === 4902) {
      const networkConfig = Object.values(NETWORKS).find(
        n => n.chainId === chainId
      );
      
      if (!networkConfig) {
        throw new Error('Network configuration not found');
      }
      
      await addNetwork(networkConfig);
    } else {
      throw error;
    }
  }
};

/**
 * 添加新网络
 */
export const addNetwork = async (networkConfig) => {
  if (!window.ethereum) {
    throw new Error('MetaMask is not installed');
  }
  
  try {
    await window.ethereum.request({
      method: 'wallet_addEthereumChain',
      params: [networkConfig]
    });
  } catch (error) {
    console.error('Failed to add network:', error);
    throw error;
  }
};

/**
 * 获取当前网络信息
 */
export const getCurrentNetwork = async () => {
  if (!window.ethereum) return null;
  
  const chainId = await window.ethereum.request({
    method: 'eth_chainId'
  });
  
  return Object.values(NETWORKS).find(n => n.chainId === chainId);
};

// 网络切换Hook
import { useState, useEffect, useCallback } from 'react';

export const useNetwork = () => {
  const [currentChainId, setCurrentChainId] = useState(null);
  const [isChanging, setIsChanging] = useState(false);
  
  const handleChainChanged = useCallback((chainId) => {
    console.log('Chain changed:', chainId);
    setCurrentChainId(parseInt(chainId, 16));
    // 刷新页面以确保状态一致
    window.location.reload();
  }, []);
  
  useEffect(() => {
    if (!window.ethereum) return;
    
    // 获取当前链ID
    window.ethereum
      .request({ method: 'eth_chainId' })
      .then(chainId => setCurrentChainId(parseInt(chainId, 16)))
      .catch(console.error);
    
    // 监听链切换
    window.ethereum.on('chainChanged', handleChainChanged);
    
    return () => {
      window.ethereum.removeListener('chainChanged', handleChainChanged);
    };
  }, [handleChainChanged]);
  
  const switchToNetwork = useCallback(async (chainId) => {
    setIsChanging(true);
    
    try {
      await switchNetwork(chainId);
    } catch (error) {
      console.error('Failed to switch network:', error);
      throw error;
    } finally {
      setIsChanging(false);
    }
  }, []);
  
  return {
    currentChainId,
    isChanging,
    switchToNetwork
  };
};
```

---

## Part 3: 交易签名 (1.5小时)

### 3.1 发送交易

```javascript
// utils/transactions.js
import { ethers } from 'ethers';

/**
 * 发送ETH
 */
export const sendETH = async (signer, to, amount) => {
  try {
    const tx = await signer.sendTransaction({
      to,
      value: ethers.parseEther(amount)
    });
    
    console.log('Transaction sent:', tx.hash);
    
    // 等待确认
    const receipt = await tx.wait();
    console.log('Transaction confirmed:', receipt);
    
    return receipt;
  } catch (error) {
    console.error('Transaction failed:', error);
    throw error;
  }
};

/**
 * 发送ERC20代币
 */
export const sendToken = async (signer, tokenAddress, to, amount) => {
  const tokenABI = [
    'function transfer(address to, uint256 amount) returns (bool)',
    'function decimals() view returns (uint8)'
  ];
  
  const tokenContract = new ethers.Contract(tokenAddress, tokenABI, signer);
  
  try {
    // 获取代币精度
    const decimals = await tokenContract.decimals();
    const value = ethers.parseUnits(amount, decimals);
    
    // 发送交易
    const tx = await tokenContract.transfer(to, value);
    console.log('Token transfer sent:', tx.hash);
    
    const receipt = await tx.wait();
    console.log('Token transfer confirmed:', receipt);
    
    return receipt;
  } catch (error) {
    console.error('Token transfer failed:', error);
    throw error;
  }
};

/**
 * 带Gas估算的交易
 */
export const sendWithGasEstimate = async (signer, txParams) => {
  try {
    // 估算Gas
    const gasLimit = await signer.estimateGas(txParams);
    console.log('Estimated gas:', gasLimit.toString());
    
    // 获取当前Gas价格
    const feeData = await signer.provider.getFeeData();
    
    // 发送交易
    const tx = await signer.sendTransaction({
      ...txParams,
      gasLimit: gasLimit * 120n / 100n, // +20% buffer
      maxFeePerGas: feeData.maxFeePerGas,
      maxPriorityFeePerGas: feeData.maxPriorityFeePerGas
    });
    
    return tx;
  } catch (error) {
    console.error('Transaction failed:', error);
    throw error;
  }
};

/**
 * 批量交易
 */
export const sendBatchTransactions = async (signer, transactions) => {
  const results = [];
  
  for (let i = 0; i < transactions.length; i++) {
    const tx = transactions[i];
    
    try {
      const txResponse = await signer.sendTransaction({
        ...tx,
        nonce: (await signer.getNonce()) + i
      });
      
      results.push({
        success: true,
        hash: txResponse.hash,
        transaction: tx
      });
      
    } catch (error) {
      results.push({
        success: false,
        error: error.message,
        transaction: tx
      });
    }
  }
  
  return results;
};
```

### 3.2 交易监控

```javascript
// components/TransactionMonitor.jsx
import React, { useState, useEffect } from 'react';
import { ethers } from 'ethers';

const TransactionMonitor = ({ provider, txHash }) => {
  const [status, setStatus] = useState('pending');
  const [receipt, setReceipt] = useState(null);
  const [confirmations, setConfirmations] = useState(0);
  
  useEffect(() => {
    if (!provider || !txHash) return;
    
    let cancelled = false;
    
    const monitorTransaction = async () => {
      try {
        // 等待交易被挖矿
        const receipt = await provider.waitForTransaction(txHash, 1);
        
        if (cancelled) return;
        
        setReceipt(receipt);
        setStatus(receipt.status === 1 ? 'success' : 'failed');
        
        // 持续监控确认数
        const updateConfirmations = async () => {
          const currentBlock = await provider.getBlockNumber();
          const confs = currentBlock - receipt.blockNumber + 1;
          setConfirmations(confs);
          
          if (confs < 12 && !cancelled) {
            setTimeout(updateConfirmations, 15000); // 每15秒更新
          }
        };
        
        updateConfirmations();
        
      } catch (error) {
        if (!cancelled) {
          setStatus('error');
          console.error('Transaction monitoring error:', error);
        }
      }
    };
    
    monitorTransaction();
    
    return () => {
      cancelled = true;
    };
  }, [provider, txHash]);
  
  return (
    <div className="transaction-monitor">
      <div className="tx-hash">
        <span>Transaction: </span>
        <a 
          href={`https://etherscan.io/tx/${txHash}`}
          target="_blank"
          rel="noopener noreferrer"
        >
          {txHash.slice(0, 10)}...{txHash.slice(-8)}
        </a>
      </div>
      
      <div className={`tx-status status-${status}`}>
        {status === 'pending' && '⏳ Pending...'}
        {status === 'success' && '✅ Success'}
        {status === 'failed' && '❌ Failed'}
        {status === 'error' && '⚠️ Error'}
      </div>
      
      {receipt && (
        <div className="tx-details">
          <p>Block: {receipt.blockNumber}</p>
          <p>Gas Used: {receipt.gasUsed.toString()}</p>
          <p>Confirmations: {confirmations}/12</p>
        </div>
      )}
    </div>
  );
};

export default TransactionMonitor;
```

---

## Part 4: 消息签名 (1小时)

### 4.1 个人签名

```javascript
// utils/signing.js

/**
 * 个人消息签名 (EIP-191)
 */
export const signPersonalMessage = async (signer, message) => {
  try {
    const signature = await signer.signMessage(message);
    console.log('Signature:', signature);
    return signature;
  } catch (error) {
    console.error('Signing failed:', error);
    throw error;
  }
};

/**
 * 验证签名
 */
export const verifySignature = (message, signature, expectedAddress) => {
  try {
    const recoveredAddress = ethers.verifyMessage(message, signature);
    return recoveredAddress.toLowerCase() === expectedAddress.toLowerCase();
  } catch (error) {
    console.error('Verification failed:', error);
    return false;
  }
};

/**
 * 登录验证示例
 */
export const signInWithEthereum = async (signer, nonce) => {
  const address = await signer.getAddress();
  const message = `Sign in to MyDApp\n\nAddress: ${address}\nNonce: ${nonce}`;
  
  const signature = await signPersonalMessage(signer, message);
  
  return {
    address,
    message,
    signature
  };
};
```

### 4.2 类型化数据签名 (EIP-712)

```javascript
/**
 * EIP-712类型化数据签名
 */
export const signTypedData = async (signer, domain, types, value) => {
  try {
    const signature = await signer.signTypedData(domain, types, value);
    return signature;
  } catch (error) {
    console.error('Typed data signing failed:', error);
    throw error;
  }
};

/**
 * 许可签名示例 (ERC20 Permit)
 */
export const signPermit = async (
  signer,
  tokenAddress,
  spenderAddress,
  amount,
  deadline
) => {
  const domain = {
    name: 'MyToken',
    version: '1',
    chainId: (await signer.provider.getNetwork()).chainId,
    verifyingContract: tokenAddress
  };
  
  const types = {
    Permit: [
      { name: 'owner', type: 'address' },
      { name: 'spender', type: 'address' },
      { name: 'value', type: 'uint256' },
      { name: 'nonce', type: 'uint256' },
      { name: 'deadline', type: 'uint256' }
    ]
  };
  
  const ownerAddress = await signer.getAddress();
  
  // 获取nonce（需要从合约读取）
  const tokenContract = new ethers.Contract(
    tokenAddress,
    ['function nonces(address) view returns (uint256)'],
    signer
  );
  const nonce = await tokenContract.nonces(ownerAddress);
  
  const value = {
    owner: ownerAddress,
    spender: spenderAddress,
    value: amount,
    nonce: nonce,
    deadline: deadline
  };
  
  const signature = await signTypedData(signer, domain, types, value);
  
  return {
    ...value,
    signature
  };
};

// 使用示例组件
import React, { useState } from 'react';
import { useWalletContext } from '../contexts/WalletContext';

const SignatureDemo = () => {
  const { signer, account } = useWalletContext();
  const [message, setMessage] = useState('');
  const [signature, setSignature] = useState('');
  const [verified, setVerified] = useState(null);
  
  const handleSign = async () => {
    if (!signer || !message) return;
    
    const sig = await signPersonalMessage(signer, message);
    setSignature(sig);
  };
  
  const handleVerify = () => {
    if (!signature || !message || !account) return;
    
    const isValid = verifySignature(message, signature, account);
    setVerified(isValid);
  };
  
  return (
    <div className="signature-demo">
      <h2>Message Signing Demo</h2>
      
      <textarea
        value={message}
        onChange={(e) => setMessage(e.target.value)}
        placeholder="Enter message to sign"
      />
      
      <button onClick={handleSign}>Sign Message</button>
      
      {signature && (
        <div className="signature-result">
          <h3>Signature:</h3>
          <code>{signature}</code>
          
          <button onClick={handleVerify}>Verify Signature</button>
          
          {verified !== null && (
            <div className={verified ? 'valid' : 'invalid'}>
              {verified ? '✅ Valid' : '❌ Invalid'}
            </div>
          )}
        </div>
      )}
    </div>
  );
};

export default SignatureDemo;
```

---

## Part 5: 高级功能 (1.5小时)

### 5.1 添加代币到MetaMask

```javascript
/**
 * 添加代币到MetaMask钱包
 */
export const addTokenToWallet = async (tokenAddress, tokenSymbol, tokenDecimals, tokenImage) => {
  if (!window.ethereum) {
    throw new Error('MetaMask not installed');
  }
  
  try {
    const wasAdded = await window.ethereum.request({
      method: 'wallet_watchAsset',
      params: {
        type: 'ERC20',
        options: {
          address: tokenAddress,
          symbol: tokenSymbol,
          decimals: tokenDecimals,
          image: tokenImage
        }
      }
    });
    
    if (wasAdded) {
      console.log('Token added successfully');
    }
    
    return wasAdded;
  } catch (error) {
    console.error('Failed to add token:', error);
    throw error;
  }
};

// 使用示例
const AddTokenButton = ({ tokenInfo }) => {
  const [adding, setAdding] = useState(false);
  
  const handleAddToken = async () => {
    setAdding(true);
    
    try {
      await addTokenToWallet(
        tokenInfo.address,
        tokenInfo.symbol,
        tokenInfo.decimals,
        tokenInfo.logoUrl
      );
      alert('Token added to MetaMask!');
    } catch (error) {
      alert('Failed to add token: ' + error.message);
    } finally {
      setAdding(false);
    }
  };
  
  return (
    <button onClick={handleAddToken} disabled={adding}>
      {adding ? 'Adding...' : 'Add to MetaMask'}
    </button>
  );
};
```

### 5.2 权限管理

```javascript
/**
 * 请求权限
 */
export const requestPermissions = async (permissions = ['eth_accounts']) => {
  try {
    const granted = await window.ethereum.request({
      method: 'wallet_requestPermissions',
      params: [
        {
          eth_accounts: {}
        }
      ]
    });
    
    console.log('Permissions granted:', granted);
    return granted;
  } catch (error) {
    console.error('Permission request failed:', error);
    throw error;
  }
};

/**
 * 获取当前权限
 */
export const getPermissions = async () => {
  try {
    const permissions = await window.ethereum.request({
      method: 'wallet_getPermissions'
    });
    
    return permissions;
  } catch (error) {
    console.error('Failed to get permissions:', error);
    return [];
  }
};

/**
 * 撤销权限
 */
export const revokePermissions = async () => {
  try {
    await window.ethereum.request({
      method: 'wallet_revokePermissions',
      params: [
        {
          eth_accounts: {}
        }
      ]
    });
    
    console.log('Permissions revoked');
  } catch (error) {
    console.error('Failed to revoke permissions:', error);
    throw error;
  }
};
```

### 5.3 完整集成示例

```javascript
// components/MetaMaskIntegration.jsx
import React, { useState, useEffect } from 'react';
import { ethers } from 'ethers';
import { 
  isMetaMaskInstalled, 
  detectWalletEnvironment 
} from '../utils/metamask';
import { NETWORKS, switchNetwork } from '../utils/networks';
import { signPersonalMessage } from '../utils/signing';

const MetaMaskIntegration = () => {
  const [environment, setEnvironment] = useState(null);
  const [account, setAccount] = useState(null);
  const [chainId, setChainId] = useState(null);
  const [balance, setBalance] = useState('0');
  const [provider, setProvider] = useState(null);
  
  useEffect(() => {
    setEnvironment(detectWalletEnvironment());
  }, []);
  
  const connect = async () => {
    if (!environment?.hasMetaMask) {
      alert('Please install MetaMask!');
      return;
    }
    
    try {
      const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
      });
      
      const ethersProvider = new ethers.BrowserProvider(window.ethereum);
      const network = await ethersProvider.getNetwork();
      const balance = await ethersProvider.getBalance(accounts[0]);
      
      setAccount(accounts[0]);
      setChainId(Number(network.chainId));
      setBalance(ethers.formatEther(balance));
      setProvider(ethersProvider);
      
    } catch (error) {
      console.error('Connection failed:', error);
    }
  };
  
  const handleSwitchNetwork = async (targetChainId) => {
    try {
      await switchNetwork(targetChainId);
    } catch (error) {
      console.error('Network switch failed:', error);
    }
  };
  
  const handleSignMessage = async () => {
    if (!provider) return;
    
    const signer = await provider.getSigner();
    const message = 'Hello from MetaMask!';
    
    try {
      const signature = await signPersonalMessage(signer, message);
      alert(`Signature: ${signature}`);
    } catch (error) {
      console.error('Signing failed:', error);
    }
  };
  
  if (!environment) {
    return <div>Loading...</div>;
  }
  
  if (!environment.hasMetaMask) {
    return (
      <div className="metamask-not-found">
        <h2>MetaMask Not Found</h2>
        <p>Please install MetaMask to use this app.</p>
        <a 
          href="https://metamask.io/download/"
          target="_blank"
          rel="noopener noreferrer"
        >
          Download MetaMask
        </a>
      </div>
    );
  }
  
  return (
    <div className="metamask-integration">
      <h1>MetaMask Integration Demo</h1>
      
      {!account ? (
        <button onClick={connect} className="btn-connect">
          Connect MetaMask
        </button>
      ) : (
        <div className="wallet-info">
          <div className="info-item">
            <label>Account:</label>
            <span>{account.slice(0, 6)}...{account.slice(-4)}</span>
          </div>
          
          <div className="info-item">
            <label>Chain ID:</label>
            <span>{chainId}</span>
          </div>
          
          <div className="info-item">
            <label>Balance:</label>
            <span>{parseFloat(balance).toFixed(4)} ETH</span>
          </div>
          
          <div className="actions">
            <button onClick={handleSignMessage}>
              Sign Message
            </button>
            
            <select onChange={(e) => handleSwitchNetwork(e.target.value)}>
              <option value="">Switch Network</option>
              {Object.entries(NETWORKS).map(([key, network]) => (
                <option key={key} value={network.chainId}>
                  {network.chainName}
                </option>
              ))}
            </select>
          </div>
        </div>
      )}
    </div>
  );
};

export default MetaMaskIntegration;
```

---

## 📝 今日作业

### 作业1: 完整钱包集成

实现完整的MetaMask集成：
1. 检测与连接
2. 账户管理
3. 网络切换
4. 交易发送

### 作业2: 签名功能

实现各种签名功能：
1. 个人消息签名
2. 类型化数据签名
3. 签名验证
4. 登录认证

### 作业3: 错误处理

完善错误处理机制：
1. 连接错误
2. 交易错误
3. 签名错误
4. 网络错误

---

## ✅ 检查清单

- [ ] 掌握MetaMask API
- [ ] 实现交易签名
- [ ] 理解消息签名
- [ ] 完成多链切换
- [ ] 处理各种事件

---

## 📅 明日预告

明天学习WalletConnect：
- WalletConnect协议
- 多钱包支持
- 二维码连接
- 移动端适配

**🎉 完成Day 3！继续前进！**
