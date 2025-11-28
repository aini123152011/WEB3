# Week 3 - Day 1: 高级Solidity特性（上）

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐ (高级)

## 📚 今日学习目标

- ✅ 掌握内联汇编(Assembly)
- ✅ 理解Yul语言
- ✅ 学习低级调用(call, delegatecall, staticcall)
- ✅ 掌握Solidity编译器优化
- ✅ 了解ABI编码

---

## 🔧 Part 1: 内联汇编基础 (2小时)

### 1.1 什么是内联汇编

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 内联汇编允许直接操作EVM
 * 优势：
 * 1. 更精细的控制
 * 2. Gas优化
 * 3. 访问Solidity无法直接访问的特性
 * 
 * 劣势：
 * 1. 降低可读性
 * 2. 增加安全风险
 * 3. 绕过类型检查
 */
contract AssemblyBasics {
    // 基础示例
    function add(uint256 a, uint256 b) public pure returns (uint256 result) {
        assembly {
            result := add(a, b)
        }
    }
    
    // 比较Solidity vs Assembly
    function addSolidity(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;
    }
    
    function addAssembly(uint256 a, uint256 b) public pure returns (uint256 result) {
        assembly {
            result := add(a, b)
        }
    }
}
```

### 1.2 Assembly变量和操作

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract AssemblyOperations {
    // 变量声明
    function variableExample() public pure returns (uint256 result) {
        assembly {
            let x := 10        // 声明变量
            let y := 20
            result := add(x, y) // 使用变量
        }
    }
    
    // 算术运算
    function arithmeticOperations(uint256 a, uint256 b) 
        public 
        pure 
        returns (
            uint256 sum,
            uint256 diff,
            uint256 product,
            uint256 quotient,
            uint256 remainder
        ) 
    {
        assembly {
            sum := add(a, b)
            diff := sub(a, b)
            product := mul(a, b)
            quotient := div(a, b)
            remainder := mod(a, b)
        }
    }
    
    // 位运算
    function bitwiseOperations(uint256 a, uint256 b) 
        public 
        pure 
        returns (
            uint256 andResult,
            uint256 orResult,
            uint256 xorResult,
            uint256 notResult,
            uint256 shiftLeft,
            uint256 shiftRight
        ) 
    {
        assembly {
            andResult := and(a, b)      // 按位与
            orResult := or(a, b)        // 按位或
            xorResult := xor(a, b)      // 按位异或
            notResult := not(a)         // 按位非
            shiftLeft := shl(2, a)      // 左移2位
            shiftRight := shr(2, a)     // 右移2位
        }
    }
    
    // 比较运算
    function comparisonOperations(uint256 a, uint256 b) 
        public 
        pure 
        returns (
            bool isEqual,
            bool isNotEqual,
            bool isLess,
            bool isGreater
        ) 
    {
        assembly {
            isEqual := eq(a, b)
            isNotEqual := iszero(eq(a, b))
            isLess := lt(a, b)
            isGreater := gt(a, b)
        }
    }
}
```

### 1.3 内存操作

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MemoryOperations {
    // 内存加载和存储
    function memoryLoadStore() public pure returns (uint256) {
        assembly {
            // 从内存位置0x80加载
            let value := mload(0x80)
            
            // 存储到内存位置0x80
            mstore(0x80, 42)
            
            // 再次加载
            value := mload(0x80)
            
            // 返回值
            mstore(0x00, value)
            return(0x00, 0x20)
        }
    }
    
    // 字节操作
    function byteOperations() public pure returns (bytes32) {
        assembly {
            // 创建32字节数据
            let data := 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
            
            // 提取第一个字节
            let firstByte := byte(0, data)
            
            // 存储结果
            mstore(0x00, firstByte)
            return(0x00, 0x20)
        }
    }
    
    // 数组操作
    function arrayOperations(uint256[] memory arr) public pure returns (uint256) {
        uint256 sum;
        assembly {
            // 数组长度存储在第一个slot
            let length := mload(arr)
            
            // 数组数据从arr + 0x20开始
            let dataPtr := add(arr, 0x20)
            
            // 遍历数组
            for { let i := 0 } lt(i, length) { i := add(i, 1) } {
                let value := mload(add(dataPtr, mul(i, 0x20)))
                sum := add(sum, value)
            }
        }
        return sum;
    }
}
```

---

## 📝 Part 2: Yul语言 (1.5小时)

### 2.1 Yul控制流

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract YulControlFlow {
    // if-else
    function absolute(int256 x) public pure returns (uint256) {
        assembly {
            // Yul使用switch而不是if-else
            switch slt(x, 0)  // 检查x < 0
            case 1 {
                // x是负数，返回-x
                mstore(0x00, sub(0, x))
            }
            default {
                // x是正数或0
                mstore(0x00, x)
            }
            return(0x00, 0x20)
        }
    }
    
    // for循环
    function sumToN(uint256 n) public pure returns (uint256) {
        assembly {
            let sum := 0
            
            // for循环：初始化; 条件; 递增
            for { let i := 1 } lt(i, add(n, 1)) { i := add(i, 1) } {
                sum := add(sum, i)
            }
            
            mstore(0x00, sum)
            return(0x00, 0x20)
        }
    }
    
    // while循环模拟
    function factorial(uint256 n) public pure returns (uint256) {
        assembly {
            let result := 1
            let counter := n
            
            // while (counter > 0)
            for {} gt(counter, 0) {} {
                result := mul(result, counter)
                counter := sub(counter, 1)
            }
            
            mstore(0x00, result)
            return(0x00, 0x20)
        }
    }
}
```

