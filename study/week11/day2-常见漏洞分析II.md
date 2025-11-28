# Week 11 Day 2: 常见智能合约漏洞分析 II

**预计学习时间**: 6-8小时  
**难度**: ⭐⭐⭐⭐

## 学习目标

通过本节课程,你将能够:
- 深入理解抢跑交易 (Front-running) 原理及防御策略
- 识别拒绝服务 (DoS) 攻击向量
- 掌握区块链随机数的正确生成方式 (Chainlink VRF)
- 理解签名重放攻击与 EIP-712 标准
- 识别钓鱼攻击 (Phishing) 的新型手法

## 课程内容

### 1. 抢跑交易 (Front-running)

也称为交易顺序依赖 (Transaction Ordering Dependence, TOD)。

#### 1.1 漏洞原理

矿工（或验证者）决定区块内交易的排序。通常按 Gas Price 排序。攻击者监听 Mempool，发现有利可图的交易（如大额买入会拉高价格），然后以更高的 Gas Price 发送自己的交易，抢在受害者之前成交。

**经典案例**: Uniswap 三明治攻击 (Sandwich Attack)。
1.  **Front-run**: 攻击者买入 Token (价格拉高)。
2.  **Victim**: 受害者大额买入 (价格进一步拉高)。
3.  **Back-run**: 攻击者卖出 Token (获利离场)。

**漏洞代码示例 (猜数字游戏)**:
```solidity
uint256 public answerHash;
function solve(uint256 solution) public {
    if (keccak256(abi.encodePacked(solution)) == answerHash) {
        payable(msg.sender).transfer(address(this).balance);
    }
}
```
攻击者看到某人提交了正确答案的交易，立马复制该答案并以更高 Gas 提交，抢走奖励。

#### 1.2 防御措施

1.  **Commit-Reveal 方案**:
    -   阶段 1: 提交答案的哈希 (Hash(salt + answer))。
    -   阶段 2: 提交明文答案和盐值。
2.  **限制滑点 (Slippage)**: DEX 交易必须设置最小获得量。
3.  **Flashbots**: 使用私有交易池，直接发送给矿工，不经过公开 Mempool。

### 2. 拒绝服务攻击 (Denial of Service)

#### 2.1 外部调用导致的 DoS

**漏洞**: 依赖外部调用的成功来执行后续逻辑。
```solidity
// 拍卖合约: 退还前一个最高出价者的钱
function bid() public payable {
    require(msg.value > highestBid);
    // 危险: 如果 highestBidder 是一个拒绝接收 ETH 的合约，
    // 这里的 transfer 会失败，导致整个 bid 函数 revert。
    // 永远没有人能出新价。
    payable(highestBidder).transfer(highestBid); 
    
    highestBidder = msg.sender;
    highestBid = msg.value;
}
```

**防御**: 使用 **Pull over Push** 模式。
让用户自己来取款 (`withdraw`)，而不是合约主动发送。

#### 2.2 Gas Limit DoS (Block Stuffing)

**漏洞**: 遍历一个无限制增长的数组。
```solidity
address[] public users;
function payAll() public {
    // 如果 users 数组太大，循环消耗的 Gas 会超过区块 Gas Limit，导致交易永远无法成功
    for (uint i = 0; i < users.length; i++) {
        payable(users[i]).transfer(1 ether);
    }
}
```

**防御**: 限制数组长度，或使用分批处理。

### 3. 随机数漏洞 (Bad Randomness)

#### 3.1 链上伪随机

区块链是确定性的系统，没有真正的随机数。

**漏洞**: 使用区块属性作为随机源。
```solidity
function random() public view returns (uint) {
    // 矿工可以控制 timestamp, difficulty 等，从而操纵结果
    return uint(keccak256(abi.encodePacked(block.difficulty, block.timestamp, users))); 
}
```

#### 3.2 防御措施

1.  **Chainlink VRF (Verifiable Random Function)**:
    预言机生成随机数及证明，链上验证证明。这是目前最标准的做法。

