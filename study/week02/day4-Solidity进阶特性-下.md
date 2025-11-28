# Week 2 - Day 4: Solidity进阶特性（下）

**学习日期**: ___________
**预计用时**: 5-6小时  
**难度等级**: ⭐⭐⭐⭐ (进阶)

## 📚 今日学习目标

- ✅ 掌握高级数据结构设计
- ✅ 学习Gas优化技巧
- ✅ 理解代理模式和可升级合约
- ✅ 掌握常用设计模式
- ✅ 了解安全编程实践

---

## 🏗️ Part 1: 高级数据结构 (1.5小时)

### 1.1 迭代映射

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 可迭代的映射
 * 解决mapping无法遍历的问题
 */
contract IterableMapping {
    struct IndexValue {
        uint256 keyIndex;
        uint256 value;
    }
    
    struct KeyFlag {
        uint256 key;
        bool deleted;
    }
    
    mapping(uint256 => IndexValue) private data;
    KeyFlag[] private keys;
    uint256 public size;
    
    // 插入或更新
    function insert(uint256 key, uint256 value) public returns (bool) {
        uint256 keyIndex = data[key].keyIndex;
        
        data[key].value = value;
        
        if (keyIndex > 0) {
            // 键已存在，更新
            return true;
        } else {
            // 新键
            keyIndex = keys.length;
            keys.push(KeyFlag(key, false));
            data[key].keyIndex = keyIndex + 1;
            size++;
            return false;
        }
    }
    
    // 删除
    function remove(uint256 key) public returns (bool) {
        uint256 keyIndex = data[key].keyIndex;
        if (keyIndex == 0) return false;
        
        delete data[key];
        keys[keyIndex - 1].deleted = true;
        size--;
        return true;
    }
    
    // 检查键是否存在
    function contains(uint256 key) public view returns (bool) {
        return data[key].keyIndex > 0;
    }
    
    // 获取值
    function getValue(uint256 key) public view returns (uint256) {
        return data[key].value;
    }
    
    // 迭代
    function iterate_start() public view returns (uint256 keyIndex) {
        return iterate_next(type(uint256).max);
    }
    
    function iterate_valid(uint256 keyIndex) public view returns (bool) {
        return keyIndex < keys.length;
    }
    
    function iterate_next(uint256 keyIndex) public view returns (uint256) {
        keyIndex++;
        while (keyIndex < keys.length && keys[keyIndex].deleted) {
            keyIndex++;
        }
        return keyIndex;
    }
    
    function iterate_get(uint256 keyIndex) public view returns (uint256 key, uint256 value) {
        key = keys[keyIndex].key;
        value = data[key].value;
    }
}
```

### 1.2 优先队列

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 最小堆实现的优先队列
 */
contract PriorityQueue {
    struct Item {
        uint256 priority;
        address data;
    }
    
    Item[] private heap;
    
    function size() public view returns (uint256) {
        return heap.length;
    }
    
    function isEmpty() public view returns (bool) {
        return heap.length == 0;
    }
    
    // 插入元素
    function insert(uint256 priority, address data) public {
        heap.push(Item(priority, data));
        _bubbleUp(heap.length - 1);
    }
    
    // 获取最小元素
    function peek() public view returns (uint256 priority, address data) {
        require(heap.length > 0, "Queue is empty");
        return (heap[0].priority, heap[0].data);
    }
    
    // 删除最小元素
    function pop() public returns (uint256 priority, address data) {
        require(heap.length > 0, "Queue is empty");
        
        Item memory min = heap[0];
        heap[0] = heap[heap.length - 1];
        heap.pop();
        
        if (heap.length > 0) {
            _bubbleDown(0);
        }
        
        return (min.priority, min.data);
    }
    
    // 向上调整
    function _bubbleUp(uint256 index) private {
        while (index > 0) {
            uint256 parent = (index - 1) / 2;
            if (heap[index].priority >= heap[parent].priority) break;
            
            // 交换
            Item memory temp = heap[index];
            heap[index] = heap[parent];
            heap[parent] = temp;
            
            index = parent;
        }
    }
    
    // 向下调整
    function _bubbleDown(uint256 index) private {
        uint256 length = heap.length;
        while (true) {
            uint256 left = 2 * index + 1;
            uint256 right = 2 * index + 2;
            uint256 smallest = index;
            
            if (left < length && heap[left].priority < heap[smallest].priority) {
                smallest = left;
            }
            if (right < length && heap[right].priority < heap[smallest].priority) {
                smallest = right;
            }
            
            if (smallest == index) break;
            
            // 交换
            Item memory temp = heap[index];
            heap[index] = heap[smallest];
            heap[smallest] = temp;
            
            index = smallest;
        }
    }
}
```

