# Week 2 - Day 1: Solidity语法基础（上）

**学习日期**: ___________
**预计用时**: 5-6小时  
**难度等级**: ⭐⭐⭐ (进阶)

## 📚 今日学习目标

- ✅ 理解Solidity语言特性
- ✅ 掌握基本数据类型
- ✅ 理解变量作用域和可见性
- ✅ 掌握函数定义和调用
- ✅ 理解修饰器(Modifier)
- ✅ 掌握事件(Event)机制

---

## 🎯 Part 1: Solidity概述 (30分钟)

### 1.1 什么是Solidity？

```solidity
/*
Solidity 是什么？
- 面向合约的高级编程语言
- 用于编写以太坊智能合约
- 语法类似JavaScript/C++
- 静态类型语言
- 支持继承、库和复杂类型

为什么学习Solidity？
- 编写智能合约的主流语言
- EVM兼容链都支持
- 丰富的开发工具生态
- 活跃的社区支持
*/
```

### 1.2 Solidity文件结构

```solidity
// SPDX-License-Identifier: MIT        // 1. 许可证标识
pragma solidity ^0.8.20;                // 2. 编译器版本

// 3. 导入其他文件
import "./OtherContract.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

// 4. 合约定义
contract MyContract {
    // 5. 状态变量
    uint256 public myNumber;
    
    // 6. 事件
    event NumberChanged(uint256 newNumber);
    
    // 7. 修饰器
    modifier onlyPositive(uint256 _num) {
        require(_num > 0, "Must be positive");
        _;
    }
    
    // 8. 构造函数
    constructor(uint256 _initialNumber) {
        myNumber = _initialNumber;
    }
    
    // 9. 函数
    function setNumber(uint256 _newNumber) public onlyPositive(_newNumber) {
        myNumber = _newNumber;
        emit NumberChanged(_newNumber);
    }
}
```

### 1.3 Solidity版本

```solidity
// 精确版本
pragma solidity 0.8.20;

// 版本范围
pragma solidity ^0.8.0;   // >= 0.8.0 且 < 0.9.0
pragma solidity >=0.8.0 <0.9.0;

// 推荐：使用^指定主版本
pragma solidity ^0.8.20;

/*
版本选择建议：
- 使用最新稳定版（0.8.20+）
- 避免使用0.8.0以下版本（安全问题）
- 锁定次版本号以保证兼容性
*/
```

---

## 📊 Part 2: 基本数据类型 (1.5小时)

