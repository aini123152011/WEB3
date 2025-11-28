# Week 7 - Day 1: React基础与Hooks

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 掌握React高级特性
- ✅ 深入理解Hooks原理
- ✅ 学习自定义Hooks封装
- ✅ 掌握性能优化技巧
- ✅ 理解React 18新特性

---

## Part 1: React高级特性 (1.5小时)

### 1.1 深入JSX与组件

```javascript
// 高阶组件 (HOC)
const withUser = (WrappedComponent) => {
  return (props) => {
    const user = useUser(); // 假设有一个useUser Hook
    return <WrappedComponent {...props} user={user} />;
  };
};

// Render Props
const MouseTracker = ({ render }) => {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMouseMove = (event) => {
      setPosition({ x: event.clientX, y: event.clientY });
    };
    window.addEventListener('mousemove', handleMouseMove);
    return () => window.removeEventListener('mousemove', handleMouseMove);
  }, []);
  
  return render(position);
};

// 使用示例
const App = () => (
  <MouseTracker render={({ x, y }) => (
    <h1>The mouse position is ({x}, {y})</h1>
  )}/>
);
```

### 1.2 Portal与Refs

```javascript
import { createPortal } from 'react-dom';
import { useRef, useEffect } from 'react';

// Modal组件使用Portal
const Modal = ({ isOpen, children, onClose }) => {
  if (!isOpen) return null;
  
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={e => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.body
  );
};

// Refs访问DOM
const AutoFocusInput = () => {
  const inputRef = useRef(null);
  
  useEffect(() => {
    if (inputRef.current) {
      inputRef.current.focus();
    }
  }, []);
  
  return <input ref={inputRef} />;
};
```

---

## Part 2: Hooks深入 (2小时)

### 2.1 useReducer

用于复杂状态管理。

```javascript
const initialState = { count: 0, loading: false };

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + 1 };
    case 'decrement':
      return { ...state, count: state.count - 1 };
    case 'setLoading':
      return { ...state, loading: action.payload };
    default:
      throw new Error();
  }
}

const Counter = () => {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  return (
    <>
      Count: {state.count}
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
    </>
  );
};
```

### 2.2 useContext + useReducer

替代Redux的轻量级方案。

```javascript
const StoreContext = createContext();

export const StoreProvider = ({ children }) => {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  return (
    <StoreContext.Provider value={{ state, dispatch }}>
      {children}
    </StoreContext.Provider>
  );
};

export const useStore = () => useContext(StoreContext);
```

### 2.3 useLayoutEffect

与useEffect的区别：在DOM变更之后同步触发。

```javascript
const Tooltip = ({ children, text }) => {
  const [coords, setCoords] = useState({ x: 0, y: 0 });
  const elRef = useRef(null);
  
  useLayoutEffect(() => {
    if (elRef.current) {
      const rect = elRef.current.getBoundingClientRect();
      setCoords({
        x: rect.left,
        y: rect.top - 30 // 显示在上方
      });
    }
  }, []);
  
  return (
    <>
      <span ref={elRef}>{children}</span>
      {createPortal(
        <div style={{ position: 'absolute', left: coords.x, top: coords.y }}>
          {text}
        </div>,
        document.body
      )}
    </>
  );
};
```

---

## Part 3: Web3自定义Hooks (2小时)

### 3.1 usePoller (轮询Hook)

```javascript
import { useEffect, useRef } from 'react';

export const usePoller = (fn, delay) => {
  const savedFn = useRef();
  
  // 保存最新的回调
  useEffect(() => {
    savedFn.current = fn;
  }, [fn]);
  
  // 设置轮询
  useEffect(() => {
    if (delay !== null) {
      const tick = () => {
        if (savedFn.current) savedFn.current();
      }
      
      const id = setInterval(tick, delay);
      return () => clearInterval(id);
    }
  }, [delay]);
  
  // 立即执行一次
  useEffect(() => {
    if (delay !== null && savedFn.current) {
      savedFn.current();
    }
  }, []);
};

// 使用示例
const BlockNumber = () => {
  const [blockNumber, setBlockNumber] = useState(0);
  const provider = useProvider();
  
  usePoller(async () => {
    if (provider) {
      const bn = await provider.getBlockNumber();
      setBlockNumber(bn);
    }
  }, 5000); // 每5秒轮询一次
  
  return <div>Block: {blockNumber}</div>;
};
```

### 3.2 useEventListener (事件监听)

```javascript
export const useEventListener = (contract, eventName, provider, startBlock) => {
  const [events, setEvents] = useState([]);
  
  useEffect(() => {
    if (!contract || !provider) return;
    
    // 监听新事件
    const listener = (...args) => {
      const event = args[args.length - 1]; // 最后一个参数是事件对象
      setEvents(prev => [...prev, event]);
    };
    
    contract.on(eventName, listener);
    
    // 获取历史事件
    const fetchPastEvents = async () => {
      const pastEvents = await contract.queryFilter(eventName, startBlock);
      setEvents(pastEvents);
    };
    
    if (startBlock) {
      fetchPastEvents();
    }
    
    return () => {
      contract.off(eventName, listener);
    };
  }, [contract, eventName, provider, startBlock]);
  
  return events;
};
```

