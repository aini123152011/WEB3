# Week 2 - Day 3: Solidity进阶特性（上）

**学习日期**: ___________
**预计用时**: 5-6小时  
**难度等级**: ⭐⭐⭐⭐ (进阶)

## 📚 今日学习目标

- ✅ 掌握合约继承机制
- ✅ 理解接口(Interface)
- ✅ 掌握抽象合约(Abstract Contract)
- ✅ 学习库(Library)的使用
- ✅ 理解修饰器(Modifier)高级用法
- ✅ 掌握事件(Event)和日志

---

## 🔗 Part 1: 合约继承 (1.5小时)

### 1.1 基础继承

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// 父合约
contract Animal {
    string public name;
    uint256 public age;
    
    event AnimalCreated(string name, uint256 age);
    
    constructor(string memory _name, uint256 _age) {
        name = _name;
        age = _age;
        emit AnimalCreated(_name, _age);
    }
    
    function eat() public virtual returns (string memory) {
        return "Animal is eating";
    }
    
    function sleep() public pure returns (string memory) {
        return "Animal is sleeping";
    }
}

// 子合约继承父合约
contract Dog is Animal {
    string public breed;
    
    constructor(
        string memory _name,
        uint256 _age,
        string memory _breed
    ) Animal(_name, _age) {
        breed = _breed;
    }
    
    // 重写父合约函数
    function eat() public pure override returns (string memory) {
        return "Dog is eating bones";
    }
    
    // 新增函数
    function bark() public pure returns (string memory) {
        return "Woof! Woof!";
    }
}

contract Cat is Animal {
    bool public isIndoor;
    
    constructor(
        string memory _name,
        uint256 _age,
        bool _isIndoor
    ) Animal(_name, _age) {
        isIndoor = _isIndoor;
    }
    
    function eat() public pure override returns (string memory) {
        return "Cat is eating fish";
    }
    
    function meow() public pure returns (string memory) {
        return "Meow!";
    }
}
```

### 1.2 多重继承

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Ownable {
    address public owner;
    
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    
    constructor() {
        owner = msg.sender;
        emit OwnershipTransferred(address(0), msg.sender);
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not the owner");
        _;
    }
    
    function transferOwnership(address newOwner) public virtual onlyOwner {
        require(newOwner != address(0), "Invalid address");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }
}

contract Pausable {
    bool public paused;
    
    event Paused(address account);
    event Unpaused(address account);
    
    modifier whenNotPaused() {
        require(!paused, "Contract is paused");
        _;
    }
    
    modifier whenPaused() {
        require(paused, "Contract is not paused");
        _;
    }
    
    function _pause() internal virtual whenNotPaused {
        paused = true;
        emit Paused(msg.sender);
    }
    
    function _unpause() internal virtual whenPaused {
        paused = false;
        emit Unpaused(msg.sender);
    }
}

// 多重继承（继承顺序很重要）
contract MyToken is Ownable, Pausable {
    string public name = "MyToken";
    mapping(address => uint256) public balances;
    
    function mint(address to, uint256 amount) public onlyOwner whenNotPaused {
        balances[to] += amount;
    }
    
    function pause() public onlyOwner {
        _pause();
    }
    
    function unpause() public onlyOwner {
        _unpause();
    }
    
    // 如果父合约有同名函数，需要明确指定
    function transferOwnership(address newOwner) public override onlyOwner {
        super.transferOwnership(newOwner);
    }
}
```

### 1.3 函数重写和super关键字

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Base1 {
    function foo() public virtual returns (string memory) {
        return "Base1";
    }
    
    function bar() public virtual returns (string memory) {
        return "Base1.bar";
    }
}

contract Base2 {
    function foo() public virtual returns (string memory) {
        return "Base2";
    }
}

