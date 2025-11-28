# Week 7 - Day 6-7: React DApp综合项目

**学习日期**: ___________
**预计用时**: 12-14小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 项目目标

构建一个生产级去中心化交易所(DEX)前端：
- ✅ 集成Uniswap V2核心逻辑
- ✅ 支持Swap和Add Liquidity
- ✅ 实时价格图表 (TradingView)
- ✅ 多钱包支持 (MetaMask/WalletConnect)
- ✅ 交易通知与历史记录
- ✅ 响应式设计与暗色模式

---

## Part 1: 项目架构 (2小时)

### 1.1 目录结构

```
src/
├── assets/             # 静态资源
├── components/         # 通用组件
│   ├── common/        # 按钮、输入框等
│   ├── layout/        # Header, Footer
│   ├── web3/          # 钱包连接、网络切换
│   └── swap/          # 交易相关组件
├── config/            # 配置文件
├── constants/         # 常量定义
├── contexts/          # React Context
├── hooks/             # 自定义Hooks
├── pages/             # 页面组件
├── state/             # Redux/Zustand状态
├── theme/             # 样式主题
└── utils/             # 工具函数
```

### 1.2 核心依赖

```json
{
  "dependencies": {
    "@chakra-ui/react": "^2.8.0",
    "@emotion/react": "^11.11.0",
    "@emotion/styled": "^11.11.0",
    "@tanstack/react-query": "^5.0.0",
    "@uniswap/sdk": "^3.0.3",
    "@web3-react/core": "^6.1.9",
    "@web3-react/injected-connector": "^6.0.7",
    "ethers": "^6.9.0",
    "framer-motion": "^10.16.0",
    "lightweight-charts": "^4.1.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0",
    "zustand": "^4.4.7"
  }
}
```

---

## Part 2: 核心业务逻辑 (3小时)

### 2.1 Swap Hook

```javascript
// hooks/useSwapCallback.js
import { useCallback } from 'react';
import { ethers } from 'ethers';
import { useWeb3React } from '@web3-react/core';
import { ROUTER_ADDRESS } from '../constants';
import ROUTER_ABI from '../constants/abis/router.json';

export function useSwapCallback(
  trade, // Uniswap SDK Trade object
  allowedSlippage = 0.5, // 0.5%
  recipient
) {
  const { account, library } = useWeb3React();

  return useCallback(async () => {
    if (!trade || !library || !account) return;

    const router = new ethers.Contract(ROUTER_ADDRESS, ROUTER_ABI, library.getSigner());
    
    const amountIn = ethers.parseUnits(trade.inputAmount.toExact(), trade.inputAmount.currency.decimals);
    const amountOutMin = ethers.parseUnits(
      trade.minimumAmountOut(allowedSlippage).toExact(),
      trade.outputAmount.currency.decimals
    );
    
    const path = trade.route.path.map(token => token.address);
    const deadline = Math.floor(Date.now() / 1000) + 60 * 20; // 20 minutes
    
    try {
      const tx = await router.swapExactTokensForTokens(
        amountIn,
        amountOutMin,
        path,
        recipient || account,
        deadline
      );
      
      return tx;
    } catch (error) {
      console.error('Swap failed:', error);
      throw error;
    }
  }, [trade, library, account, allowedSlippage, recipient]);
}
```

### 2.2 价格查询 Hook

```javascript
// hooks/useTokenPrice.js
import { useQuery } from '@tanstack/react-query';
import { Route, Token, WETH, Fetcher } from '@uniswap/sdk';
import { useWeb3React } from '@web3-react/core';

export function useTokenPrice(tokenAddress) {
  const { chainId, library } = useWeb3React();

  return useQuery({
    queryKey: ['tokenPrice', tokenAddress, chainId],
    queryFn: async () => {
      if (!tokenAddress || !chainId) return null;
      
      const token = new Token(chainId, tokenAddress, 18);
      const pair = await Fetcher.fetchPairData(token, WETH[chainId], library);
      const route = new Route([pair], WETH[chainId]);
      
      return route.midPrice.toSignificant(6);
    },
    refetchInterval: 10000 // 10s refresh
  });
}
```

---

## Part 3: UI组件开发 (3小时)

### 3.1 交易面板组件