### 3.3 useGasPrice (Gas价格)

```javascript
export const useGasPrice = (provider, speed = 'fast') => {
  const [gasPrice, setGasPrice] = useState(null);
  
  usePoller(async () => {
    if (!provider) return;
    
    try {
      const feeData = await provider.getFeeData();
      // 简单的Gas Price
      if (feeData.gasPrice) {
        setGasPrice(feeData.gasPrice);
      }
      // EIP-1559
      if (feeData.maxFeePerGas) {
        // 这里可以根据speed调整倍率
        setGasPrice(feeData.maxFeePerGas);
      }
    } catch (e) {
      console.log(e);
    }
  }, 10000);
  
  return gasPrice;
};
```

---

## Part 4: 性能优化 (1.5小时)

### 4.1 React.memo

```javascript
// 只有当props变化时才重新渲染
const TokenRow = React.memo(({ symbol, balance }) => {
  console.log(`Rendering ${symbol}`);
  return (
    <div className="token-row">
      <span>{symbol}</span>
      <span>{balance}</span>
    </div>
  );
});

// 自定义比较函数
const areEqual = (prevProps, nextProps) => {
  return prevProps.balance === nextProps.balance; // 只关心余额变化
};

const OptimizedTokenRow = React.memo(TokenRow, areEqual);
```

### 4.2 useMemo & useCallback 最佳实践

```javascript
const SwapInterface = ({ tokens }) => {
  const [amount, setAmount] = useState(0);
  const [selectedToken, setToken] = useState(tokens[0]);
  
  // 昂贵的计算
  const derivedAmount = useMemo(() => {
    return complexCalculation(amount, selectedToken.price);
  }, [amount, selectedToken.price]);
  
  // 传递给子组件的回调
  const handleApprove = useCallback(async () => {
    await approveToken(selectedToken.address, amount);
  }, [selectedToken.address, amount]);
  
  return (
    <div>
      <Input value={amount} onChange={setAmount} />
      <Output value={derivedAmount} />
      <Button onClick={handleApprove}>Approve</Button>
    </div>
  );
};
```

### 4.3 虚拟列表 (Virtualization)

用于处理大量数据的列表（如交易记录）。

```javascript
import { FixedSizeList as List } from 'react-window';

const TransactionList = ({ transactions }) => {
  const Row = ({ index, style }) => (
    <div style={style} className="tx-row">
      Transaction {transactions[index].hash}
    </div>
  );
  
  return (
    <List
      height={400}
      itemCount={transactions.length}
      itemSize={50}
      width={'100%'}
    >
      {Row}
    </List>
  );
};
```

---

## Part 5: React 18新特性 (1小时)

### 5.1 Automatic Batching

React 18自动批量处理状态更新，减少渲染次数。

```javascript
// React 17: 会触发两次渲染
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
}, 1000);

// React 18: 只触发一次渲染 (自动批处理)
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
}, 1000);
```

### 5.2 Transitions

标记非紧急更新。

```javascript
import { useTransition } from 'react';

const Search = () => {
  const [isPending, startTransition] = useTransition();
  const [input, setInput] = useState('');
  const [list, setList] = useState([]);
  
  const handleChange = (e) => {
    // 紧急更新：输入框回显
    setInput(e.target.value);
    
    // 非紧急更新：过滤列表
    startTransition(() => {
      const filtered = filterLargeList(e.target.value);
      setList(filtered);
    });
  };
  
  return (
    <div>
      <input value={input} onChange={handleChange} />
      {isPending ? 'Loading...' : <List items={list} />}
    </div>
  );
};
```

---

## 📝 今日作业

### 作业1: 实现TokenList组件

要求：
1. 使用useMemo优化排序和过滤
2. 使用React.memo优化行组件
3. 使用虚拟列表处理大量Token
4. 集成usePoller更新余额

### 作业2: 封装useContract Hook

封装一个通用的合约交互Hook：
1. 自动处理Provider/Signer
2. 处理合约加载状态
3. 提供只读和写入方法
4. 集成错误处理

### 作业3: 优化DApp性能

对之前的Dashboard进行优化：
1. 减少不必要的重渲染
2. 使用Suspense和Lazy加载组件
3. 优化Context Context Value
4. 使用Transition处理复杂状态更新

---

## ✅ 检查清单

- [ ] 掌握HOC和Render Props
- [ ] 熟练使用useReducer
- [ ] 能编写自定义Web3 Hooks
- [ ] 理解React性能优化
- [ ] 了解React 18特性

---

## 📅 明日预告

明天深入Web3React库：
- web3-react v6/v8
- 连接器配置
- 多链状态管理
- 错误处理策略

**🎉 完成Week 7 Day 1！React功底更上一层楼！**