### 2.1 值类型概览

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract DataTypes {
    // ========== 布尔类型 ==========
    bool public isActive = true;
    bool public isPaused = false;
    
    // ========== 整数类型 ==========
    // 无符号整数
    uint8 public smallNumber = 255;      // 0 to 2^8-1
    uint256 public largeNumber = 1000;   // 0 to 2^256-1
    uint public defaultUint = 100;       // uint = uint256
    
    // 有符号整数
    int8 public smallInt = -128;         // -2^7 to 2^7-1
    int256 public largeInt = -1000;      // -2^255 to 2^255-1
    int public defaultInt = -100;        // int = int256
    
    // ========== 地址类型 ==========
    address public myAddress = 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4;
    address payable public payableAddress;
    
    // ========== 字节类型 ==========
    bytes1 public singleByte = 0x01;
    bytes32 public hash = 0x1234567890123456789012345678901234567890123456789012345678901234;
    
    // ========== 枚举类型 ==========
    enum Status { Pending, Active, Completed, Cancelled }
    Status public currentStatus = Status.Pending;
    
    function demonstrateTypes() public pure {
        // 布尔运算
        bool result = true && false;     // false
        bool result2 = true || false;    // true
        bool result3 = !true;            // false
        
        // 整数运算
        uint a = 10;
        uint b = 3;
        uint sum = a + b;         // 13
        uint diff = a - b;        // 7
        uint product = a * b;     // 30
        uint quotient = a / b;    // 3
        uint remainder = a % b;   // 1
        uint power = a ** 2;      // 100
        
        // 比较运算
        bool isEqual = (a == b);      // false
        bool isGreater = (a > b);     // true
        bool isLess = (a < b);        // false
        
        // 位运算
        uint x = 5;  // 0101
        uint y = 3;  // 0011
        uint and = x & y;         // 0001 = 1
        uint or = x | y;          // 0111 = 7
        uint xor = x ^ y;         // 0110 = 6
        uint not = ~x;            // 1010
        uint leftShift = x << 1;  // 1010 = 10
        uint rightShift = x >> 1; // 0010 = 2
    }
}
```

### 2.2 地址类型详解

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract AddressType {
    address public owner;
    address payable public recipient;
    
    constructor() {
        owner = msg.sender;
        recipient = payable(msg.sender);
    }
    
    // 地址属性
    function addressProperties(address _addr) public view returns (
        uint256 balance,
        uint256 codeSize,
        bytes32 codehash
    ) {
        balance = _addr.balance;           // 地址余额(wei)
        codeSize = _addr.code.length;      // 代码大小
        codehash = _addr.codehash;         // 代码哈希
    }
    
    // 地址类型转换
    function addressConversion() public view {
        // address -> address payable
        address payable payAddr = payable(owner);
        
        // address -> uint160
        uint160 addressAsUint = uint160(owner);
        
        // uint160 -> address
        address addressFromUint = address(addressAsUint);
    }
    
    // 发送ETH
    function sendEth() public payable {
        // 1. transfer (2300 gas, 失败会revert)
        recipient.transfer(1 ether);
        
        // 2. send (2300 gas, 返回bool)
        bool success = recipient.send(1 ether);
        require(success, "Send failed");
        
        // 3. call (转发所有gas, 推荐)
        (bool success2, ) = recipient.call{value: 1 ether}("");
        require(success2, "Call failed");
    }
    
    // 检查地址是否是合约
    function isContract(address _addr) public view returns (bool) {
        return _addr.code.length > 0;
    }
}
```