```javascript
// components/swap/SwapPanel.jsx
import React, { useState } from 'react';
import { Box, Button, VStack, useToast } from '@chakra-ui/react';
import { TokenInput } from './TokenInput';
import { useSwapCallback } from '../../hooks/useSwapCallback';
import { SettingsModal } from './SettingsModal';

export const SwapPanel = () => {
  const [inputToken, setInputToken] = useState(null);
  const [outputToken, setOutputToken] = useState(null);
  const [amountIn, setAmountIn] = useState('');
  const [slippage, setSlippage] = useState(0.5);
  
  const toast = useToast();
  
  // 假设这里已经通过Hooks计算出了trade对象
  const trade = useTrade(inputToken, outputToken, amountIn);
  const swapCallback = useSwapCallback(trade, slippage);
  
  const handleSwap = async () => {
    try {
      const tx = await swapCallback();
      toast({
        title: 'Swap Submitted',
        description: `Tx Hash: ${tx.hash}`,
        status: 'info',
      });
      
      await tx.wait();
      toast({
        title: 'Swap Confirmed',
        status: 'success',
      });
    } catch (err) {
      toast({
        title: 'Swap Failed',
        description: err.message,
        status: 'error',
      });
    }
  };

  return (
    <Box 
      w="480px" 
      bg="bg.surface" 
      p={4} 
      borderRadius="2xl" 
      boxShadow="lg"
    >
      <VStack spacing={4}>
        <SettingsModal slippage={slippage} setSlippage={setSlippage} />
        
        <TokenInput
          label="Pay"
          token={inputToken}
          amount={amountIn}
          onAmountChange={setAmountIn}
          onTokenSelect={setInputToken}
        />
        
        <Button variant="ghost" onClick={() => {
          // switch tokens logic
        }}>↓</Button>
        
        <TokenInput
          label="Receive"
          token={outputToken}
          amount={trade?.outputAmount?.toSignificant(6) || ''}
          onTokenSelect={setOutputToken}
          readOnly
        />
        
        <Button 
          w="full" 
          size="lg" 
          colorScheme="brand" 
          onClick={handleSwap}
          isDisabled={!trade}
        >
          {trade ? 'Swap' : 'Select Tokens'}
        </Button>
      </VStack>
    </Box>
  );
};
```

### 3.2 K线图表组件

使用TradingView轻量级图表库。

```javascript
// components/swap/PriceChart.jsx
import React, { useEffect, useRef } from 'react';
import { createChart } from 'lightweight-charts';
import { Box } from '@chakra-ui/react';

export const PriceChart = ({ data }) => {
  const chartContainerRef = useRef();
  const chartRef = useRef();

  useEffect(() => {
    if (!chartContainerRef.current) return;

    const chart = createChart(chartContainerRef.current, {
      width: chartContainerRef.current.clientWidth,
      height: 400,
      layout: {
        backgroundColor: '#1a202c',
        textColor: '#d1d4dc',
      },
      grid: {
        vertLines: { color: '#2B2B43' },
        horzLines: { color: '#2B2B43' },
      },
    });

    const candlestickSeries = chart.addCandlestickSeries({
      upColor: '#26a69a',
      downColor: '#ef5350',
      borderVisible: false,
      wickUpColor: '#26a69a',
      wickDownColor: '#ef5350',
    });

    candlestickSeries.setData(data);
    chartRef.current = chart;

    return () => chart.remove();
  }, [data]);

  return <Box ref={chartContainerRef} w="full" h="400px" />;
};
```

---

## Part 4: 状态管理与缓存 (2小时)

### 4.1 交易历史Store

```javascript
// state/transactions.js
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useTransactionStore = create(
  persist(
    (set, get) => ({
      transactions: {},
      
      addTransaction: (chainId, hash, summary) => {
        set(state => ({
          transactions: {
            ...state.transactions,
            [chainId]: {
              ...state.transactions[chainId],
              [hash]: { hash, summary, addedTime: Date.now(), status: 'pending' }
            }
          }
        }));
      },
      
      updateTransaction: (chainId, hash, status) => {
        set(state => {
          const txs = { ...state.transactions };
          if (txs[chainId] && txs[chainId][hash]) {
            txs[chainId][hash].status = status;
          }
          return { transactions: txs };
        });
      }
    }),
    { name: 'transactions-storage' }
  )
);
```

---

## Part 5: 综合调试与优化 (2小时)

### 5.1 性能优化

1. **代码分割**: 使用 `React.lazy` 和 `Suspense` 加载不同页面。
2. **Web3调用优化**: 使用 `Multicall` 批量查询余额。
3. **缓存**: 对Token列表、价格数据进行缓存配置 (React Query)。

### 5.2 错误边界

```javascript
// components/ErrorBoundary.jsx
class ErrorBoundary extends React.Component {
  // ... implementation from previous lessons
  render() {
    if (this.state.hasError) {
      return (
        <Box textAlign="center" py={10} px={6}>
          <Heading as="h2" size="xl" mt={6} mb={2}>
            Something went wrong
          </Heading>
          <Text color={'gray.500'}>
            {this.state.error?.message}
          </Text>
          <Button
            colorScheme="teal"
            bgGradient="linear(to-r, teal.400, teal.500, teal.600)"
            color="white"
            variant="solid"
            onClick={() => window.location.reload()}
          >
            Reload Page
          </Button>
        </Box>
      );
    }
    return this.props.children;
  }
}
```

---

## 📝 项目总结

### 核心功能完成度
1. **Swap**: 核心交易逻辑 (通过Uniswap Router)
2. **Liquidity**: 添加/移除流动性
3. **Wallet**: 多钱包连接与状态管理
4. **Info**: 实时价格与图表

### 扩展方向
- 集成更多DEX聚合器 (1inch API)
- 支持Limit Order (限价单)
- 跨链Swap支持
- 移动端PWA适配

---

## ✅ 检查清单

- [ ] 能够连接钱包并显示余额
- [ ] 能够正确估算Swap价格和滑点
- [ ] 成功执行Swap交易
- [ ] 交易历史正确记录并持久化
- [ ] 界面在不同设备上显示正常

---

## 📅 下周预告

下周将进入NFT Marketplace开发：
- ERC721/1155标准深入
- IPFS元数据存储
- NFT展示与交易逻辑
- 拍卖系统实现

**🎉 恭喜！你已经完成了React DApp开发周，具备了独立开发DeFi应用的能力！**
