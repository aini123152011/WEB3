# Week 2 - Day 2: Solidity语法基础（下）

**学习日期**: ___________
**预计用时**: 5-6小时  
**难度等级**: ⭐⭐⭐ (进阶)

## 📚 今日学习目标

- ✅ 掌握数组操作
- ✅ 掌握映射(Mapping)使用
- ✅ 理解结构体(Struct)
- ✅ 深入理解函数
- ✅ 掌握错误处理机制
- ✅ 理解特殊函数(receive/fallback)

---

## 📊 Part 1: 数组 (1小时)

### 1.1 固定长度数组

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract FixedArrays {
    // 固定长度数组声明
    uint256[5] public fixedArray;
    address[3] public addresses;
    bool[10] public flags;
    
    constructor() {
        // 初始化固定数组
        fixedArray = [1, 2, 3, 4, 5];
        addresses = [
            0x5B38Da6a701c568545dCfcB03FcB875f56beddC4,
            0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2,
            0x4B20993Bc481177ec7E8f571ceCaE8A9e22C02db
        ];
    }
    
    // 访问数组元素
    function getElement(uint256 index) public view returns (uint256) {
        require(index < fixedArray.length, "Index out of bounds");
        return fixedArray[index];
    }
    
    // 修改数组元素
    function setElement(uint256 index, uint256 value) public {
        require(index < fixedArray.length, "Index out of bounds");
        fixedArray[index] = value;
    }
    
    // 获取数组长度（固定数组长度是常量）
    function getLength() public view returns (uint256) {
        return fixedArray.length;  // 总是返回5
    }
    
    // 遍历数组
    function sumArray() public view returns (uint256) {
        uint256 sum = 0;
        for (uint256 i = 0; i < fixedArray.length; i++) {
            sum += fixedArray[i];
        }
        return sum;
    }
}
```

### 1.2 动态数组

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract DynamicArrays {
    // 动态数组声明
    uint256[] public numbers;
    address[] public users;
    string[] public names;
    
    // 添加元素
    function pushElement(uint256 _value) public {
        numbers.push(_value);
    }
    
    // 删除最后一个元素
    function popElement() public {
        require(numbers.length > 0, "Array is empty");
        numbers.pop();
    }
    
    // 获取长度
    function getLength() public view returns (uint256) {
        return numbers.length;
    }
    
    // 访问元素
    function getElement(uint256 index) public view returns (uint256) {
        require(index < numbers.length, "Index out of bounds");
        return numbers[index];
    }
    
    // 修改元素
    function setElement(uint256 index, uint256 value) public {
        require(index < numbers.length, "Index out of bounds");
        numbers[index] = value;
    }
    
    // 删除元素（不改变数组长度，只是置为默认值）
    function deleteElement(uint256 index) public {
        require(index < numbers.length, "Index out of bounds");
        delete numbers[index];  // 值变为0，但长度不变
    }
    
    // 批量添加
    function pushMultiple(uint256[] memory _values) public {
        for (uint256 i = 0; i < _values.length; i++) {
            numbers.push(_values[i]);
        }
    }
    
    // 返回整个数组
    function getAllNumbers() public view returns (uint256[] memory) {
        return numbers;
    }
    
    // 清空数组
    function clearArray() public {
        delete numbers;  // 长度变为0
    }
    
    // 数组求和
    function sumArray() public view returns (uint256) {
        uint256 sum = 0;
        for (uint256 i = 0; i < numbers.length; i++) {
            sum += numbers[i];
        }
        return sum;
    }
    
    // 查找元素
    function findElement(uint256 value) public view returns (bool found, uint256 index) {
        for (uint256 i = 0; i < numbers.length; i++) {
            if (numbers[i] == value) {
                return (true, i);
            }
        }
        return (false, 0);
    }
}
```