### 2.3 字节类型和字符串

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract BytesAndStrings {
    // 固定大小字节数组
    bytes1 public b1 = 0x01;
    bytes2 public b2 = 0x0102;
    bytes32 public b32;
    
    // 动态大小字节数组
    bytes public dynamicBytes;
    
    // 字符串
    string public name = "Alice";
    string public greeting = "Hello, World!";
    
    // 固定字节数组操作
    function fixedBytesOperations() public pure {
        bytes1 a = 0x01;
        bytes1 b = 0x02;
        
        // 比较
        bool isEqual = (a == b);        // false
        bool isLess = (a < b);          // true
        
        // 位运算
        bytes1 and = a & b;             // 0x00
        bytes1 or = a | b;              // 0x03
        bytes1 xor = a ^ b;             // 0x03
        bytes1 not = ~a;                // 0xfe
        
        // 移位
        bytes1 left = a << 1;           // 0x02
        bytes1 right = a >> 1;          // 0x00
    }
    
    // 动态字节数组操作
    function dynamicBytesOperations() public {
        // 添加元素
        dynamicBytes.push(0x01);
        dynamicBytes.push(0x02);
        
        // 访问元素
        bytes1 first = dynamicBytes[0];
        
        // 获取长度
        uint len = dynamicBytes.length;
        
        // 删除最后一个元素
        dynamicBytes.pop();
    }
    
    // 字符串操作
    function stringOperations() public view {
        // 获取长度（字节长度，不是字符数）
        uint len = bytes(name).length;
        
        // 字符串拼接（需要转换）
        string memory fullName = string(
            abi.encodePacked(name, " ", "Smith")
        );
        
        // 字符串比较
        bool isEqual = keccak256(bytes(name)) == keccak256(bytes("Alice"));
    }
    
    // 类型转换
    function conversions() public pure {
        // string -> bytes
        string memory str = "Hello";
        bytes memory b = bytes(str);
        
        // bytes32 -> bytes
        bytes32 b32 = keccak256("data");
        bytes memory dynamicB = abi.encodePacked(b32);
        
        // bytes -> bytes32 (如果长度匹配)
        bytes memory data = new bytes(32);
        bytes32 result;
        assembly {
            result := mload(add(data, 32))
        }
    }
}
```

### 2.4 枚举类型

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract EnumExample {
    // 定义枚举
    enum OrderStatus {
        Pending,      // 0
        Confirmed,    // 1
        Shipped,      // 2
        Delivered,    // 3
        Cancelled     // 4
    }
    
    enum PaymentMethod {
        Cash,
        CreditCard,
        Crypto
    }
    
    // 使用枚举
    OrderStatus public status = OrderStatus.Pending;
    PaymentMethod public paymentMethod;
    
    // 状态变量
    struct Order {
        uint256 id;
        OrderStatus status;
        PaymentMethod payment;
        uint256 amount;
    }
    
    mapping(uint256 => Order) public orders;
    uint256 public orderCount;
    
    // 创建订单
    function createOrder(uint256 _amount, PaymentMethod _payment) public returns (uint256) {
        orderCount++;
        orders[orderCount] = Order({
            id: orderCount,
            status: OrderStatus.Pending,
            payment: _payment,
            amount: _amount
        });
        return orderCount;
    }
    
    // 更新订单状态
    function updateOrderStatus(uint256 _orderId, OrderStatus _newStatus) public {
        require(_orderId <= orderCount, "Order does not exist");
        orders[_orderId].status = _newStatus;
    }
    
    // 枚举比较
    function isOrderPending(uint256 _orderId) public view returns (bool) {
        return orders[_orderId].status == OrderStatus.Pending;
    }
    
    // 枚举转换
    function statusToUint(OrderStatus _status) public pure returns (uint) {
        return uint(_status);
    }
    
    function uintToStatus(uint _value) public pure returns (OrderStatus) {
        require(_value <= uint(OrderStatus.Cancelled), "Invalid status");
        return OrderStatus(_value);
    }
    
    // 获取枚举范围
    function getStatusRange() public pure returns (uint min, uint max) {
        min = uint(type(OrderStatus).min);  // 0
        max = uint(type(OrderStatus).max);  // 4
    }
}
```

---

## 🔧 Part 3: 变量和作用域 (1小时)

### 3.1 状态变量

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract StateVariables {
    // 状态变量存储在区块链上（永久存储）
    uint256 public count = 0;
    address public owner;
    mapping(address => uint256) public balances;
    
    // 可见性修饰符
    uint256 public publicVar = 1;       // 自动生成getter函数
    uint256 internal internalVar = 2;   // 只能在当前合约和子合约访问
    uint256 private privateVar = 3;     // 只能在当前合约访问
    
    // 常量（编译时确定，不占存储槽）
    uint256 public constant MAX_SUPPLY = 1000000;
    address public constant DEAD_ADDRESS = 0x000000000000000000000000000000000000dEaD;
    
    // 不可变量（构造时确定，不占存储槽）
    uint256 public immutable DEPLOYMENT_TIME;
    address public immutable DEPLOYER;
    
    constructor() {
        owner = msg.sender;
        DEPLOYMENT_TIME = block.timestamp;
        DEPLOYER = msg.sender;
    }
    
    // 修改状态变量
    function incrementCount() public {
        count += 1;
    }
    
    function updateBalance(address _user, uint256 _amount) public {
        balances[_user] = _amount;
    }
}
```

### 3.2 局部变量和内存管理

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract LocalVariables {
    uint256[] public numbers;
    
    struct Person {
        string name;
        uint256 age;
    }
    
    mapping(uint256 => Person) public people;
    
    // 局部变量（函数内部，临时存储）
    function calculate(uint256 a, uint256 b) public pure returns (uint256) {
        uint256 sum = a + b;         // 局部变量
        uint256 product = a * b;     // 局部变量
        return sum + product;
    }
    
    // 数据位置：storage vs memory vs calldata
    function dataLocations(uint256 _id) public view returns (string memory) {
        // storage - 指向状态变量，永久存储
        Person storage person = people[_id];
        
        // memory - 临时存储，函数结束后释放
        Person memory tempPerson = Person({
            name: "Temp",
            age: 25
        });
        
        // 返回值必须是memory
        return person.name;
    }
    
    // calldata - 只读的函数参数
    function processData(uint256[] calldata _data) public pure returns (uint256) {
        uint256 sum = 0;
        for (uint256 i = 0; i < _data.length; i++) {
            sum += _data[i];
        }
        return sum;
    }
    
    // 复制 vs 引用
    function storageReference() public {
        numbers.push(1);
        numbers.push(2);
        numbers.push(3);
        
        // storage引用（修改会影响原数据）
        uint256[] storage numsRef = numbers;
        numsRef[0] = 100;  // numbers[0]也变成100
        
        // memory复制（修改不影响原数据）
        uint256[] memory numsCopy = numbers;
        numsCopy[1] = 200;  // numbers[1]仍然是2
    }
}
```

