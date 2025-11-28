# Week 3 - Day 2: 高级Solidity特性（下）

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐ (高级)

## 📚 今日学习目标

- ✅ 深入理解库合约(Library)
- ✅ 掌握接口(Interface)和抽象合约(Abstract)
- ✅ 学习多重继承
- ✅ 掌握函数选择器和签名
- ✅ 了解元编程技巧

---

## 📚 Part 1: 库合约深入 (2小时)

### 1.1 库合约基础

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 库合约(Library)
 * - 只部署一次
 * - 代码可被多个合约复用
 * - 节省gas
 * - 不能有状态变量
 * - 不能继承或被继承
 * - 不能接收ETH
 */
library SafeMath {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "SafeMath: addition overflow");
        return c;
    }
    
    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b <= a, "SafeMath: subtraction overflow");
        return a - b;
    }
    
    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) return 0;
        uint256 c = a * b;
        require(c / a == b, "SafeMath: multiplication overflow");
        return c;
    }
    
    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b > 0, "SafeMath: division by zero");
        return a / b;
    }
}

contract UsingLibrary {
    using SafeMath for uint256;
    
    function calculate(uint256 a, uint256 b) public pure returns (uint256) {
        // 使用库函数
        return a.add(b).mul(2);
    }
}
```

### 1.2 高级库模式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 结构体库
 */
library StructLib {
    struct Data {
        uint256 value;
        mapping(address => uint256) balances;
    }
    
    // 修改结构体的函数
    function setValue(Data storage self, uint256 _value) internal {
        self.value = _value;
    }
    
    function addBalance(Data storage self, address account, uint256 amount) internal {
        self.balances[account] += amount;
    }
    
    function getBalance(Data storage self, address account) internal view returns (uint256) {
        return self.balances[account];
    }
}

contract UsingStructLib {
    using StructLib for StructLib.Data;
    
    StructLib.Data private data;
    
    function set(uint256 value) public {
        data.setValue(value);
    }
    
    function deposit() public payable {
        data.addBalance(msg.sender, msg.value);
    }
    
    function getMyBalance() public view returns (uint256) {
        return data.getBalance(msg.sender);
    }
}
```

### 1.3 可链接库

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 外部库（需要单独部署和链接）
 */
library MathLib {
    // public函数可以被外部调用
    function power(uint256 base, uint256 exponent) public pure returns (uint256) {
        uint256 result = 1;
        for (uint256 i = 0; i < exponent; i++) {
            result *= base;
        }
        return result;
    }
    
    function sqrt(uint256 x) public pure returns (uint256) {
        if (x == 0) return 0;
        uint256 z = (x + 1) / 2;
        uint256 y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
        return y;
    }
}

contract UsingMathLib {
    function calculate(uint256 a, uint256 b) public pure returns (uint256, uint256) {
        return (MathLib.power(a, b), MathLib.sqrt(a));
    }
}
```

---

## 🔌 Part 2: 接口和抽象合约 (1.5小时)

### 2.1 接口(Interface)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 接口规则：
 * 1. 不能有状态变量
 * 2. 不能有构造函数
 * 3. 不能继承合约（可以继承接口）
 * 4. 所有函数必须是external
 * 5. 不能定义函数实现
 */
interface IERC20 {
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}

// 接口继承
interface IERC20Metadata is IERC20 {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function decimals() external view returns (uint8);
}

// 实现接口
contract MyToken is IERC20Metadata {
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    uint256 private _totalSupply;
    
    string private _name;
    string private _symbol;
    
    constructor(string memory name_, string memory symbol_) {
        _name = name_;
        _symbol = symbol_;
    }
    
    // 实现所有接口函数...
    function name() public view override returns (string memory) {
        return _name;
    }
    
    function symbol() public view override returns (string memory) {
        return _symbol;
    }
    
    function decimals() public pure override returns (uint8) {
        return 18;
    }
    
    function totalSupply() public view override returns (uint256) {
        return _totalSupply;
    }
    
    function balanceOf(address account) public view override returns (uint256) {
        return _balances[account];
    }
    
    function transfer(address to, uint256 amount) public override returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }
    
    function allowance(address owner, address spender) public view override returns (uint256) {
        return _allowances[owner][spender];
    }
    
    function approve(address spender, uint256 amount) public override returns (bool) {
        _approve(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) public override returns (bool) {
        _spendAllowance(from, msg.sender, amount);
        _transfer(from, to, amount);
        return true;
    }
    
    function _transfer(address from, address to, uint256 amount) internal {
        require(from != address(0), "Transfer from zero");
        require(to != address(0), "Transfer to zero");
        
        _balances[from] -= amount;
        _balances[to] += amount;
        
        emit Transfer(from, to, amount);
    }
    
    function _approve(address owner, address spender, uint256 amount) internal {
        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }
    
    function _spendAllowance(address owner, address spender, uint256 amount) internal {
        uint256 currentAllowance = _allowances[owner][spender];
        require(currentAllowance >= amount, "Insufficient allowance");
        _approve(owner, spender, currentAllowance - amount);
    }
}
```