### 1.3 数组高级操作

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract AdvancedArrays {
    uint256[] public numbers;
    
    // 移除指定位置元素（保持顺序）
    function removeOrdered(uint256 index) public {
        require(index < numbers.length, "Index out of bounds");
        
        // 将后面的元素前移
        for (uint256 i = index; i < numbers.length - 1; i++) {
            numbers[i] = numbers[i + 1];
        }
        numbers.pop();
    }
    
    // 移除指定位置元素（不保持顺序，更省gas）
    function removeUnordered(uint256 index) public {
        require(index < numbers.length, "Index out of bounds");
        
        // 将最后一个元素移到要删除的位置
        numbers[index] = numbers[numbers.length - 1];
        numbers.pop();
    }
    
    // 插入元素
    function insert(uint256 index, uint256 value) public {
        require(index <= numbers.length, "Index out of bounds");
        
        numbers.push();  // 增加长度
        
        // 将元素后移
        for (uint256 i = numbers.length - 1; i > index; i--) {
            numbers[i] = numbers[i - 1];
        }
        
        numbers[index] = value;
    }
    
    // 数组切片（memory操作）
    function slice(uint256 start, uint256 end) public view returns (uint256[] memory) {
        require(start < end && end <= numbers.length, "Invalid range");
        
        uint256[] memory result = new uint256[](end - start);
        for (uint256 i = start; i < end; i++) {
            result[i - start] = numbers[i];
        }
        return result;
    }
    
    // 数组反转
    function reverse() public {
        uint256 length = numbers.length;
        for (uint256 i = 0; i < length / 2; i++) {
            uint256 temp = numbers[i];
            numbers[i] = numbers[length - 1 - i];
            numbers[length - 1 - i] = temp;
        }
    }
    
    // 数组去重（保持顺序）
    function removeDuplicates() public {
        if (numbers.length == 0) return;
        
        uint256 writeIndex = 1;
        for (uint256 readIndex = 1; readIndex < numbers.length; readIndex++) {
            bool isDuplicate = false;
            for (uint256 j = 0; j < writeIndex; j++) {
                if (numbers[readIndex] == numbers[j]) {
                    isDuplicate = true;
                    break;
                }
            }
            if (!isDuplicate) {
                numbers[writeIndex] = numbers[readIndex];
                writeIndex++;
            }
        }
        
        // 调整数组长度
        while (numbers.length > writeIndex) {
            numbers.pop();
        }
    }
}
```

---

## 🗺️ Part 2: 映射 (1小时)

### 2.1 基础映射

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract BasicMapping {
    // 简单映射
    mapping(address => uint256) public balances;
    mapping(uint256 => string) public names;
    mapping(address => bool) public isRegistered;
    
    // 添加/更新映射
    function setBalance(address user, uint256 amount) public {
        balances[user] = amount;
    }
    
    function register(address user) public {
        isRegistered[user] = true;
    }
    
    // 读取映射（不存在的key返回默认值）
    function getBalance(address user) public view returns (uint256) {
        return balances[user];  // 如果不存在，返回0
    }
    
    // 删除映射值（恢复为默认值）
    function deleteBalance(address user) public {
        delete balances[user];  // 余额变为0
    }
    
    // 检查是否存在（对于bool类型）
    function checkRegistered(address user) public view returns (bool) {
        return isRegistered[user];
    }
}
```