### 3.3 全局变量

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract GlobalVariables {
    // 区块相关
    function blockInfo() public view returns (
        uint256 blockNumber,
        uint256 timestamp,
        uint256 difficulty,
        uint256 gasLimit,
        address coinbase
    ) {
        blockNumber = block.number;        // 当前区块号
        timestamp = block.timestamp;       // 当前区块时间戳
        difficulty = block.prevrandao;     // 前一个区块的随机数
        gasLimit = block.gaslimit;         // 当前区块gas限制
        coinbase = block.coinbase;         // 当前矿工地址
    }
    
    // 交易相关
    function transactionInfo() public view returns (
        address sender,
        uint256 gasPrice,
        uint256 value,
        bytes memory data
    ) {
        sender = msg.sender;               // 消息发送者
        gasPrice = tx.gasprice;            // 交易gas价格
        value = msg.value;                 // 发送的ETH数量(wei)
        data = msg.data;                   // 完整的calldata
    }
    
    // Gas相关
    function gasInfo() public view returns (uint256) {
        return gasleft();                  // 剩余gas
    }
    
    // 合约相关
    function contractInfo() public view returns (
        address thisAddress,
        uint256 balance
    ) {
        thisAddress = address(this);       // 当前合约地址
        balance = address(this).balance;   // 当前合约ETH余额
    }
    
    // ABI编码函数
    function abiEncode() public pure returns (bytes memory) {
        // abi.encode - 完整编码
        bytes memory encoded = abi.encode(123, "hello");
        
        // abi.encodePacked - 紧密编码（无填充）
        bytes memory packed = abi.encodePacked(uint8(1), uint8(2));
        
        // abi.encodeWithSignature - 包含函数签名
        bytes memory withSig = abi.encodeWithSignature(
            "transfer(address,uint256)",
            0x123,
            100
        );
        
        return encoded;
    }
    
    // 哈希函数
    function hashFunctions(string memory _text) public pure returns (
        bytes32 keccak,
        bytes32 sha
    ) {
        keccak = keccak256(abi.encodePacked(_text));  // Keccak-256
        sha = sha256(abi.encodePacked(_text));        // SHA-256
    }
}
```

---

## 📝 今日作业

### 作业1: 数据类型练习

```solidity
// contracts/DataTypesExercise.sol

/**
 * 创建一个合约，包含以下功能：
 * 
 * 1. 定义各种数据类型的状态变量：
 *    - uint256, int256, bool, address
 *    - bytes32, string
 *    - 自定义enum (UserRole: Admin, Member, Guest)
 * 
 * 2. 实现函数：
 *    - 整数运算函数（加减乘除模）
 *    - 地址类型转换函数
 *    - 字符串拼接函数
 *    - 枚举操作函数
 * 
 * 3. 测试边界情况：
 *    - 整数溢出（应该revert）
 *    - 除零错误
 *    - 类型转换
 */