// 多重继承中的函数重写
contract Child is Base1, Base2 {
    // 必须override所有父合约的同名函数
    function foo() public override(Base1, Base2) returns (string memory) {
        return "Child";
    }
    
    // 调用父合约函数
    function callParentFoo() public view returns (string memory) {
        return super.foo();  // 调用最近的父合约
    }
    
    function callBase1Foo() public view returns (string memory) {
        return Base1.foo();  // 明确指定调用Base1
    }
    
    function bar() public override returns (string memory) {
        string memory parentResult = super.bar();
        return string(abi.encodePacked(parentResult, " + Child"));
    }
}
```

### 1.4 构造函数继承

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Parent {
    uint256 public parentValue;
    
    constructor(uint256 _value) {
        parentValue = _value;
    }
}

// 方式1：在继承时传参
contract Child1 is Parent(100) {
    uint256 public childValue;
    
    constructor(uint256 _childValue) {
        childValue = _childValue;
    }
}

// 方式2：在子合约构造函数中传参
contract Child2 is Parent {
    uint256 public childValue;
    
    constructor(uint256 _parentValue, uint256 _childValue) 
        Parent(_parentValue) 
    {
        childValue = _childValue;
    }
}

// 多重继承的构造函数
contract A {
    uint256 public a;
    constructor(uint256 _a) {
        a = _a;
    }
}

contract B {
    uint256 public b;
    constructor(uint256 _b) {
        b = _b;
    }
}

contract C is A, B {
    // 调用父合约构造函数（按继承顺序）
    constructor(uint256 _a, uint256 _b) 
        A(_a) 
        B(_b) 
    {
        // 子合约初始化
    }
}
```

---

## 🔌 Part 2: 接口 (1小时)

### 2.1 定义和使用接口

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// 定义接口
interface IERC20 {
    // 接口只能有函数声明，不能有实现
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    
    // 接口可以声明事件
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

// 实现接口
contract MyToken is IERC20 {
    string public name = "MyToken";
    string public symbol = "MTK";
    uint8 public decimals = 18;
    uint256 private _totalSupply;
    
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    
    constructor(uint256 initialSupply) {
        _totalSupply = initialSupply;
        _balances[msg.sender] = initialSupply;
        emit Transfer(address(0), msg.sender, initialSupply);
    }
    
    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }
    
    function balanceOf(address account) external view override returns (uint256) {
        return _balances[account];
    }
    