### 2.2 Yul函数

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract YulFunctions {
    // 定义Yul函数
    function power(uint256 base, uint256 exponent) public pure returns (uint256) {
        assembly {
            // 内部函数定义
            function pow(b, e) -> result {
                result := 1
                for {} gt(e, 0) {} {
                    result := mul(result, b)
                    e := sub(e, 1)
                }
            }
            
            // 调用函数
            let res := pow(base, exponent)
            mstore(0x00, res)
            return(0x00, 0x20)
        }
    }
    
    // 多返回值函数
    function divMod(uint256 a, uint256 b) 
        public 
        pure 
        returns (uint256 quotient, uint256 remainder) 
    {
        assembly {
            function divmod(dividend, divisor) -> q, r {
                q := div(dividend, divisor)
                r := mod(dividend, divisor)
            }
            
            quotient, remainder := divmod(a, b)
        }
    }
    
    // 递归函数（注意：递归在assembly中很危险）
    function fibonacci(uint256 n) public pure returns (uint256) {
        assembly {
            function fib(x) -> result {
                switch lt(x, 2)
                case 1 {
                    result := x
                }
                default {
                    let a := fib(sub(x, 1))
                    let b := fib(sub(x, 2))
                    result := add(a, b)
                }
            }
            
            let res := fib(n)
            mstore(0x00, res)
            return(0x00, 0x20)
        }
    }
}
```

---

## 🔌 Part 3: 低级调用 (1.5小时)

### 3.1 call, delegatecall, staticcall

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Target {
    uint256 public value;
    address public sender;
    
    function setValue(uint256 _value) public {
        value = _value;
        sender = msg.sender;
    }
    
    function getValue() public view returns (uint256) {
        return value;
    }
}

contract Caller {
    uint256 public value;
    address public sender;
    
    // 使用call调用
    function callSetValue(address target, uint256 _value) public returns (bool) {
        // call会在目标合约的上下文中执行
        // msg.sender是当前合约地址
        (bool success, ) = target.call(
            abi.encodeWithSignature("setValue(uint256)", _value)
        );
        return success;
    }
    
    // 使用delegatecall调用
    function delegatecallSetValue(address target, uint256 _value) public returns (bool) {
        // delegatecall在当前合约的上下文中执行
        // msg.sender保持原始调用者
        // 修改的是当前合约的storage
        (bool success, ) = target.delegatecall(
            abi.encodeWithSignature("setValue(uint256)", _value)
        );
        return success;
    }
    
    // 使用staticcall调用
    function staticcallGetValue(address target) public view returns (uint256) {
        // staticcall不能修改状态
        (bool success, bytes memory data) = target.staticcall(
            abi.encodeWithSignature("getValue()")
        );
        require(success, "Static call failed");
        return abi.decode(data, (uint256));
    }
    
    // Assembly级别的call
    function assemblyCall(address target, uint256 _value) public returns (bool success) {
        assembly {
            // 准备calldata
            let ptr := mload(0x40)
            mstore(ptr, 0x55241077) // setValue(uint256)的函数选择器
            mstore(add(ptr, 0x04), _value)
            
            // 执行call
            success := call(
                gas(),        // 转发所有gas
                target,       // 目标地址
                0,            // 不发送ETH
                ptr,          // calldata起始位置
                0x24,         // calldata长度 (4 + 32)
                0,            // 返回数据位置
                0             // 返回数据长度
            )
        }
    }
}
```