### 1.3 链表

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 双向链表
 */
contract DoublyLinkedList {
    struct Node {
        uint256 value;
        uint256 next;
        uint256 prev;
        bool exists;
    }
    
    mapping(uint256 => Node) public nodes;
    uint256 public head;
    uint256 public tail;
    uint256 public length;
    uint256 private nextId = 1;
    
    // 在尾部添加
    function append(uint256 value) public returns (uint256) {
        uint256 id = nextId++;
        nodes[id] = Node(value, 0, tail, true);
        
        if (length == 0) {
            head = id;
            tail = id;
        } else {
            nodes[tail].next = id;
            tail = id;
        }
        
        length++;
        return id;
    }
    
    // 在头部添加
    function prepend(uint256 value) public returns (uint256) {
        uint256 id = nextId++;
        nodes[id] = Node(value, head, 0, true);
        
        if (length == 0) {
            head = id;
            tail = id;
        } else {
            nodes[head].prev = id;
            head = id;
        }
        
        length++;
        return id;
    }
    
    // 删除节点
    function remove(uint256 id) public returns (bool) {
        if (!nodes[id].exists) return false;
        
        Node memory node = nodes[id];
        
        if (node.prev != 0) {
            nodes[node.prev].next = node.next;
        } else {
            head = node.next;
        }
        
        if (node.next != 0) {
            nodes[node.next].prev = node.prev;
        } else {
            tail = node.prev;
        }
        
        delete nodes[id];
        length--;
        return true;
    }
    
    // 获取所有值
    function getAll() public view returns (uint256[] memory) {
        uint256[] memory values = new uint256[](length);
        uint256 current = head;
        uint256 index = 0;
        
        while (current != 0) {
            values[index] = nodes[current].value;
            current = nodes[current].next;
            index++;
        }
        
        return values;
    }
}
```

---

## ⚡ Part 2: Gas优化技巧 (1.5小时)

### 2.1 存储优化

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract StorageOptimization {
    // ❌ 低效：每个变量占用一个存储槽
    contract Inefficient {
        uint8 a;     // slot 0
        uint256 b;   // slot 1
        uint8 c;     // slot 2
        uint256 d;   // slot 3
    }
    
    // ✅ 高效：打包存储
    contract Efficient {
        uint8 a;     // slot 0 (byte 0)
        uint8 c;     // slot 0 (byte 1)
        uint256 b;   // slot 1
        uint256 d;   // slot 2
    }
    
    // 使用struct打包
    struct PackedData {
        uint64 timestamp;
        uint64 amount;
        uint128 value;
    }  // 共256位，占用1个存储槽
    
    // 合理使用mapping vs array
    mapping(address => uint256) public balances;  // 稀疏数据用mapping
    address[] public users;  // 需要遍历用array
    
    // 批量操作减少存储写入
    function batchUpdate(address[] memory addresses, uint256[] memory amounts) public {
        require(addresses.length == amounts.length, "Length mismatch");
        
        for (uint256 i = 0; i < addresses.length; i++) {
            balances[addresses[i]] = amounts[i];
        }
    }
}
```