### 2.2 嵌套映射

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract NestedMapping {
    // 二维映射：授权额度
    mapping(address => mapping(address => uint256)) public allowances;
    
    // 三维映射
    mapping(address => mapping(uint256 => mapping(string => bool))) public complexData;
    
    // 设置授权
    function approve(address spender, uint256 amount) public {
        allowances[msg.sender][spender] = amount;
    }
    
    // 查询授权
    function getAllowance(address owner, address spender) public view returns (uint256) {
        return allowances[owner][spender];
    }
    
    // 使用授权
    function transferFrom(address from, address to, uint256 amount) public {
        require(allowances[from][msg.sender] >= amount, "Insufficient allowance");
        
        allowances[from][msg.sender] -= amount;
        // 执行转账逻辑...
    }
    
    // 设置复杂数据
    function setComplexData(
        address user,
        uint256 id,
        string memory key,
        bool value
    ) public {
        complexData[user][id][key] = value;
    }
}
```

### 2.3 映射与数组结合

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MappingWithArray {
    // 用户信息
    struct User {
        string name;
        uint256 age;
        bool isActive;
    }
    
    // 映射存储用户
    mapping(address => User) public users;
    
    // 数组存储所有用户地址（用于遍历）
    address[] public userAddresses;
    
    // 检查用户是否存在
    mapping(address => bool) public userExists;
    
    // 注册用户
    function registerUser(string memory name, uint256 age) public {
        require(!userExists[msg.sender], "User already exists");
        
        users[msg.sender] = User({
            name: name,
            age: age,
            isActive: true
        });
        
        userAddresses.push(msg.sender);
        userExists[msg.sender] = true;
    }
    
    // 更新用户
    function updateUser(string memory name, uint256 age) public {
        require(userExists[msg.sender], "User does not exist");
        
        users[msg.sender].name = name;
        users[msg.sender].age = age;
    }
    
    // 停用用户
    function deactivateUser() public {
        require(userExists[msg.sender], "User does not exist");
        users[msg.sender].isActive = false;
    }
    
    // 获取所有用户数量
    function getUserCount() public view returns (uint256) {
        return userAddresses.length;
    }
    
    // 获取所有活跃用户
    function getActiveUsers() public view returns (address[] memory) {
        uint256 activeCount = 0;
        
        // 计数
        for (uint256 i = 0; i < userAddresses.length; i++) {
            if (users[userAddresses[i]].isActive) {
                activeCount++;
            }
        }
        
        // 填充结果
        address[] memory activeUsers = new address[](activeCount);
        uint256 index = 0;
        for (uint256 i = 0; i < userAddresses.length; i++) {
            if (users[userAddresses[i]].isActive) {
                activeUsers[index] = userAddresses[i];
                index++;
            }
        }
        
        return activeUsers;
    }
}
```

---

## 🏗️ Part 3: 结构体 (45分钟)

### 3.1 基础结构体

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract BasicStruct {
    // 定义结构体
    struct Book {
        string title;
        string author;
        uint256 price;
        bool isAvailable;
    }
    
    struct Person {
        string name;
        uint256 age;
        address wallet;
    }
    
    // 状态变量
    Book public featuredBook;
    Person[] public people;
    mapping(uint256 => Book) public books;
    
    uint256 public bookCount;
    
    // 创建结构体（方式1：按顺序）
    function createBook1() public {
        featuredBook = Book("Solidity Guide", "Alice", 100, true);
    }
    
    // 创建结构体（方式2：命名参数）
    function createBook2() public {
        featuredBook = Book({
            title: "Web3 Development",
            author: "Bob",
            price: 200,
            isAvailable: true
        });
    }
    
    // 创建结构体（方式3：逐个赋值）
    function createBook3() public {
        Book memory newBook;
        newBook.title = "DeFi Handbook";
        newBook.author = "Charlie";
        newBook.price = 150;
        newBook.isAvailable = false;
        
        featuredBook = newBook;
    }
    
    // 添加到数组
    function addPerson(string memory name, uint256 age, address wallet) public {
        people.push(Person(name, age, wallet));
        
        // 或使用命名参数
        people.push(Person({
            name: name,
            age: age,
            wallet: wallet
        }));
    }
    
    // 添加到映射
    function addBook(string memory title, string memory author, uint256 price) public {
        bookCount++;
        books[bookCount] = Book({
            title: title,
            author: author,
            price: price,
            isAvailable: true
        });
    }
    
    // 修改结构体
    function updateBookPrice(uint256 id, uint256 newPrice) public {
        require(id <= bookCount, "Book does not exist");
        books[id].price = newPrice;
    }
    
    // 获取结构体
    function getBook(uint256 id) public view returns (
        string memory title,
        string memory author,
        uint256 price,
        bool isAvailable
    ) {
        Book storage book = books[id];
        return (book.title, book.author, book.price, book.isAvailable);
    }
}
```

### 3.2 嵌套结构体

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract NestedStruct {
    struct Address {
        string street;
        string city;
        string country;
        uint256 zipCode;
    }
    
    struct Contact {
        string email;
        string phone;
    }
    
    struct Employee {
        string name;
        uint256 age;
        Address homeAddress;
        Contact contact;
        uint256 salary;
    }
    
    struct Company {
        string name;
        Address headquarters;
        Employee[] employees;
    }
    
    Company public myCompany;
    mapping(address => Employee) public employees;
    
    // 创建公司
    function createCompany(
        string memory companyName,
        string memory street,
        string memory city
    ) public {
        myCompany.name = companyName;
        myCompany.headquarters = Address({
            street: street,
            city: city,
            country: "USA",
            zipCode: 12345
        });
    }
    
    // 添加员工
    function addEmployee(
        string memory name,
        uint256 age,
        string memory email,
        uint256 salary
    ) public {
        Employee memory newEmployee = Employee({
            name: name,
            age: age,
            homeAddress: Address({
                street: "",
                city: "",
                country: "",
                zipCode: 0
            }),
            contact: Contact({
                email: email,
                phone: ""
            }),
            salary: salary
        });
        
        myCompany.employees.push(newEmployee);
        employees[msg.sender] = newEmployee;
    }
    
    // 更新员工地址
    function updateEmployeeAddress(
        uint256 employeeIndex,
        string memory street,
        string memory city
    ) public {
        require(employeeIndex < myCompany.employees.length, "Invalid index");
        
        myCompany.employees[employeeIndex].homeAddress.street = street;
        myCompany.employees[employeeIndex].homeAddress.city = city;
    }
    
    // 获取员工数量
    function getEmployeeCount() public view returns (uint256) {
        return myCompany.employees.length;
    }
}
```