### 3.2 低级调用的安全考虑

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SafeLowLevelCalls {
    // ✅ 安全的call示例
    function safeCall(address target, bytes memory data) 
        public 
        returns (bool success, bytes memory returnData) 
    {
        // 1. 检查目标地址
        require(target != address(0), "Invalid target");
        
        // 2. 限制gas（防止gas耗尽攻击）
        (success, returnData) = target.call{gas: 10000}(data);
        
        // 3. 检查返回值
        require(success, "Call failed");
        
        return (success, returnData);
    }
    
    // ✅ 重入保护的call
    bool private locked;
    
    modifier noReentrancy() {
        require(!locked, "No reentrancy");
        locked = true;
        _;
        locked = false;
    }
    
    function protectedCall(address target, bytes memory data) 
        public 
        noReentrancy 
        returns (bool success) 
    {
        (success, ) = target.call(data);
    }
    
    // ❌ 危险的delegatecall
    function dangerousDelegatecall(address target, bytes memory data) public {
        // 危险！target可以修改任意storage
        target.delegatecall(data);
    }
    
    // ✅ 白名单delegatecall
    mapping(address => bool) public trustedTargets;
    
    function safeDelegatecall(address target, bytes memory data) public {
        require(trustedTargets[target], "Target not trusted");
        (bool success, ) = target.delegatecall(data);
        require(success, "Delegatecall failed");
    }
}
```

---

## ⚙️ Part 4: 编译器优化 (1小时)

### 4.1 Solidity优化器

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 编译器优化技巧
 * 
 * 在hardhat.config.js中配置:
 * solidity: {
 *   version: "0.8.20",
 *   settings: {
 *     optimizer: {
 *       enabled: true,
 *       runs: 200  // 优化运行次数
 *     }
 *   }
 * }
 * 
 * runs参数:
 * - 低值(1-50): 优化部署成本
 * - 高值(200+): 优化运行成本
 * - 默认: 200
 */
contract OptimizerExample {
    // 优化前：每次读取storage
    function unoptimized(uint256 n) public view returns (uint256) {
        uint256 result = 0;
        for (uint256 i = 0; i < n; i++) {
            result += i;
        }
        return result;
    }
    
    // 优化后：使用unchecked
    function optimized(uint256 n) public pure returns (uint256) {
        uint256 result = 0;
        for (uint256 i = 0; i < n;) {
            result += i;
            unchecked { i++; }
        }
        return result;
    }
}
```

### 4.2 Gas优化模式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract GasOptimizationPatterns {
    // ✅ 使用calldata代替memory
    function sumArray(uint256[] calldata arr) external pure returns (uint256) {
        uint256 sum = 0;
        uint256 length = arr.length;
        
        for (uint256 i = 0; i < length;) {
            sum += arr[i];
            unchecked { i++; }
        }
        
        return sum;
    }
    
    // ✅ 打包变量
    struct Packed {
        uint128 a;  // slot 0 (前16字节)
        uint128 b;  // slot 0 (后16字节)
        uint256 c;  // slot 1
    }
    
    // ❌ 未打包
    struct Unpacked {
        uint256 a;  // slot 0
        uint128 b;  // slot 1
        uint128 c;  // slot 2
    }
    
    // ✅ 批量操作
    function batchTransfer(
        address[] calldata recipients,
        uint256[] calldata amounts
    ) external {
        require(recipients.length == amounts.length, "Length mismatch");
        
        for (uint256 i = 0; i < recipients.length;) {
            // 转账逻辑
            unchecked { i++; }
        }
    }
    
    // ✅ 使用事件代替storage
    event DataStored(uint256 indexed id, bytes data);
    
    function storeData(uint256 id, bytes calldata data) external {
        // 不存储在storage，只发出事件
        emit DataStored(id, data);
    }
}
```

---

## 🔤 Part 5: ABI编码 (1小时)

### 5.1 ABI编码基础

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract ABIEncoding {
    // abi.encode - 标准编码
    function encodeExample(uint256 a, string memory b) 
        public 
        pure 
        returns (bytes memory) 
    {
        // 每个参数都对齐到32字节
        return abi.encode(a, b);
    }
    
    // abi.encodePacked - 紧凑编码
    function encodePackedExample(uint256 a, string memory b) 
        public 
        pure 
        returns (bytes memory) 
    {
        // 不填充，直接拼接
        return abi.encodePacked(a, b);
    }
    
    // abi.encodeWithSignature - 包含函数签名
    function encodeWithSignatureExample() public pure returns (bytes memory) {
        return abi.encodeWithSignature("transfer(address,uint256)", address(0), 100);
    }
    
    // abi.encodeWithSelector - 使用函数选择器
    function encodeWithSelectorExample() public pure returns (bytes memory) {
        bytes4 selector = bytes4(keccak256("transfer(address,uint256)"));
        return abi.encodeWithSelector(selector, address(0), 100);
    }
    
    // abi.decode - 解码
    function decodeExample(bytes memory data) 
        public 
        pure 
        returns (uint256, string memory) 
    {
        return abi.decode(data, (uint256, string));
    }
}
```

