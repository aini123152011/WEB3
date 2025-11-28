# Week 7 - Day 4: UI组件库集成

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 掌握Chakra UI/Material-UI集成
- ✅ 定制Web3风格主题
- ✅ 实现响应式布局
- ✅ 学习Framer Motion动画
- ✅ 构建通用Web3组件

---

## Part 1: UI组件库选择与集成 (1.5小时)

### 1.1 为什么选择Chakra UI

对于Web3项目，推荐使用 **Chakra UI**，原因：
- 优秀的TypeScript支持
- 灵活的主题定制
- 内置暗色模式
- 易于组合的组件API

### 1.2 安装与配置

```bash
npm i @chakra-ui/react @emotion/react @emotion/styled framer-motion
```

```javascript
// App.js
import { ChakraProvider } from '@chakra-ui/react';
import theme from './theme';

function App({ children }) {
  return (
    <ChakraProvider theme={theme}>
      {children}
    </ChakraProvider>
  );
}
```

---

## Part 2: 定制Web3主题 (1.5小时)

### 2.1 颜色与字体

```javascript
// theme/index.js
import { extendTheme } from '@chakra-ui/react';

const colors = {
  brand: {
    50: '#e3f2fd',
    100: '#bbdefb',
    500: '#2196f3', // 主色调
    900: '#0d47a1',
  },
  accent: {
    500: '#ff4081', // 强调色
  },
  bg: {
    light: '#ffffff',
    dark: '#0a0b0d', // 深色背景
  }
};

const config = {
  initialColorMode: 'dark',
  useSystemColorMode: false,
};

const styles = {
  global: (props) => ({
    body: {
      bg: props.colorMode === 'dark' ? 'bg.dark' : 'bg.light',
      color: props.colorMode === 'dark' ? 'white' : 'gray.800',
    },
  }),
};

export default extendTheme({ colors, config, styles });
```

### 2.2 组件样式覆盖

```javascript
// theme/components/Button.js
export const Button = {
  baseStyle: {
    fontWeight: 'bold',
    borderRadius: 'xl', // 圆角风格
  },
  variants: {
    solid: (props) => ({
      bg: props.colorMode === 'dark' ? 'brand.500' : 'brand.500',
      color: 'white',
      _hover: {
        bg: 'brand.600',
        transform: 'translateY(-2px)',
        boxShadow: 'lg',
      },
    }),
    outline: {
      border: '2px solid',
      borderColor: 'brand.500',
      color: 'brand.500',
    },
  },
};
```

---

## Part 3: 通用Web3组件构建 (2小时)

### 3.1 Address Display (地址显示)

```javascript
import { Text, Tooltip, useClipboard, IconButton } from '@chakra-ui/react';
import { CopyIcon, CheckIcon } from '@chakra-ui/icons';

export const AddressDisplay = ({ address, shorten = true }) => {
  const { hasCopied, onCopy } = useClipboard(address);
  
  const displayAddress = shorten 
    ? `${address.slice(0, 6)}...${address.slice(-4)}`
    : address;

  return (
    <Tooltip label={hasCopied ? 'Copied!' : 'Copy Address'} closeOnClick={false}>
      <div style={{ display: 'flex', alignItems: 'center', gap: '8px' }}>
        <Text fontFamily="mono" fontWeight="medium">
          {displayAddress}
        </Text>
        <IconButton
          size="xs"
          icon={hasCopied ? <CheckIcon /> : <CopyIcon />}
          onClick={onCopy}
          variant="ghost"
          aria-label="Copy address"
        />
      </div>
    </Tooltip>
  );
};
```

### 3.2 Token Input (代币输入框)

```javascript
import { 
  InputGroup, 
  Input, 
  InputRightElement, 
  Button, 
  Image, 
  Stack 
} from '@chakra-ui/react';

export const TokenInput = ({ value, onChange, token, balance, onMax }) => {
  return (
    <Stack spacing={2}>
      <div style={{ display: 'flex', justifyContent: 'space-between', fontSize: '0.875rem' }}>
        <span>Input</span>
        <span>Balance: {balance}</span>
      </div>
      
      <InputGroup size="lg">
        <Input
          type="number"
          placeholder="0.0"
          value={value}
          onChange={(e) => onChange(e.target.value)}
          borderRadius="xl"
        />
        <InputRightElement width="auto" pr={2}>
          <Button size="sm" variant="ghost" onClick={onMax} mr={2}>
            MAX
          </Button>
          {token && (
            <div style={{ display: 'flex', alignItems: 'center', gap: '4px' }}>
              <Image src={token.logoURI} boxSize="24px" />
              <span>{token.symbol}</span>
            </div>
          )}
        </InputRightElement>
      </InputGroup>
    </Stack>
  );
};
```