2.  **Commit-Reveal**:
    如前所述，用于简单的博弈场景。

### 4. 签名重放 (Signature Replay)

#### 4.1 漏洞原理

用户签名的消息如果没有包含足够的信息（如 Nonce, ChainID），可能被重复使用。

**场景**:
Alice 签名授权 Bob 提款 100 Token。Bob 拿着签名提款成功。
Bob 再次拿着同样的签名去调用合约，如果合约没检查 Nonce，Bob 又提走 100 Token。

#### 4.2 防御措施

1.  **Nonce**: 每个用户维护一个 nonce，签名必须包含当前 nonce，使用后 nonce++。
2.  **ChainID**: 包含 ChainID 防止跨链重放 (ETH 分叉链攻击)。
3.  **EIP-712**: 标准化结构化数据签名，包含 Domain Separator (含 ChainID, Contract Address)。

```solidity
// EIP-712 示例
bytes32 public constant PERMIT_TYPEHASH = 
    keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");

function permit(...) public {
    bytes32 structHash = keccak256(abi.encode(
        PERMIT_TYPEHASH,
        owner,
        spender,
        value,
        nonces[owner]++, // 检查并递增 nonce
        deadline
    ));
    // ... 验证签名
}
```

### 5. 钓鱼攻击与社会工程学

#### 5.1 函数签名碰撞 (Zero-width / Collision)

攻击者生成一个函数签名与 `transfer` 相同的恶意函数，诱导用户调用。虽然 Solidity 编译器会检查，但在某些代理模式下可能发生。

#### 5.2 地址投毒 (Address Poisoning)

攻击者生成一个与受害者常用地址首尾相同（如 `0x123...abc`）的地址。
攻击者向受害者转账 0 元。
受害者在历史记录中复制地址时，如果不仔细检查中间字符，可能转账给攻击者。

**防御**: 总是检查完整地址，或使用 ENS / 地址白名单。

## 实践练习

### 练习1: 修复 DoS 漏洞

1.  部署 `BadAuction` 合约（Push 模式）。
2.  编写一个恶意合约 `Hacker`，其 `fallback` 函数直接 revert。
3.  使用 `Hacker` 参与竞拍，成为最高出价者。
4.  尝试用另一个账号出更高价，观察交易失败。
5.  重写 Auction 合约，使用 Pull 模式修复漏洞。

### 练习2: 抢跑攻击演示

1.  编写一个基于 Hash 的猜谜合约。
2.  在 Remix 中模拟两笔交易：
    -   Account A 提交正确答案 (Gas Price = 10)。
    -   Account B 监听并提交相同答案 (Gas Price = 20)。
    -   观察 Account B 先成交。

### 练习3: 实现 EIP-712 签名验证

1.  使用 `ethers.js` 在前端生成 EIP-712 签名。
2.  编写 Solidity 合约验证该签名。
3.  测试修改 ChainID 或 Contract Address 后验证失败。

## 学习检查清单

- [ ] 理解为什么链上没有真随机数
- [ ] 掌握 Chainlink VRF 的基本集成流程
- [ ] 能够识别 Push 模式导致的 DoS 风险
- [ ] 理解 Flashbots 如何解决抢跑问题
- [ ] 掌握 EIP-712 签名的构造与验证
- [ ] 了解地址投毒的原理

## 扩展阅读

-   [Chainlink VRF Documentation](https://docs.chain.link/vrf/v2/introduction)
-   [EIP-712 Standard](https://eips.ethereum.org/EIPS/eip-712)
-   [Flashbots Docs](https://docs.flashbots.net/)
-   [Ethereum Smart Contract Security Best Practices](https://consensys.github.io/smart-contract-best-practices/)

## 下节预告

下一节我们将学习**安全审计工具实战**,包括:
- Slither 静态分析
- Mythril 符号执行
- Echidna 模糊测试 (Fuzzing)
- Manticore 自动化分析