### 2.2 抽象合约(Abstract Contract)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 抽象合约：
 * 1. 至少有一个未实现的函数
 * 2. 不能直接部署
 * 3. 必须被继承实现
 */
abstract contract Token {
    // 状态变量
    mapping(address => uint256) public balances;
    
    // 已实现的函数
    function balanceOf(address account) public view returns (uint256) {
        return balances[account];
    }
    
    // 未实现的函数（必须由子合约实现）
    function transfer(address to, uint256 amount) public virtual returns (bool);
    
    // 可选实现的函数
    function name() public view virtual returns (string memory) {
        return "Token";
    }
}

// 继承并实现抽象合约
contract ConcreteToken is Token {
    function transfer(address to, uint256 amount) public override returns (bool) {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] -= amount;
        balances[to] += amount;
        return true;
    }
    
    // 可以重写可选函数
    function name() public pure override returns (string memory) {
        return "ConcreteToken";
    }
}
```

---

## 🌳 Part 3: 多重继承 (1.5小时)

### 3.1 继承基础

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Base1 {
    uint256 public value1;
    
    function setValue1(uint256 _value) public virtual {
        value1 = _value;
    }
    
    function getValue() public view virtual returns (uint256) {
        return value1;
    }
}

contract Base2 {
    uint256 public value2;
    
    function setValue2(uint256 _value) public virtual {
        value2 = _value;
    }
    
    function getValue() public view virtual returns (uint256) {
        return value2;
    }
}

// 多重继承
contract Derived is Base1, Base2 {
    // 必须重写有冲突的函数
    function getValue() public view override(Base1, Base2) returns (uint256) {
        return value1 + value2;
    }
}
```

### 3.2 钻石继承(C3线性化)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 钻石继承问题
 * 
 *      A
 *     / \
 *    B   C
 *     \ /
 *      D
 * 
 * Solidity使用C3线性化解决
 * D的继承顺序：D, C, B, A
 */
contract A {
    event Log(string message);
    
    function foo() public virtual {
        emit Log("A.foo");
    }
}

contract B is A {
    function foo() public virtual override {
        emit Log("B.foo");
        super.foo();
    }
}

contract C is A {
    function foo() public virtual override {
        emit Log("C.foo");
        super.foo();
    }
}

// 继承顺序很重要！
contract D is B, C {
    function foo() public override(B, C) {
        super.foo();  // 调用顺序：D -> C -> B -> A
    }
}
```

### 3.3 构造函数继承

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Base {
    uint256 public x;
    
    constructor(uint256 _x) {
        x = _x;
    }
}

contract Derived1 is Base {
    // 方式1：在继承列表中传参
    constructor() Base(10) {}
}

contract Derived2 is Base {
    // 方式2：在构造函数中传参
    constructor(uint256 _x) Base(_x) {}
}

// 多重继承的构造函数
contract Base1 {
    uint256 public a;
    constructor(uint256 _a) {
        a = _a;
    }
}

contract Base2 {
    uint256 public b;
    constructor(uint256 _b) {
        b = _b;
    }
}

contract MultiDerived is Base1, Base2 {
    // 构造函数按照继承顺序调用：Base1 -> Base2 -> MultiDerived
    constructor(uint256 _a, uint256 _b) Base1(_a) Base2(_b) {}
}
```

---

## 🔍 Part 4: 函数选择器 (1小时)

### 4.1 函数签名和选择器

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract FunctionSelector {
    // 函数选择器是函数签名的keccak256哈希的前4字节
    
    // 获取函数选择器
    function getSelector(string memory functionSignature) 
        public 
        pure 
        returns (bytes4) 
    {
        return bytes4(keccak256(bytes(functionSignature)));
    }
    
    // 示例函数
    function transfer(address to, uint256 amount) public pure returns (bool) {
        return true;
    }
    
    // 获取transfer的选择器
    function getTransferSelector() public pure returns (bytes4) {
        // 两种方式获取：
        
        // 方式1：使用字符串
        bytes4 selector1 = bytes4(keccak256("transfer(address,uint256)"));
        
        // 方式2：使用this
        bytes4 selector2 = this.transfer.selector;
        
        return selector1; // 结果相同
    }
    
    // 动态调用
    function callTransfer(address target, address to, uint256 amount) 
        public 
        returns (bool) 
    {
        bytes memory data = abi.encodeWithSelector(
            bytes4(keccak256("transfer(address,uint256)")),
            to,
            amount
        );
        
        (bool success, bytes memory returnData) = target.call(data);
        require(success, "Call failed");
        
        return abi.decode(returnData, (bool));
    }
}
```

### 4.2 fallback和receive

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract FallbackExample {
    event Log(string func, address sender, uint256 value, bytes data);
    
    // receive: 接收ETH且calldata为空时调用
    receive() external payable {
        emit Log("receive", msg.sender, msg.value, "");
    }
    
    // fallback: 
    // 1. 调用不存在的函数时
    // 2. 接收ETH且calldata不为空时
    fallback() external payable {
        emit Log("fallback", msg.sender, msg.value, msg.data);
    }
    
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}

contract Caller {
    function testFallback(address payable target) public payable {
        // 调用不存在的函数 -> fallback
        (bool success, ) = target.call{value: msg.value}(
            abi.encodeWithSignature("nonExistentFunction()")
        );
        require(success, "Call failed");
    }
    
    function testReceive(address payable target) public payable {
        // 只发送ETH -> receive
        (bool success, ) = target.call{value: msg.value}("");
        require(success, "Call failed");
    }
}
```