---

## Part 4: 响应式布局与适配 (1小时)

### 4.1 响应式Grid布局

```javascript
import { Grid, GridItem, Box } from '@chakra-ui/react';

const DashboardLayout = ({ children }) => {
  return (
    <Grid
      templateAreas={{
        base: `"header" "main" "footer"`,
        lg: `"header header" "sidebar main" "footer footer"`
      }}
      gridTemplateRows={'auto 1fr auto'}
      gridTemplateColumns={{ base: '1fr', lg: '250px 1fr' }}
      h="100vh"
      gap={4}
    >
      <GridItem area={'header'}><Header /></GridItem>
      
      {/* 侧边栏只在桌面端显示 */}
      <GridItem area={'sidebar'} display={{ base: 'none', lg: 'block' }}>
        <Sidebar />
      </GridItem>
      
      <GridItem area={'main'} p={4}>
        {children}
      </GridItem>
      
      <GridItem area={'footer'}><Footer /></GridItem>
    </Grid>
  );
};
```

---

## Part 5: 动画效果 (Framer Motion) (1小时)

### 5.1 模态框动画

```javascript
import { motion, AnimatePresence } from 'framer-motion';

const ModalOverlay = ({ isOpen, onClose, children }) => {
  return (
    <AnimatePresence>
      {isOpen && (
        <>
          <motion.div
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
            style={{ position: 'fixed', inset: 0, background: 'rgba(0,0,0,0.5)' }}
          />
          <motion.div
            initial={{ y: 50, opacity: 0 }}
            animate={{ y: 0, opacity: 1 }}
            exit={{ y: 50, opacity: 0 }}
            style={{ 
              position: 'fixed', 
              top: '50%', 
              left: '50%', 
              transform: 'translate(-50%, -50%)',
              background: 'white',
              padding: '20px',
              borderRadius: '12px'
            }}
          >
            {children}
          </motion.div>
        </>
      )}
    </AnimatePresence>
  );
};
```

### 5.2 列表项动画

```javascript
const TransactionList = ({ transactions }) => {
  return (
    <Stack spacing={2}>
      <AnimatePresence>
        {transactions.map((tx) => (
          <motion.div
            key={tx.hash}
            initial={{ opacity: 0, x: -20 }}
            animate={{ opacity: 1, x: 0 }}
            exit={{ opacity: 0, x: 20 }}
            transition={{ duration: 0.2 }}
          >
            <TransactionItem tx={tx} />
          </motion.div>
        ))}
      </AnimatePresence>
    </Stack>
  );
};
```

---

## 📝 今日作业

### 作业1: 构建Web3组件库

创建一个`components/web3`目录，实现以下组件：
1. `ConnectWalletButton` (带状态指示灯)
2. `NetworkSelector` (带图标和下拉菜单)
3. `TokenSelectorModal` (带搜索功能)
4. `TransactionStatus` (带进度条动画)

### 作业2: 实现暗色模式切换

1. 使用Chakra UI的`useColorMode` Hook
2. 创建一个切换按钮，带旋转动画
3. 确保所有自定义组件都适配暗色模式

### 作业3: 响应式Dashboard布局

1. 桌面端：侧边栏导航
2. 移动端：底部导航栏 (Bottom Tab Bar)
3. 内容区域自适应网格布局

---

## ✅ 检查清单

- [ ] 成功集成Chakra UI
- [ ] 配置好自定义Theme
- [ ] 实现响应式布局
- [ ] 添加Framer Motion动画
- [ ] 构建至少3个通用Web3组件

---

## 📅 明日预告

明天学习实时数据更新：
- WebSocket订阅
- 区块监听
- 价格流更新
- 通知系统实战

**🎉 完成Day 4！界面越来越专业了！**
