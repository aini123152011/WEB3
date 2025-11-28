# Week 11 Day 3: 安全审计工具实战

**预计学习时间**: 6-8小时  
**难度**: ⭐⭐⭐⭐⭐

## 学习目标

通过本节课程,你将能够:
- 熟练使用 Slither 进行静态代码分析
- 使用 Mythril 进行符号执行分析
- 使用 Echidna 进行基于属性的模糊测试 (Property-based Fuzzing)
- 解读自动化审计工具生成的报告

## 课程内容

### 1. 静态分析工具: Slither

Slither 是最流行的 Solidity 静态分析工具，由 Trail of Bits 开发。它速度快，误报率相对较低。

#### 1.1 安装

Slither 是 Python 编写的。

```bash
pip3 install slither-analyzer
# 或者使用 docker
docker pull trailofbits/eth-security-toolbox
```

#### 1.2 基本使用

在 Hardhat 项目根目录下：

```bash
slither .
```

**常用命令**:
- `slither . --print human-summary`: 打印代码概览。
- `slither . --print contract-summary`: 打印合约摘要。
- `slither . --exclude-naming-convention`: 忽略命名规范警告（减少噪音）。

#### 1.3 分析报告解读

Slither 会将漏洞分为 High, Medium, Low, Informational 四个等级。

**示例输出**:
```
Reentrancy in VulnerableBank.withdraw() (VulnerableBank.sol#10-18):
    External calls:
    - (success) = msg.sender.call{value: balance}("") (VulnerableBank.sol#14)
    State variables written after the call(s):
    - balances[msg.sender] = 0 (VulnerableBank.sol#17)
Reference: https://github.com/crytic/slither/wiki/Detector-Documentation#reentrancy-vulnerabilities
```

### 2. 符号执行工具: Mythril

Mythril 是 ConsenSys 开发的 EVM 字节码安全分析工具。它使用符号执行来探索可能导致漏洞的程序路径。

#### 2.1 安装

```bash
pip3 install mythril
```

#### 2.2 基本使用

Mythril 分析速度较慢，适合深度检查。

```bash
# 分析单个 Solidity 文件
myth analyze contracts/VulnerableBank.sol --solc-json mythril.config.json

# 分析指定的合约地址 (需要 Infura ID)
myth analyze -a 0x... --rpc infura-mainnet
```

**参数**:
- `-t <transaction_count>`: 限制探索深度。
- `-m <modules>`: 仅运行特定检测模块 (如 `suicide`, `ether_thief`)。

### 3. 模糊测试工具: Echidna

Echidna 是 Trail of Bits 开发的 Haskell 编写的高级模糊测试器。它通过随机生成大量交易来试图破坏用户定义的"不变量" (Invariants)。

#### 3.1 安装

推荐使用 Docker：
```bash
docker pull trailofbits/eth-security-toolbox
```

或者直接下载二进制文件。

#### 3.2 编写测试属性

Echidna 需要你编写一个继承自被测合约的 Test 合约，并定义返回 `bool` 的属性函数。如果函数返回 `false`，则说明找到了漏洞。

**示例**: 测试一个 Token 合约的总供应量是否永远不变。

```solidity
// TestToken.sol
import "./MyToken.sol";

contract TestToken is MyToken {
    constructor() {
        _mint(msg.sender, 1000);
    }

    // 属性: 用户余额总和应该等于总供应量?
    // 或者: 任何用户的余额不应超过总供应量
    function echidna_balance_under_total_supply() public view returns (bool) {
        return balanceOf(msg.sender) <= totalSupply();
    }
}
```

#### 3.3 运行测试

```bash
echidna-test contracts/TestToken.sol --contract TestToken
```

Echidna 会自动生成各种 `msg.sender`, `amount` 等参数调用合约函数，试图让 `echidna_balance_under_total_supply` 返回 `false`。

### 4. 工具对比与工作流

| 工具 | 类型 | 优点 | 缺点 | 推荐场景 |
| :--- | :--- | :--- | :--- | :--- |
| **Slither** | 静态分析 | 极快，易于集成 CI，无需配置 | 只能发现模式匹配的漏洞，无法发现逻辑漏洞 | 每次提交代码时运行 |
| **Mythril** | 符号执行 | 不需要写测试代码，能发现深层路径 | 极慢，容易状态爆炸，可能有误报 | 重大版本发布前运行 |
| **Echidna** | 模糊测试 | 能发现复杂的逻辑漏洞，不仅限于已知漏洞模式 | 需要编写专门的测试代码，学习曲线陡峭 | 核心业务逻辑验证 |

**推荐审计工作流**:
1.  **开发阶段**: Hardhat/Foundry 单元测试 + Slither 本地检查。
2.  **CI 阶段**: GitHub Actions 自动运行 Slither 和 Solhint。
3.  **预发布阶段**: 编写 Echidna 不变量测试，运行 24 小时以上。
4.  **审计阶段**: 运行 Mythril，人工复核所有报告。

## 实践练习

### 练习1: 使用 Slither 发现漏洞

1.  准备一个包含重入漏洞和整数溢出的合约。
2.  运行 `slither .`。
3.  解读输出报告，定位代码行号。
4.  修复漏洞，再次运行 Slither，确保 Clean。

### 练习2: Echidna 实战

编写一个简单的 `Bank` 合约，包含一个逻辑漏洞：允许用户提款超过其存款（如果某种特殊条件满足）。

```solidity
contract Bank {
    mapping(address => uint) public balances;
    bool public emergencyMode;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function setEmergency(bool _mode) public {
        emergencyMode = _mode;
    }

    function withdraw(uint amount) public {
        if (emergencyMode) {
            // 漏洞: 紧急模式下忘记扣除余额
            payable(msg.sender).transfer(amount);
            return; 
        }
        require(balances[msg.sender] >= amount);
        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
}
```

**任务**:
1.  编写 Echidna 测试属性 `echidna_solvent()`: 检查合约的 ETH 余额是否始终大于等于所有用户的账面余额之和（简化版：检查合约余额不小于 0）。
2.  运行 Echidna，看它是否能发现开启 `emergencyMode` 后可以无限提款的漏洞。

### 练习3: 使用 Docker 运行 eth-security-toolbox

1.  启动容器: `docker run -it -v $(pwd):/share trailofbits/eth-security-toolbox`。
2.  在容器内使用 `slither`, `echidna-test` 等工具分析挂载的 `/share` 目录下的代码。

## 学习检查清单

- [ ] 能够在本地环境安装并运行 Slither
- [ ] 理解静态分析与动态分析(Fuzzing/Symbolic Execution)的区别
- [ ] 能够编写简单的 Echidna Property 测试
- [ ] 知道如何在 CI/CD 中集成这些工具
- [ ] 能够根据工具报告过滤误报并修复真漏洞

## 扩展阅读

- [Slither GitHub](https://github.com/crytic/slither)
- [Echidna Tutorial](https://github.com/crytic/building-secure-contracts/tree/master/program-analysis/echidna)
- [Mythril Classic](https://github.com/ConsenSys/mythril)
- [Trail of Bits Publications](https://github.com/trailofbits/publications)

## 下节预告

下一节我们将学习**人工审计方法论**,包括:
- 审计流程标准 (Pre-audit, Audit, Report)
- 审计检查清单 (Checklist)
- 常见业务逻辑漏洞分析
- 如何撰写专业的审计报告