---

## 🔧 Part 4: 函数详解 (1小时)

### 4.1 函数可见性

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract FunctionVisibility {
    uint256 private data = 100;
    
    // public - 任何人都可以调用
    function publicFunction() public view returns (uint256) {
        return data;
    }
    
    // external - 只能从外部调用
    function externalFunction() external view returns (uint256) {
        return data;
    }
    
    // internal - 只能在当前合约和子合约调用
    function internalFunction() internal view returns (uint256) {
        return data;
    }
    
    // private - 只能在当前合约调用
    function privateFunction() private view returns (uint256) {
        return data;
    }
    
    // 测试调用
    function testCalls() public view returns (uint256) {
        // 可以调用public和internal
        uint256 a = publicFunction();
        uint256 b = internalFunction();
        uint256 c = privateFunction();
        
        // 不能直接调用external，需要使用this
        // uint256 d = externalFunction();  // 错误
        uint256 d = this.externalFunction();  // 正确
        
        return a + b + c + d;
    }
}
```

### 4.2 函数修饰符

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract FunctionModifiers {
    uint256 public value;
    
    // pure - 不读取也不修改状态
    function pureFunction(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;
    }
    
    // view - 只读取状态，不修改
    function viewFunction() public view returns (uint256) {
        return value;
    }
    
    // 默认 - 可以修改状态
    function normalFunction(uint256 newValue) public {
        value = newValue;
    }
    
    // payable - 可以接收ETH
    function payableFunction() public payable {
        value = msg.value;
    }
    
    // virtual - 可以被重写
    function virtualFunction() public virtual returns (string memory) {
        return "Base";
    }
}

contract Child is FunctionModifiers {
    // override - 重写父合约函数
    function virtualFunction() public override returns (string memory) {
        return "Child";
    }
}
```

