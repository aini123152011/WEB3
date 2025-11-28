# Week 7 - Day 5: 实时数据更新

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 学习目标

- ✅ 理解实时数据的重要性
- ✅ 掌握WebSocket订阅
- ✅ 学习Provider事件监听
- ✅ 实现价格实时流
- ✅ 构建通知中心

---

## Part 1: Web3事件监听 (1.5小时)

### 1.1 Provider事件

```javascript
// hooks/useBlockNumber.js
import { useState, useEffect } from 'react';
import { useWeb3React } from '@web3-react/core';

export function useBlockNumber() {
  const { library } = useWeb3React();
  const [blockNumber, setBlockNumber] = useState(0);

  useEffect(() => {
    if (!library) return;

    const updateBlockNumber = (blockNumber) => {
      setBlockNumber(blockNumber);
    };

    // 监听新区块
    library.on('block', updateBlockNumber);

    // 获取当前区块
    library.getBlockNumber().then(updateBlockNumber);

    return () => {
      library.removeListener('block', updateBlockNumber);
    };
  }, [library]);

  return blockNumber;
}
```

### 1.2 合约事件过滤

```javascript
// hooks/useEventFilter.js
import { useState, useEffect } from 'react';
import { ethers } from 'ethers';

export function useTransferEvents(tokenContract, address) {
  const [events, setEvents] = useState([]);

  useEffect(() => {
    if (!tokenContract || !address) return;

    // 创建过滤器：From OR To == address
    const filterFrom = tokenContract.filters.Transfer(address, null);
    const filterTo = tokenContract.filters.Transfer(null, address);

    const handleTransfer = (from, to, amount, event) => {
      const newEvent = {
        from,
        to,
        amount: ethers.formatEther(amount),
        hash: event.transactionHash,
        blockNumber: event.blockNumber,
        timestamp: Date.now()
      };
      
      setEvents(prev => [newEvent, ...prev]);
    };

    tokenContract.on(filterFrom, handleTransfer);
    tokenContract.on(filterTo, handleTransfer);

    return () => {
      tokenContract.off(filterFrom, handleTransfer);
      tokenContract.off(filterTo, handleTransfer);
    };
  }, [tokenContract, address]);

  return events;
}
```

---

## Part 2: WebSocket集成 (2小时)

### 2.1 配置WebSocket Provider

```javascript
// utils/provider.js
import { ethers } from 'ethers';

export const getWebSocketProvider = (chainId) => {
  const wsUrls = {
    1: 'wss://mainnet.infura.io/ws/v3/YOUR_PROJECT_ID',
    5: 'wss://goerli.infura.io/ws/v3/YOUR_PROJECT_ID'
  };

  const url = wsUrls[chainId];
  if (!url) return null;

  return new ethers.WebSocketProvider(url);
};
```

### 2.2 实时价格流 (Chainlink)

```javascript
// hooks/usePriceFeed.js
import { useState, useEffect } from 'react';
import { ethers } from 'ethers';

const AGGREGATOR_ABI = [
  'event AnswerUpdated(int256 indexed current, uint256 indexed roundId, uint256 updatedAt)',
  'function latestRoundData() view returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)'
];

export function useEthPrice(provider) {
  const [price, setPrice] = useState(null);

  useEffect(() => {
    if (!provider) return;

    // ETH/USD Aggregator Address
    const address = '0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419';
    const aggregator = new ethers.Contract(address, AGGREGATOR_ABI, provider);

    const handleUpdate = (current, roundId, updatedAt) => {
      // Chainlink价格通常有8位小数
      setPrice(ethers.formatUnits(current, 8));
    };

    // 订阅更新事件
    aggregator.on('AnswerUpdated', handleUpdate);

    // 初始化
    aggregator.latestRoundData().then((data) => {
      setPrice(ethers.formatUnits(data.answer, 8));
    });

    return () => {
      aggregator.off('AnswerUpdated', handleUpdate);
    };
  }, [provider]);

  return price;
}
```

---

## Part 3: 数据轮询策略 (1.5小时)

### 3.1 SWR轮询

使用SWR (Stale-While-Revalidate) 库进行智能轮询。

```javascript
import useSWR from 'swr';

const fetcher = (library) => (...args) => {
  const [method, ...params] = args;
  return library[method](...params);
};

export function useEthBalance(address) {
  const { library } = useWeb3React();

  const { data, mutate } = useSWR(
    library ? ['getBalance', address, 'latest'] : null,
    {
      fetcher: fetcher(library),
      refreshInterval: 10000, // 每10秒轮询
      dedupingInterval: 5000, // 防抖
    }
  );

  useEffect(() => {
    if (library) {
      // 监听新区块，触发SWR重新验证
      library.on('block', () => {
        mutate();
      });
      
      return () => {
        library.removeAllListeners('block');
      };
    }
  }, [library, mutate]);

  return data;
}
```

---

## Part 4: 通知中心实战 (1小时)

### 4.1 Toast通知

使用 `react-hot-toast` 构建通知系统。

```javascript
// components/Notification.js
import toast from 'react-hot-toast';

export const notify = {
  success: (msg, hash) => toast.success(
    <div>
      <p>{msg}</p>
      {hash && (
        <a 
          href={`https://etherscan.io/tx/${hash}`} 
          target="_blank"
          rel="noopener noreferrer"
          className="text-xs underline"
        >
          View on Etherscan
        </a>
      )}
    </div>
  ),
  error: (msg) => toast.error(msg),
  loading: (msg) => toast.loading(msg)
};
```

### 4.2 交易状态监控

```javascript
// hooks/useTransactionMonitor.js
import { useEffect } from 'react';
import { notify } from '../components/Notification';

export function useTransactionMonitor(provider, txHash) {
  useEffect(() => {
    if (!provider || !txHash) return;

    const monitor = async () => {
      const toastId = notify.loading('Transaction Pending...');
      
      try {
        const receipt = await provider.waitForTransaction(txHash);
        
        toast.dismiss(toastId);
        if (receipt.status === 1) {
          notify.success('Transaction Confirmed!', txHash);
        } else {
          notify.error('Transaction Failed!');
        }
      } catch (error) {
        toast.dismiss(toastId);
        notify.error('Transaction Error: ' + error.message);
      }
    };

    monitor();
  }, [provider, txHash]);
}
```

---

## 📝 今日作业

### 作业1: 实时价格看板

创建一个Dashboard，包含：
1. 实时ETH/USD价格（WebSocket更新）
2. 当前Gas Price（每区块更新）
3. 最新区块高度（实时更新）

### 作业2: 交易通知系统

实现一个全局通知系统：
1. 监听所有待处理交易
2. 交易完成时弹出通知
3. 点击通知跳转到Etherscan
4. 支持浏览器原生通知 API

### 作业3: 余额自动刷新

实现余额自动刷新机制：
1. 页面聚焦时立即刷新
2. 新区块产生时刷新
3. 交易发送后乐观更新
4. 交易确认后强制刷新

---

## ✅ 检查清单

- [ ] 掌握WebSocket Provider的使用
- [ ] 理解SWR/React Query轮询机制
- [ ] 能够监听合约事件
- [ ] 实现完善的通知系统
- [ ] 优化数据刷新策略

---

## 📅 周末预告

周末进行React DApp综合项目：
- 整合Week 7所学知识
- 构建一个完整的Token Swap应用
- 实现多链支持
- 集成实时数据与通知

**🎉 完成Day 5！你的应用现在有了生命！**