### 2.2 计算优化

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract ComputationOptimization {
    // ✅ 使用unchecked节省gas（确保不会溢出时）
    function efficientLoop(uint256 n) public pure returns (uint256) {
        uint256 sum = 0;
        for (uint256 i = 0; i < n;) {
            sum += i;
            unchecked { i++; }  // 节省gas
        }
        return sum;
    }
    
    // ✅ 缓存数组长度
    function sumArray(uint256[] memory arr) public pure returns (uint256) {
        uint256 sum = 0;
        uint256 length = arr.length;  // 缓存长度
        
        for (uint256 i = 0; i < length;) {
            sum += arr[i];
            unchecked { i++; }
        }
        return sum;
    }
    
    // ✅ 短路求值
    function shortCircuit(uint256 a, uint256 b) public pure returns (bool) {
        // 便宜的检查放在前面
        return a > 0 && expensiveCheck(b);
    }
    
    function expensiveCheck(uint256 b) private pure returns (bool) {
        // 模拟昂贵的计算
        return b > 100;
    }
    
    // ✅ 使用常量
    uint256 public constant MAX_SUPPLY = 1000000;  // 不占存储
    
    // ❌ 避免
    // uint256 public maxSupply = 1000000;  // 占存储，消耗gas
}
```

### 2.3 函数优化

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract FunctionOptimization {
    uint256 public value;
    
    // ✅ external比public省gas（对于外部调用）
    function externalFunc(uint256[] calldata data) external pure returns (uint256) {
        return data[0];  // calldata不复制到内存
    }
    
    // ✅ 使用calldata代替memory（只读参数）
    function processData(uint256[] calldata data) external pure returns (uint256) {
        uint256 sum = 0;
        for (uint256 i = 0; i < data.length; i++) {
            sum += data[i];
        }
        return sum;
    }
    
    // ✅ 提前返回减少计算
    function earlyReturn(uint256 a) public pure returns (uint256) {
        if (a == 0) return 0;  // 提前返回
        
        // 昂贵的计算
        return a ** 2;
    }
    
    // ✅ 合并多个require
    function multipleChecks(uint256 a, uint256 b) public pure {
        require(a > 0 && b > 0, "Invalid input");  // 一次检查
        // vs
        // require(a > 0, "a must be positive");
        // require(b > 0, "b must be positive");
    }
}
```

### 2.4 事件vs存储

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract EventVsStorage {
    // ✅ 历史数据用事件（更便宜）
    event Transfer(address indexed from, address indexed to, uint256 amount);
    
    // ❌ 避免存储所有历史
    // Transfer[] public transfers;  // 昂贵
    
    mapping(address => uint256) public balances;
    
    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        balances[msg.sender] -= amount;
        balances[to] += amount;
        
        // 使用事件记录历史
        emit Transfer(msg.sender, to, amount);
    }
    
    // ✅ 只存储必要的状态
    uint256 public totalTransfers;  // 计数器
    
    function incrementTransfers() internal {
        unchecked {
            totalTransfers++;
        }
    }
}
```

---

## 🔄 Part 3: 代理模式和可升级合约 (1小时)

### 3.1 简单代理

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 简单代理合约
 * 使用delegatecall将调用转发到实现合约
 */
contract SimpleProxy {
    address public implementation;
    address public admin;
    
    constructor(address _implementation) {
        implementation = _implementation;
        admin = msg.sender;
    }
    
    // 升级实现
    function upgrade(address newImplementation) external {
        require(msg.sender == admin, "Not admin");
        implementation = newImplementation;
    }
    
    // fallback处理所有调用
    fallback() external payable {
        address _impl = implementation;
        assembly {
            // 复制calldata
            calldatacopy(0, 0, calldatasize())
            
            // delegatecall到实现合约
            let result := delegatecall(gas(), _impl, 0, calldatasize(), 0, 0)
            
            // 复制返回数据
            returndatacopy(0, 0, returndatasize())
            
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
    
    receive() external payable {}
}

/**
 * 实现合约V1
 */
contract ImplementationV1 {
    uint256 public value;
    
    function setValue(uint256 _value) public {
        value = _value;
    }
    
    function getValue() public view returns (uint256) {
        return value;
    }
}

/**
 * 实现合约V2（升级版）
 */
contract ImplementationV2 {
    uint256 public value;
    
    function setValue(uint256 _value) public {
        value = _value * 2;  // 新逻辑
    }
    
    function getValue() public view returns (uint256) {
        return value;
    }
    
    // 新功能
    function doubleValue() public view returns (uint256) {
        return value * 2;
    }
}
```