### 4.3 函数返回值

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract FunctionReturns {
    // 单个返回值
    function singleReturn() public pure returns (uint256) {
        return 42;
    }
    
    // 多个返回值
    function multipleReturns() public pure returns (
        uint256 a,
        string memory b,
        bool c
    ) {
        return (42, "hello", true);
    }
    
    // 命名返回值
    function namedReturns() public pure returns (
        uint256 sum,
        uint256 product
    ) {
        sum = 10;
        product = 20;
        // 自动返回
    }
    
    // 解构返回值
    function destructureReturns() public pure returns (uint256, uint256) {
        (uint256 a, uint256 b) = multipleReturns();
        return (a, b);
    }
    
    // 忽略部分返回值
    function ignoreReturns() public pure returns (uint256) {
        (uint256 a, , ) = multipleReturns();
        return a;
    }
    
    // 返回数组
    function returnArray() public pure returns (uint256[] memory) {
        uint256[] memory arr = new uint256[](3);
        arr[0] = 1;
        arr[1] = 2;
        arr[2] = 3;
        return arr;
    }
    
    // 返回结构体
    struct Data {
        uint256 id;
        string name;
    }
    
    function returnStruct() public pure returns (Data memory) {
        return Data({
            id: 1,
            name: "Test"
        });
    }
}
```

---

## ⚠️ Part 5: 错误处理 (45分钟)

### 5.1 require、assert、revert

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract ErrorHandling {
    uint256 public balance;
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    // require - 验证输入和条件
    function withdraw(uint256 amount) public {
        require(msg.sender == owner, "Not the owner");
        require(amount <= balance, "Insufficient balance");
        require(amount > 0, "Amount must be positive");
        
        balance -= amount;
    }
    
    // assert - 检查不变量和内部错误
    function internalCalculation(uint256 a, uint256 b) public pure returns (uint256) {
        uint256 result = a + b;
        assert(result >= a);  // 检查没有溢出
        return result;
    }
    
    // revert - 条件性回退
    function complexValidation(uint256 value) public pure {
        if (value < 10) {
            revert("Value too small");
        }
        
        if (value > 100) {
            revert("Value too large");
        }
        
        // 继续执行...
    }
    
    // 使用自定义错误（更省gas）
    error InsufficientBalance(uint256 requested, uint256 available);
    error Unauthorized(address caller);
    
    function withdrawWithCustomError(uint256 amount) public {
        if (msg.sender != owner) {
            revert Unauthorized(msg.sender);
        }
        
        if (amount > balance) {
            revert InsufficientBalance(amount, balance);
        }
        
        balance -= amount;
    }
}
```

### 5.2 Try-Catch

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract ExternalContract {
    function mightFail(uint256 value) public pure returns (uint256) {
        require(value > 0, "Value must be positive");
        return value * 2;
    }
}

contract TryCatchExample {
    ExternalContract public externalContract;
    
    event Success(uint256 result);
    event Failure(string reason);
    event LowLevelFailure(bytes data);
    
    constructor(address _externalContract) {
        externalContract = ExternalContract(_externalContract);
    }
    
    // try-catch 外部调用
    function callExternal(uint256 value) public {
        try externalContract.mightFail(value) returns (uint256 result) {
            // 成功
            emit Success(result);
        } catch Error(string memory reason) {
            // require/revert with reason
            emit Failure(reason);
        } catch (bytes memory lowLevelData) {
            // 其他错误
            emit LowLevelFailure(lowLevelData);
        }
    }
    
    // try-catch 合约创建
    function deployContract() public {
        try new ExternalContract() returns (ExternalContract newContract) {
            // 部署成功
            externalContract = newContract;
        } catch Error(string memory reason) {
            emit Failure(reason);
        } catch {
            emit Failure("Unknown error");
        }
    }
}
```

---

## 💰 Part 6: 特殊函数 (30分钟)

### 6.1 receive 和 fallback

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SpecialFunctions {
    event Received(address sender, uint256 amount);
    event FallbackCalled(address sender, uint256 amount, bytes data);
    
    // receive - 接收ETH（没有数据）
    receive() external payable {
        emit Received(msg.sender, msg.value);
    }
    
    // fallback - 接收ETH（有数据）或调用不存在的函数
    fallback() external payable {
        emit FallbackCalled(msg.sender, msg.value, msg.data);
    }
    
    // 查询余额
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
    
    /*
    调用流程：
    
    send ETH
        |
        v
    msg.data为空?
        |
        +-- Yes --> receive()存在? --> Yes --> receive()
        |                           --> No --> fallback()
        |
        +-- No --> 调用的函数存在? --> Yes --> 调用函数
                                    --> No --> fallback()存在? --> Yes --> fallback()
                                                                 --> No --> 交易失败
    */
}

contract SendEther {
    // 三种发送ETH的方式
    function sendViaTransfer(address payable _to) public payable {
        _to.transfer(msg.value);  // 2300 gas, 失败会revert
    }
    
    function sendViaSend(address payable _to) public payable {
        bool success = _to.send(msg.value);  // 2300 gas, 返回bool
        require(success, "Send failed");
    }
    
    function sendViaCall(address payable _to) public payable {
        (bool success, ) = _to.call{value: msg.value}("");
        require(success, "Call failed");
    }
}
```

