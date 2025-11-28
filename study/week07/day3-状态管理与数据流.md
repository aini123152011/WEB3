# Week 7 - Day 3: 状态管理与数据流

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 掌握Zustand状态管理
- ✅ 学习React Query (TanStack Query)
- ✅ 理解Web3数据缓存策略
- ✅ 实现乐观UI更新
- ✅ 优化数据加载性能

---

## Part 1: Zustand状态管理 (1.5小时)

Zustand是一个轻量级、现代化的状态管理库，非常适合React DApp。

### 1.1 基础Store

```javascript
// stores/useAppStore.js
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useAppStore = create(
  persist(
    (set) => ({
      // 主题设置
      theme: 'light',
      toggleTheme: () => set((state) => ({ 
        theme: state.theme === 'light' ? 'dark' : 'light' 
      })),
      
      // 交易记录
      transactions: [],
      addTransaction: (tx) => set((state) => ({ 
        transactions: [tx, ...state.transactions] 
      })),
      clearTransactions: () => set({ transactions: [] }),
      
      // 用户偏好
      settings: {
        slippage: 0.5,
        deadline: 20
      },
      updateSettings: (newSettings) => set((state) => ({
        settings: { ...state.settings, ...newSettings }
      }))
    }),
    {
      name: 'app-storage', // LocalStorage key
      partialize: (state) => ({ 
        theme: state.theme,
        settings: state.settings 
      }) // 只持久化部分状态
    }
  )
);
```

### 1.2 异步操作

```javascript
// stores/useTokenStore.js
import { create } from 'zustand';
import { ethers } from 'ethers';

export const useTokenStore = create((set, get) => ({
  tokens: [],
  loading: false,
  error: null,
  
  fetchTokens: async (provider, address) => {
    set({ loading: true, error: null });
    try {
      // 模拟获取代币列表
      const tokenList = await fetchTokenList(address);
      
      // 并行获取余额
      const tokensWithBalance = await Promise.all(
        tokenList.map(async (token) => {
          const contract = new ethers.Contract(
            token.address,
            ['function balanceOf(address) view returns (uint256)'],
            provider
          );
          const balance = await contract.balanceOf(address);
          return { ...token, balance: ethers.formatUnits(balance, token.decimals) };
        })
      );
      
      set({ tokens: tokensWithBalance, loading: false });
    } catch (error) {
      set({ error: error.message, loading: false });
    }
  }
}));
```

---

## Part 2: React Query集成 (2小时)

React Query是处理异步数据的神器，特别适合区块链数据。

### 2.1 配置QueryClient

```javascript
// App.js
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      refetchOnWindowFocus: false,
      staleTime: 10000, // 数据10秒后过期
      cacheTime: 60000, // 缓存保留1分钟
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MyComponent />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

### 2.2 Web3数据获取 Hook

```javascript
// hooks/useTokenBalance.js
import { useQuery } from '@tanstack/react-query';
import { ethers } from 'ethers';
import { useWeb3React } from '@web3-react/core';

export function useTokenBalance(tokenAddress) {
  const { account, library } = useWeb3React();
  
  return useQuery({
    queryKey: ['tokenBalance', account, tokenAddress],
    queryFn: async () => {
      if (!account || !library || !tokenAddress) return '0';
      
      const contract = new ethers.Contract(
        tokenAddress,
        ['function balanceOf(address) view returns (uint256)'],
        library
      );
      
      const balance = await contract.balanceOf(account);
      return ethers.formatEther(balance);
    },
    enabled: !!account && !!library && !!tokenAddress, // 依赖项存在时才执行
    refetchInterval: 15000, // 每15秒轮询一次
  });
}
```

---

## Part 3: 乐观UI更新 (1.5小时)

### 3.1 什么是乐观UI

乐观UI（Optimistic UI）是指在服务器确认之前，先在界面上显示操作成功的状态，从而提升用户体验。

### 3.2 实现乐观更新 (useMutation)

```javascript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function useSendTransaction() {
  const queryClient = useQueryClient();
  const { library } = useWeb3React();
  
  return useMutation({
    mutationFn: async ({ to, amount }) => {
      const signer = library.getSigner();
      const tx = await signer.sendTransaction({
        to,
        value: ethers.parseEther(amount)
      });
      return tx;
    },
    // 提交时触发
    onMutate: async (newTx) => {
      await queryClient.cancelQueries(['balance']); // 取消正在进行的查询
      
      const previousBalance = queryClient.getQueryData(['balance']);
      
      // 乐观更新余额
      queryClient.setQueryData(['balance'], (old) => {
        return (parseFloat(old) - parseFloat(newTx.amount)).toString();
      });
      
      return { previousBalance };
    },
    // 错误时回滚
    onError: (err, newTx, context) => {
      queryClient.setQueryData(['balance'], context.previousBalance);
    },
    // 成功或失败后重新获取最新数据
    onSettled: () => {
      queryClient.invalidateQueries(['balance']);
    }
  });
}
```

---

## Part 4: 数据缓存与持久化 (1小时)

### 4.1 缓存策略

- **Stale-While-Revalidate**: 先显示旧数据，后台更新新数据。
- **Polling**: 定期轮询更新关键数据（如区块高度、余额）。
- **Infinite Queries**: 分页加载历史记录。

### 4.2 本地持久化

```javascript
// 持久化React Query缓存到localStorage
import { persistQueryClient } from '@tanstack/react-query-persist-client';
import { createSyncStoragePersister } from '@tanstack/query-sync-storage-persister';

const persister = createSyncStoragePersister({
  storage: window.localStorage,
});

persistQueryClient({
  queryClient,
  persister,
  maxAge: 1000 * 60 * 60 * 24, // 24小时
});
```

---

## 📝 今日作业

### 作业1: 实现Zustand Store

创建一个全局Store，管理：
1. 交易列表（待确认、成功、失败）
2. 代币白名单
3. 用户设置（语言、货币单位）

### 作业2: 使用React Query重构数据获取

将之前的`useEffect`数据获取逻辑迁移到`useQuery`：
1. 获取ETH余额
2. 获取Token余额
3. 获取交易历史

### 作业3: 实现转账的乐观更新

在转账组件中：
1. 点击发送后立即在UI扣除余额
2. 添加"Pending"状态的交易记录
3. 交易确认后更新真实余额
4. 交易失败时回滚余额

---

## ✅ 检查清单

- [ ] 熟练使用Zustand管理全局状态
- [ ] 掌握React Query的基本用法
- [ ] 理解Query Key和Query Function
- [ ] 实现乐观UI更新模式
- [ ] 配置数据缓存和轮询

---

## 📅 明日预告

明天学习UI组件库集成：
- Material-UI / Chakra UI
- 定制Web3主题
- 响应式布局
- 动画效果

**🎉 完成Day 3！数据流管理大师！**