### 3.2 透明代理

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 透明代理（区分管理员和用户调用）
 */
contract TransparentProxy {
    bytes32 private constant IMPLEMENTATION_SLOT = 
        keccak256("proxy.implementation");
    bytes32 private constant ADMIN_SLOT = 
        keccak256("proxy.admin");
    
    constructor(address _implementation, address _admin) {
        _setImplementation(_implementation);
        _setAdmin(_admin);
    }
    
    modifier ifAdmin() {
        if (msg.sender == _getAdmin()) {
            _;
        } else {
            _fallback();
        }
    }
    
    function admin() external ifAdmin returns (address) {
        return _getAdmin();
    }
    
    function implementation() external ifAdmin returns (address) {
        return _getImplementation();
    }
    
    function changeAdmin(address newAdmin) external ifAdmin {
        require(newAdmin != address(0), "Invalid admin");
        _setAdmin(newAdmin);
    }
    
    function upgradeTo(address newImplementation) external ifAdmin {
        _setImplementation(newImplementation);
    }
    
    function _getAdmin() private view returns (address adm) {
        bytes32 slot = ADMIN_SLOT;
        assembly {
            adm := sload(slot)
        }
    }
    
    function _setAdmin(address newAdmin) private {
        bytes32 slot = ADMIN_SLOT;
        assembly {
            sstore(slot, newAdmin)
        }
    }
    
    function _getImplementation() private view returns (address impl) {
        bytes32 slot = IMPLEMENTATION_SLOT;
        assembly {
            impl := sload(slot)
        }
    }
    
    function _setImplementation(address newImplementation) private {
        bytes32 slot = IMPLEMENTATION_SLOT;
        assembly {
            sstore(slot, newImplementation)
        }
    }
    
    function _fallback() private {
        _delegate(_getImplementation());
    }
    
    function _delegate(address implementation) private {
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
    
    fallback() external payable {
        _fallback();
    }
    
    receive() external payable {
        _fallback();
    }
}
```

---

## 🎨 Part 4: 常用设计模式 (1小时)

### 4.1 工厂模式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Token {
    string public name;
    string public symbol;
    address public owner;
    
    constructor(string memory _name, string memory _symbol, address _owner) {
        name = _name;
        symbol = _symbol;
        owner = _owner;
    }
}

/**
 * 工厂合约
 */
contract TokenFactory {
    Token[] public tokens;
    mapping(address => Token[]) public userTokens;
    
    event TokenCreated(address indexed token, address indexed owner);
    
    function createToken(string memory name, string memory symbol) public returns (address) {
        Token newToken = new Token(name, symbol, msg.sender);
        
        tokens.push(newToken);
        userTokens[msg.sender].push(newToken);
        
        emit TokenCreated(address(newToken), msg.sender);
        return address(newToken);
    }
    
    function getTokenCount() public view returns (uint256) {
        return tokens.length;
    }
    
    function getUserTokens(address user) public view returns (Token[] memory) {
        return userTokens[user];
    }
}
```

### 4.2 拉取支付模式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 拉取支付（Pull Payment）
 * 让接收方主动提取，而不是推送支付
 */
contract PullPayment {
    mapping(address => uint256) public payments;
    
    event PaymentDeposited(address indexed payee, uint256 amount);
    event PaymentWithdrawn(address indexed payee, uint256 amount);
    
    // 存入支付
    function depositPayment(address payee) public payable {
        require(msg.value > 0, "No value sent");
        payments[payee] += msg.value;
        emit PaymentDeposited(payee, msg.value);
    }
    
    // 提取支付
    function withdrawPayment() public {
        uint256 payment = payments[msg.sender];
        require(payment > 0, "No payments");
        
        payments[msg.sender] = 0;  // 先清零防止重入
        
        (bool success, ) = msg.sender.call{value: payment}("");
        require(success, "Transfer failed");
        
        emit PaymentWithdrawn(msg.sender, payment);
    }
    
    // 查询可提取金额
    function getPayment(address payee) public view returns (uint256) {
        return payments[payee];
    }
}
```

### 4.3 限速模式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 限速模式（Rate Limiting）
 */
contract RateLimiter {
    uint256 public constant MAX_WITHDRAWALS_PER_DAY = 3;
    uint256 public constant MAX_AMOUNT_PER_DAY = 10 ether;
    
    struct WithdrawalInfo {
        uint256 lastDay;
        uint256 dailyCount;
        uint256 dailyAmount;
    }
    
    mapping(address => WithdrawalInfo) public withdrawalInfo;
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        uint256 currentDay = block.timestamp / 1 days;
        WithdrawalInfo storage info = withdrawalInfo[msg.sender];
        
        // 重置每日计数
        if (info.lastDay < currentDay) {
            info.lastDay = currentDay;
            info.dailyCount = 0;
            info.dailyAmount = 0;
        }
        
        // 检查限制
        require(info.dailyCount < MAX_WITHDRAWALS_PER_DAY, "Daily withdrawal limit reached");
        require(info.dailyAmount + amount <= MAX_AMOUNT_PER_DAY, "Daily amount limit exceeded");
        
        // 更新计数
        info.dailyCount++;
        info.dailyAmount += amount;
        
        // 执行提取
        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
}
```

### 4.4 状态机模式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 状态机模式
 */
contract StateMachine {
    enum State {
        Created,
        Locked,
        Inactive
    }
    
    State public state = State.Created;
    address public owner;
    
    event StateChanged(State from, State to);
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    modifier inState(State _state) {
        require(state == _state, "Invalid state");
        _;
    }
    
    function lock() public onlyOwner inState(State.Created) {
        _setState(State.Locked);
    }
    
    function unlock() public onlyOwner inState(State.Locked) {
        _setState(State.Created);
    }
    
    function deactivate() public onlyOwner {
        require(state != State.Inactive, "Already inactive");
        _setState(State.Inactive);
    }
    
    function _setState(State newState) private {
        State oldState = state;
        state = newState;
        emit StateChanged(oldState, newState);
    }
}
```

---

## 🛡️ Part 5: 安全编程实践 (1小时)

### 5.1 重入攻击防护

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 防止重入攻击
 */
contract ReentrancyGuard {
    uint256 private constant NOT_ENTERED = 1;
    uint256 private constant ENTERED = 2;
    uint256 private status = NOT_ENTERED;
    
    modifier nonReentrant() {
        require(status != ENTERED, "Reentrant call");
        status = ENTERED;
        _;
        status = NOT_ENTERED;
    }
}

contract SecureBank is ReentrancyGuard {
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    // ✅ 安全的提取（使用nonReentrant）
    function withdraw(uint256 amount) public nonReentrant {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        // Checks-Effects-Interactions模式
        // 1. Checks
        uint256 balance = balances[msg.sender];
        require(balance >= amount, "Insufficient balance");
        
        // 2. Effects
        balances[msg.sender] = 0;
        
        // 3. Interactions
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### 5.2 访问控制

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 基于角色的访问控制(RBAC)
 */
contract AccessControl {
    mapping(bytes32 => mapping(address => bool)) private roles;
    
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN");
    bytes32 public constant MINTER_ROLE = keccak256("MINTER");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER");
    
    event RoleGranted(bytes32 indexed role, address indexed account);
    event RoleRevoked(bytes32 indexed role, address indexed account);
    
    constructor() {
        _grantRole(ADMIN_ROLE, msg.sender);
    }
    
    modifier onlyRole(bytes32 role) {
        require(hasRole(role, msg.sender), "Access denied");
        _;
    }
    
    function hasRole(bytes32 role, address account) public view returns (bool) {
        return roles[role][account];
    }
    
    function grantRole(bytes32 role, address account) public onlyRole(ADMIN_ROLE) {
        _grantRole(role, account);
    }
    
    function revokeRole(bytes32 role, address account) public onlyRole(ADMIN_ROLE) {
        _revokeRole(role, account);
    }
    
    function _grantRole(bytes32 role, address account) private {
        if (!roles[role][account]) {
            roles[role][account] = true;
            emit RoleGranted(role, account);
        }
    }
    
    function _revokeRole(bytes32 role, address account) private {
        if (roles[role][account]) {
            roles[role][account] = false;
            emit RoleRevoked(role, account);
        }
    }
}
```

### 5.3 整数溢出防护

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SafeMathExample {
    // Solidity 0.8.0+ 自动检查溢出
    // 无需SafeMath库
    
    function safeAdd(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;  // 自动检查溢出
    }
    
    // 如果需要不检查溢出（明确知道不会溢出时）
    function uncheckedAdd(uint256 a, uint256 b) public pure returns (uint256) {
        unchecked {
            return a + b;  // 不检查溢出，节省gas
        }
    }
}
```

---

## 📝 今日作业

### 作业1: Gas优化挑战

```solidity
/**
 * 优化以下合约的gas消耗
 * 
 * 目标：将总gas消耗降低至少30%
 * 
 * 需要优化的合约：
 */
contract UnoptimizedContract {
    uint256 public value1;
    uint8 public value2;
    uint256 public value3;
    uint8 public value4;
    
    uint256[] public data;
    
    function addData(uint256 item) public {
        data.push(item);
    }
    
    function sumData() public view returns (uint256) {
        uint256 sum = 0;
        for (uint256 i = 0; i < data.length; i++) {
            sum = sum + data[i];
        }
        return sum;
    }
}

// TODO: 创建OptimizedContract并应用所有优化技巧
```

### 作业2: 可升级代币

```solidity
/**
 * 实现一个可升级的ERC20代币系统
 * 
 * 要求：
 * 1. 使用代理模式
 * 2. 实现V1版本（基础ERC20）
 * 3. 实现V2版本（添加暂停功能）
 * 4. 实现V3版本（添加税收功能）
 * 5. 确保升级后数据不丢失
 * 6. 添加管理员权限控制
 */
```

### 作业3: 设计模式应用

```solidity
/**
 * 设计一个众筹平台合约
 * 
 * 要求使用以下模式：
 * 1. 工厂模式：创建众筹项目
 * 2. 拉取支付：退款机制
 * 3. 状态机：项目状态管理
 * 4. 访问控制：角色权限
 * 5. 重入防护：安全提取
 * 
 * 功能：
 * - 创建众筹项目
 * - 用户投资
 * - 达标后提取资金
 * - 失败后退款
 * - 项目状态查询
 */
```

---

## ✅ 今日检查清单

- [ ] 理解高级数据结构实现
- [ ] 掌握Gas优化技巧
- [ ] 理解代理模式原理
- [ ] 掌握常用设计模式
- [ ] 了解安全编程最佳实践
- [ ] 完成所有作业

---

## 🆘 常见问题FAQ

### Q1: 如何测量Gas消耗？
```solidity
// 在Hardhat测试中
const tx = await contract.function();
const receipt = await tx.wait();
console.log("Gas used:", receipt.gasUsed.toString());

// 或使用hardhat-gas-reporter插件
```

### Q2: 代理模式的存储槽冲突如何避免？
```solidity
// 使用EIP-1967标准的存储槽
bytes32 private constant IMPLEMENTATION_SLOT = 
    bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1);
```

### Q3: 何时使用unchecked？
```solidity
// ✅ 安全使用unchecked的场景：
// 1. 循环计数器
for (uint i = 0; i < n;) {
    unchecked { i++; }  // i不会溢出
}

// 2. 已经检查过的计算
if (a >= b) {
    unchecked { c = a - b; }  // 不会下溢
}
```

---

## 📅 明日预告

明天学习智能合约安全：
- 常见漏洞分析
- 安全审计方法
- 防御技术
- Slither静态分析

**🎉 完成Day 4！继续加油！**