### 5.2 ABI编码实战

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract ABIEncodingAdvanced {
    // 生成函数选择器
    function getSelector(string memory functionSignature) 
        public 
        pure 
        returns (bytes4) 
    {
        return bytes4(keccak256(bytes(functionSignature)));
    }
    
    // 编码多个调用
    function encodeMulticall(
        address[] memory targets,
        bytes[] memory calldatas
    ) public pure returns (bytes memory) {
        return abi.encode(targets, calldatas);
    }
    
    // 解码并执行
    function decodeAndExecute(bytes memory data) public {
        (address[] memory targets, bytes[] memory calldatas) = 
            abi.decode(data, (address[], bytes[]));
        
        require(targets.length == calldatas.length, "Length mismatch");
        
        for (uint256 i = 0; i < targets.length; i++) {
            (bool success, ) = targets[i].call(calldatas[i]);
            require(success, "Call failed");
        }
    }
    
    // Packed encoding碰撞示例
    function packingCollision() public pure returns (bool) {
        // 这两个会产生相同的packed encoding!
        bytes memory a = abi.encodePacked("aaa", "bbb");
        bytes memory b = abi.encodePacked("aa", "abbb");
        
        return keccak256(a) == keccak256(b);  // true!
    }
    
    // 使用abi.encode避免碰撞
    function safeEncoding() public pure returns (bool) {
        bytes memory a = abi.encode("aaa", "bbb");
        bytes memory b = abi.encode("aa", "abbb");
        
        return keccak256(a) == keccak256(b);  // false
    }
}
```

---

## 📝 今日作业

### 作业1: Assembly实现ERC20

使用内联汇编实现一个简化的ERC20代币：
1. 使用assembly实现transfer
2. 使用assembly实现balanceOf
3. 优化gas消耗
4. 添加安全检查

### 作业2: 低级调用路由器

实现一个调用路由器合约：
1. 支持批量call
2. 支持批量delegatecall
3. 添加权限控制
4. 实现回滚机制

### 作业3: ABI编码工具

实现一个ABI编码工具合约：
1. 函数选择器生成
2. 任意函数调用编码
3. 返回值解码
4. 批量调用编码

---

## ✅ 今日检查清单

- [ ] 理解内联汇编语法
- [ ] 掌握Yul语言基础
- [ ] 理解低级调用区别
- [ ] 了解编译器优化
- [ ] 掌握ABI编码方法
- [ ] 完成所有作业

---

## 🆘 常见问题FAQ

### Q1: 何时使用内联汇编？

A: 
- Gas优化关键路径
- 实现Solidity不支持的功能
- 与其他合约交互的特殊需求

注意：只在必要时使用，优先考虑可读性和安全性。

### Q2: call和delegatecall的区别？

A:
```
call:
- 在目标合约上下文中执行
- msg.sender是调用者
- 修改目标合约的storage

delegatecall:
- 在当前合约上下文中执行
- msg.sender是原始调用者
- 修改当前合约的storage
```

### Q3: abi.encode和abi.encodePacked的区别？

A:
- `abi.encode`: 标准ABI编码，每个参数对齐到32字节
- `abi.encodePacked`: 紧凑编码，不填充，可能碰撞

---

## 📅 明日预告

明天学习更多高级特性：
- 库合约深入
- 接口和抽象合约
- 多重继承详解
- 元编程技巧

**🎉 完成Day 1！继续加油！**