---

## 🎨 Part 5: 元编程技巧 (1小时)

### 5.1 类型信息

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract TypeInformation {
    // type()获取类型信息
    function getTypeInfo() public pure returns (
        uint256 min,
        uint256 max,
        string memory name
    ) {
        min = type(uint256).min;
        max = type(uint256).max;
        name = "uint256";
    }
    
    // 合约类型信息
    function getContractInfo() public view returns (
        bytes memory creationCode,
        bytes memory runtimeCode,
        string memory name
    ) {
        creationCode = type(TypeInformation).creationCode;
        runtimeCode = type(TypeInformation).runtimeCode;
        name = type(TypeInformation).name;
    }
    
    // 接口ID
    function getInterfaceId() public pure returns (bytes4) {
        return type(IERC20).interfaceId;
    }
}

interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
}
```

### 5.2 内省(Introspection)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ERC165 - 标准接口检测
 */
interface IERC165 {
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
}

contract ERC165 is IERC165 {
    mapping(bytes4 => bool) private _supportedInterfaces;
    
    constructor() {
        // 注册ERC165接口
        _registerInterface(type(IERC165).interfaceId);
    }
    
    function supportsInterface(bytes4 interfaceId) 
        public 
        view 
        virtual 
        override 
        returns (bool) 
    {
        return _supportedInterfaces[interfaceId];
    }
    
    function _registerInterface(bytes4 interfaceId) internal {
        require(interfaceId != 0xffffffff, "Invalid interface id");
        _supportedInterfaces[interfaceId] = true;
    }
}

// 使用示例
contract MyContract is ERC165, IERC20 {
    constructor() {
        _registerInterface(type(IERC20).interfaceId);
    }
    
    // 实现IERC20...
    function totalSupply() external pure returns (uint256) {
        return 0;
    }
    
    function balanceOf(address) external pure returns (uint256) {
        return 0;
    }
}
```

### 5.3 动态合约创建

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Clone {
    uint256 public value;
    
    constructor(uint256 _value) {
        value = _value;
    }
}

contract Factory {
    Clone[] public clones;
    
    // 使用new创建合约
    function createClone(uint256 _value) public returns (address) {
        Clone clone = new Clone(_value);
        clones.push(clone);
        return address(clone);
    }
    
    // 使用CREATE2创建（地址可预测）
    function createClone2(uint256 _value, bytes32 salt) 
        public 
        returns (address) 
    {
        Clone clone = new Clone{salt: salt}(_value);
        clones.push(clone);
        return address(clone);
    }
    
    // 预测CREATE2地址
    function predictAddress(bytes32 salt, uint256 _value) 
        public 
        view 
        returns (address) 
    {
        bytes memory bytecode = abi.encodePacked(
            type(Clone).creationCode,
            abi.encode(_value)
        );
        
        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(this),
                salt,
                keccak256(bytecode)
            )
        );
        
        return address(uint160(uint256(hash)));
    }
}
```

---

## 📝 今日作业

### 作业1: 可升级代理库

实现一个代理库合约：
1. 使用library实现代理逻辑
2. 支持UUPS升级模式
3. 添加初始化逻辑
4. 实现存储槽管理

### 作业2: 插件系统

使用接口和继承实现插件系统：
1. 定义插件接口
2. 实现插件管理器
3. 支持动态加载/卸载
4. 插件间通信

### 作业3: 合约工厂

实现高级合约工厂：
1. CREATE和CREATE2部署
2. 克隆模式优化
3. 地址预测
4. 批量部署

---

## ✅ 今日检查清单

- [ ] 掌握库合约使用
- [ ] 理解接口和抽象合约
- [ ] 掌握多重继承规则
- [ ] 理解函数选择器
- [ ] 了解元编程技巧
- [ ] 完成所有作业

---

## 🆘 常见问题FAQ

### Q1: 什么时候用library，什么时候用contract?

A:
- Library: 纯逻辑复用，无状态，节省gas
- Contract: 需要状态存储，可独立部署

### Q2: Interface vs Abstract Contract?

A:
- Interface: 纯接口定义，无实现
- Abstract: 可以有部分实现

### Q3: 多重继承的顺序重要吗？

A: 非常重要！按照C3线性化规则，从最基础到最派生排列。

---

## 📅 明日预告

明天学习智能合约安全：
- 常见漏洞类型
- 攻击向量分析
- 防御技术
- 安全审计方法

**🎉 完成Day 2！继续加油！**