---

## 📝 今日作业

### 作业1: 学生管理系统

```solidity
/**
 * 创建一个学生管理系统合约
 * 
 * 要求：
 * 1. 定义Student结构体（姓名、年龄、成绩数组）
 * 2. 使用mapping存储学生（地址 => Student）
 * 3. 使用数组记录所有学生地址
 * 4. 实现功能：
 *    - 注册学生
 *    - 添加成绩
 *    - 计算平均分
 *    - 获取所有学生
 *    - 获取优秀学生（平均分>90）
 * 5. 添加适当的错误处理
 */
```

### 作业2: 代币交易所

```solidity
/**
 * 创建一个简单的代币交易订单簿
 * 
 * 要求：
 * 1. 定义Order结构体（买/卖、价格、数量、创建者）
 * 2. 使用数组存储买单和卖单
 * 3. 实现功能：
 *    - 创建买单
 *    - 创建卖单
 *    - 取消订单
 *    - 获取所有订单
 *    - 匹配订单（自动成交）
 * 4. 使用receive接收ETH
 * 5. 完善错误处理
 */
```

### 作业3: 投票系统升级

```solidity
/**
 * 升级Week1的投票系统
 * 
 * 新增功能：
 * 1. 使用结构体重构Proposal
 * 2. 添加投票选项（支持多选项投票）
 * 3. 实现投票委托功能
 * 4. 添加投票权重系统
 * 5. 使用自定义错误
 * 6. 添加try-catch错误处理
 */
```

---

## ✅ 今日检查清单

- [ ] 掌握数组的所有操作
- [ ] 理解映射的使用场景
- [ ] 能够设计复杂的数据结构
- [ ] 掌握函数的各种修饰符
- [ ] 理解错误处理机制
- [ ] 掌握receive和fallback
- [ ] 完成所有作业

---

## 🆘 常见问题FAQ

### Q1: 数组删除元素的最佳方式？
```solidity
// 方式1：保持顺序（较贵）
function removeOrdered(uint index) {
    for (uint i = index; i < arr.length - 1; i++) {
        arr[i] = arr[i + 1];
    }
    arr.pop();
}

// 方式2：不保持顺序（省gas）
function removeUnordered(uint index) {
    arr[index] = arr[arr.length - 1];
    arr.pop();
}
```

### Q2: 如何遍历mapping？
```solidity
// mapping不能直接遍历，需要配合数组
mapping(address => uint256) public balances;
address[] public users;  // 记录所有key

// 添加时同时更新数组
function add(address user, uint256 amount) public {
    balances[user] = amount;
    users.push(user);
}
```

### Q3: require vs assert的区别？
```solidity
// require - 用于验证输入
function withdraw(uint amount) public {
    require(amount <= balance, "Insufficient");  // 返还剩余gas
}

// assert - 用于检查不变量
function add(uint a, uint b) public pure returns (uint) {
    uint c = a + b;
    assert(c >= a);  // 不返还gas，用于发现bug
    return c;
}
```

---

## 📅 明日预告

明天学习Solidity进阶特性（上）：
- 继承
- 接口
- 抽象合约
- 库(Library)

**🎉 完成Day 2！继续加油！**