// TODO: 实现合约
```

### 作业2: 变量作用域练习

```solidity
// contracts/VariableScope.sol

/**
 * 创建一个图书馆管理合约：
 * 
 * 1. 状态变量：
 *    - 图书总数 (public)
 *    - 管理员地址 (immutable)
 *    - 最大图书数量 (constant)
 * 
 * 2. 数据结构：
 *    struct Book {
 *        string title;
 *        string author;
 *        bool isAvailable;
 *    }
 *    mapping(uint256 => Book) books;
 * 
 * 3. 函数：
 *    - addBook(string memory, string memory)
 *    - borrowBook(uint256) - 使用storage reference
 *    - returnBook(uint256)
 *    - getBookCopy(uint256) - 返回memory copy
 * 
 * 4. 测试storage vs memory的区别
 */

// TODO: 实现合约
```

### 作业3: 全局变量应用

```solidity
// contracts/TimeLock.sol

/**
 * 创建一个时间锁合约：
 * 
 * 1. 功能：
 *    - 存入ETH并设置解锁时间
 *    - 只有在解锁时间后才能提取
 *    - 记录每次存入和提取的时间戳
 * 
 * 2. 使用全局变量：
 *    - block.timestamp
 *    - msg.sender
 *    - msg.value
 *    - address(this).balance
 * 
 * 3. 实现函数：
 *    - deposit(uint256 lockTime) payable
 *    - withdraw()
 *    - checkLockTime() view
 *    - getContractBalance() view
 */

// TODO: 实现合约
```

---

## ✅ 今日检查清单

### 理论知识
- [ ] 理解Solidity文件结构
- [ ] 掌握所有基本数据类型
- [ ] 理解值类型vs引用类型
- [ ] 理解地址类型和payable

### 数据类型
- [ ] 布尔、整数、地址
- [ ] 字节数组和字符串
- [ ] 枚举类型
- [ ] 类型转换

### 变量作用域
- [ ] 状态变量、局部变量
- [ ] storage、memory、calldata
- [ ] 常量和不可变量
- [ ] 全局变量

---

## 🆘 常见问题FAQ

### Q1: uint和uint256有什么区别？
```solidity
// 没有区别，uint是uint256的别名
uint public a = 100;      // 等同于
uint256 public b = 100;

// 但建议使用uint256以提高可读性
```

### Q2: storage和memory的区别？
```solidity
// storage - 永久存储在区块链
uint256[] storage nums = numbers;  // 引用原数据
nums[0] = 100;  // 修改会影响原数据

// memory - 临时存储在内存
uint256[] memory nums = numbers;  // 复制数据
nums[0] = 100;  // 修改不影响原数据
```

### Q3: 为什么需要payable？
```solidity
// payable表示函数可以接收ETH
function deposit() public payable {
    // msg.value包含发送的ETH数量
}

// 没有payable的函数不能接收ETH
function normalFunction() public {
    // 发送ETH会失败
}
```

### Q4: 字符串如何拼接？
```solidity
string memory str1 = "Hello";
string memory str2 = "World";

// 使用abi.encodePacked
string memory result = string(
    abi.encodePacked(str1, " ", str2)
);
// result = "Hello World"
```

---

## 📅 明日预告: Solidity语法基础（下）

明天我们将学习：
- 数组和映射
- 结构体
- 函数详解
- 错误处理
- 特殊函数（receive、fallback）

**今晚准备**：
- 完成今天的所有作业
- 复习数据类型
- 测试各种类型转换

---

## ✍️ 我的学习记录

**完成日期**: ___________
**实际耗时**: _____ 小时

### ✅ 完成情况
- [ ] 理解Solidity基础
- [ ] 掌握数据类型
- [ ] 理解变量作用域
- [ ] 完成作业

### 💡 今日收获
1. 最重要的概念:
2. 最难理解的部分:
3. 实际应用场景:

### 🤔 疑问与思考
- 问题1:
- 问题2:

**🎉 完成Day 1！继续加油！**