    function transfer(address to, uint256 amount) external override returns (bool) {
        require(to != address(0), "Transfer to zero address");
        require(_balances[msg.sender] >= amount, "Insufficient balance");
        
        _balances[msg.sender] -= amount;
        _balances[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function allowance(address owner, address spender) external view override returns (uint256) {
        return _allowances[owner][spender];
    }
    
    function approve(address spender, uint256 amount) external override returns (bool) {
        _allowances[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external override returns (bool) {
        require(to != address(0), "Transfer to zero address");
        require(_balances[from] >= amount, "Insufficient balance");
        require(_allowances[from][msg.sender] >= amount, "Insufficient allowance");
        
        _balances[from] -= amount;
        _balances[to] += amount;
        _allowances[from][msg.sender] -= amount;
        
        emit Transfer(from, to, amount);
        return true;
    }
}
```

### 2.2 接口的使用场景

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// 使用接口与其他合约交互
contract TokenExchange {
    IERC20 public token;
    
    constructor(address tokenAddress) {
        token = IERC20(tokenAddress);
    }
    
    // 存入代币
    function deposit(uint256 amount) public {
        // 调用ERC20代币的transferFrom
        require(
            token.transferFrom(msg.sender, address(this), amount),
            "Transfer failed"
        );
    }
    
    // 提取代币
    function withdraw(uint256 amount) public {
        require(
            token.transfer(msg.sender, amount),
            "Transfer failed"
        );
    }
    
    // 查询余额
    function getBalance() public view returns (uint256) {
        return token.balanceOf(address(this));
    }
}

// 多个接口组合
interface IUniswapV2Router {
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);
}

contract DeFiIntegration {
    IERC20 public tokenA;
    IERC20 public tokenB;
    IUniswapV2Router public router;
    
    constructor(
        address _tokenA,
        address _tokenB,
        address _router
    ) {
        tokenA = IERC20(_tokenA);
        tokenB = IERC20(_tokenB);
        router = IUniswapV2Router(_router);
    }
    
    function swap(uint256 amountIn) public {
        // 授权router使用代币
        tokenA.approve(address(router), amountIn);
        
        // 构建交易路径
        address[] memory path = new address[](2);
        path[0] = address(tokenA);
        path[1] = address(tokenB);
        
        // 执行交换
        router.swapExactTokensForTokens(
            amountIn,
            0,
            path,
            msg.sender,
            block.timestamp + 300
        );
    }
}
```

---

## 📄 Part 3: 抽象合约 (45分钟)

### 3.1 抽象合约基础

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// 抽象合约（包含未实现的函数）
abstract contract AbstractToken {
    string public name;
    uint256 public totalSupply;
    
    constructor(string memory _name, uint256 _totalSupply) {
        name = _name;
        totalSupply = _totalSupply;
    }
    
    // 已实现的函数
    function getName() public view returns (string memory) {
        return name;
    }
    
    // 未实现的函数（必须在子合约中实现）
    function transfer(address to, uint256 amount) public virtual returns (bool);
    function balanceOf(address account) public view virtual returns (uint256);
}

// 实现抽象合约
contract ConcreteToken is AbstractToken {
    mapping(address => uint256) private balances;
    
    constructor(string memory _name, uint256 _totalSupply) 
        AbstractToken(_name, _totalSupply) 
    {
        balances[msg.sender] = _totalSupply;
    }
    
    // 实现抽象函数
    function transfer(address to, uint256 amount) public override returns (bool) {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] -= amount;
        balances[to] += amount;
        return true;
    }
    
    function balanceOf(address account) public view override returns (uint256) {
        return balances[account];
    }
}
```

### 3.2 抽象合约vs接口

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// 接口：只能声明外部函数
interface IStorage {
    function store(uint256 value) external;
    function retrieve() external view returns (uint256);
}

// 抽象合约：可以有状态变量和实现的函数
abstract contract AbstractStorage {
    uint256 internal storedValue;
    
    // 已实现的函数
    function retrieve() public view virtual returns (uint256) {
        return storedValue;
    }
    
    // 未实现的函数
    function store(uint256 value) public virtual;
    
    // 内部函数
    function _validate(uint256 value) internal pure returns (bool) {
        return value > 0;
    }
}

// 实现抽象合约
contract ConcreteStorage is AbstractStorage {
    event ValueStored(uint256 value);
    
    function store(uint256 value) public override {
        require(_validate(value), "Invalid value");
        storedValue = value;
        emit ValueStored(value);
    }
}
```

---

## 📚 Part 4: 库(Library) (1小时)

### 4.1 库的定义和使用

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// 定义库
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

// 使用库（方式1：直接调用）
contract Calculator1 {
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return SafeMath.add(a, b);
    }
    
    function sub(uint256 a, uint256 b) public pure returns (uint256) {
        return SafeMath.sub(a, b);
    }
}

// 使用库（方式2：using for）
contract Calculator2 {
    using SafeMath for uint256;
    
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a.add(b);  // 等同于 SafeMath.add(a, b)
    }
    
    function complexCalculation(uint256 a, uint256 b, uint256 c) 
        public 
        pure 
        returns (uint256) 
    {
        return a.add(b).mul(c).div(2);
    }
}
```

### 4.2 高级库使用

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

// 字符串操作库
library StringUtils {
    function length(string memory str) internal pure returns (uint256) {
        return bytes(str).length;
    }
    
    function concat(string memory a, string memory b) 
        internal 
        pure 
        returns (string memory) 
    {
        return string(abi.encodePacked(a, b));
    }
    
    function equals(string memory a, string memory b) 
        internal 
        pure 
        returns (bool) 
    {
        return keccak256(bytes(a)) == keccak256(bytes(b));
    }
}

// 地址操作库
library AddressUtils {
    function isContract(address account) internal view returns (bool) {
        return account.code.length > 0;
    }
    
    function sendValue(address payable recipient, uint256 amount) internal {
        require(address(this).balance >= amount, "Insufficient balance");
        (bool success, ) = recipient.call{value: amount}("");
        require(success, "Transfer failed");
    }
}

// 数组操作库
library ArrayUtils {
    function sum(uint256[] memory arr) internal pure returns (uint256) {
        uint256 total = 0;
        for (uint256 i = 0; i < arr.length; i++) {
            total += arr[i];
        }
        return total;
    }
    
    function average(uint256[] memory arr) internal pure returns (uint256) {
        require(arr.length > 0, "Empty array");
        return sum(arr) / arr.length;
    }
    
    function contains(uint256[] memory arr, uint256 value) 
        internal 
        pure 
        returns (bool) 
    {
        for (uint256 i = 0; i < arr.length; i++) {
            if (arr[i] == value) return true;
        }
        return false;
    }
}

// 使用多个库
contract UtilityContract {
    using StringUtils for string;
    using AddressUtils for address;
    using ArrayUtils for uint256[];
    
    function testString() public pure returns (uint256) {
        string memory str = "Hello";
        return str.length();
    }
    
    function testAddress(address addr) public view returns (bool) {
        return addr.isContract();
    }
    
    function testArray(uint256[] memory arr) public pure returns (uint256) {
        return arr.average();
    }
}
```

---

## 🎨 Part 5: 修饰器高级用法 (45分钟)

### 5.1 带参数的修饰器

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract ModifierAdvanced {
    address public owner;
    mapping(address => bool) public admins;
    mapping(address => uint256) public balances;
    
    constructor() {
        owner = msg.sender;
        admins[msg.sender] = true;
    }
    
    // 基础修饰器
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    modifier onlyAdmin() {
        require(admins[msg.sender], "Not admin");
        _;
    }
    
    // 带参数的修饰器
    modifier minAmount(uint256 amount) {
        require(msg.value >= amount, "Insufficient amount");
        _;
    }
    
    modifier validAddress(address addr) {
        require(addr != address(0), "Invalid address");
        _;
    }
    
    modifier hasBalance(address user, uint256 amount) {
        require(balances[user] >= amount, "Insufficient balance");
        _;
    }
    
    // 使用多个修饰器
    function deposit() 
        public 
        payable 
        minAmount(0.01 ether) 
    {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) 
        public 
        hasBalance(msg.sender, amount) 
    {
        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
    
    function transferAdmin(address newAdmin) 
        public 
        onlyOwner 
        validAddress(newAdmin) 
    {
        admins[newAdmin] = true;
    }
}
```

### 5.2 修饰器组合

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract ModifierComposition {
    bool public paused;
    uint256 public lockTime;
    mapping(address => bool) public whitelist;
    
    modifier whenNotPaused() {
        require(!paused, "Paused");
        _;
    }
    
    modifier afterLockTime() {
        require(block.timestamp > lockTime, "Still locked");
        _;
    }
    
    modifier onlyWhitelisted() {
        require(whitelist[msg.sender], "Not whitelisted");
        _;
    }
    
    // 修饰器的执行顺序是从左到右
    function restrictedFunction() 
        public 
        whenNotPaused 
        afterLockTime 
        onlyWhitelisted 
    {
        // 函数逻辑
    }
    
    // 修饰器可以在函数体中多次使用_
    modifier checkAndUpdate() {
        // 执行前检查
        require(!paused, "Paused");
        _;  // 执行函数
        // 执行后操作
        emit FunctionExecuted(msg.sender);
    }
    
    event FunctionExecuted(address user);
}
```

---

## 📢 Part 6: 事件和日志 (45分钟)

### 6.1 事件基础

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract EventsExample {
    // 定义事件
    event Transfer(address indexed from, address indexed to, uint256 amount);
    event Approval(address indexed owner, address indexed spender, uint256 amount);
    event DataUpdated(string indexed key, string value, uint256 timestamp);
    
    mapping(address => uint256) public balances;
    
    // 触发事件
    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        balances[msg.sender] -= amount;
        balances[to] += amount;
        
        // 触发事件
        emit Transfer(msg.sender, to, amount);
    }
    
    // 带索引的事件（最多3个indexed参数）
    function updateData(string memory key, string memory value) public {
        emit DataUpdated(key, value, block.timestamp);
    }
}
```

### 6.2 事件监听和查询

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract EventLogger {
    // 不同类型的事件
    event SimpleEvent(string message);
    event IndexedEvent(address indexed user, uint256 indexed id, string data);
    event ComplexEvent(
        address indexed user,
        uint256 indexed action,
        bytes32 indexed hash,
        string message,
        uint256 timestamp
    );
    
    struct Action {
        address user;
        uint256 timestamp;
        string action;
    }
    
    Action[] public actions;
    
    function logSimple(string memory message) public {
        emit SimpleEvent(message);
    }
    
    function logIndexed(uint256 id, string memory data) public {
        emit IndexedEvent(msg.sender, id, data);
    }
    
    function logComplex(uint256 actionType, string memory message) public {
        bytes32 hash = keccak256(abi.encodePacked(msg.sender, actionType, message));
        emit ComplexEvent(msg.sender, actionType, hash, message, block.timestamp);
        
        actions.push(Action({
            user: msg.sender,
            timestamp: block.timestamp,
            action: message
        }));
    }
    
    function getActionsCount() public view returns (uint256) {
        return actions.length;
    }
}
```

---

## 📝 今日作业

### 作业1: 继承体系设计

```solidity
/**
 * 设计一个完整的权限管理系统
 * 
 * 要求：
 * 1. Ownable合约：所有者管理
 * 2. Roles合约：角色管理（管理员、操作员、用户）
 * 3. Pausable合约：暂停功能
 * 4. 主合约继承以上所有合约
 * 5. 实现：
 *    - 添加/删除角色
 *    - 基于角色的权限控制
 *    - 暂停/恢复合约
 *    - 所有权转移
 */
```

### 作业2: 接口实现

```solidity
/**
 * 实现一个完整的ERC20代币
 * 
 * 要求：
 * 1. 实现IERC20接口的所有函数
 * 2. 添加铸造和销毁功能
 * 3. 添加暂停功能
 * 4. 使用SafeMath库
 * 5. 添加详细的事件日志
 * 6. 编写测试用例
 */
```

### 作业3: 库的创建和使用

```solidity
/**
 * 创建工具库集合
 * 
 * 要求：
 * 1. MathLib：数学运算库
 *    - 百分比计算
 *    - 最大最小值
 *    - 平均值
 * 2. ArrayLib：数组操作库
 *    - 排序
 *    - 去重
 *    - 查找
 * 3. StringLib：字符串操作库
 *    - 转换
 *    - 比较
 *    - 拼接
 * 4. 在主合约中使用所有库
 */
```

---

## ✅ 今日检查清单

- [ ] 掌握单继承和多重继承
- [ ] 理解函数重写和super
- [ ] 掌握接口的定义和使用
- [ ] 理解抽象合约
- [ ] 掌握库的创建和使用
- [ ] 理解修饰器的高级用法
- [ ] 掌握事件和日志
- [ ] 完成所有作业

---

## 🆘 常见问题FAQ

### Q1: 接口和抽象合约的区别？

```solidity
// 接口：
// - 只能声明函数，不能实现
// - 不能有状态变量
// - 不能有构造函数
// - 所有函数必须是external

// 抽象合约：
// - 可以有实现的函数
// - 可以有状态变量
// - 可以有构造函数
// - 函数可以是任何可见性
```

### Q2: 库和合约的区别？

```solidity
// 库：
// - 不能有状态变量（除非是constant）
// - 不能继承或被继承
// - 不能接收ETH
// - 主要用于代码复用

// 合约：
// - 可以有状态变量
// - 可以继承
// - 可以接收ETH
```

### Q3: 何时使用继承vs组合？

```solidity
// 使用继承：is-a关系
contract Dog is Animal { }  // Dog是一种Animal

// 使用组合：has-a关系
contract Wallet {
    IERC20 public token;  // Wallet有一个token
}
```

---

## 📅 明日预告

明天学习Solidity进阶特性（下）：
- 高级数据结构
- Gas优化技巧
- 内联汇编
- 设计模式

**🎉 完成Day 3！继续加油！**
