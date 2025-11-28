# Week 8 - Day 4: 拍卖与竞价系统

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 学习目标

- ✅ 实现英式拍卖 (English Auction)
- ✅ 实现荷兰式拍卖 (Dutch Auction)
- ✅ 处理出价与退款
- ✅ 解决拍卖中的安全问题
- ✅ 掌握自动化结算

---

## Part 1: 英式拍卖 (English Auction) (2小时)

英式拍卖是最常见的拍卖形式：起拍价低，出价逐渐升高，规定时间内最高出价者得。

### 1.1 合约结构

```solidity
// EnglishAuction.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract EnglishAuction is ReentrancyGuard {
    struct Auction {
        address seller;
        uint256 price; // 当前最高出价
        address highestBidder;
        uint256 endAt;
        bool started;
        bool ended;
    }

    IERC721 public nft;
    uint256 public nftId;
    
    Auction public auction;
    
    // 记录之前的出价（用于退款）
    mapping(address => uint256) public bids;

    event Start();
    event Bid(address indexed sender, uint256 amount);
    event Withdraw(address indexed bidder, uint256 amount);
    event End(address winner, uint256 amount);

    constructor(address _nft, uint256 _nftId, uint256 _startingBid) {
        nft = IERC721(_nft);
        nftId = _nftId;
        
        auction.seller = msg.sender;
        auction.price = _startingBid;
    }

    function start(uint256 duration) external {
        require(msg.sender == auction.seller, "Not seller");
        require(!auction.started, "Started");

        auction.started = true;
        auction.endAt = block.timestamp + duration;
        
        nft.transferFrom(msg.sender, address(this), nftId);
        emit Start();
    }
}
```

### 1.2 竞价逻辑

为了防止DoS攻击（恶意合约拒绝接收退款导致交易失败），采用 **Pull Pattern**（用户手动取回退款）。

```solidity
    function bid() external payable nonReentrant {
        require(auction.started, "Not started");
        require(block.timestamp < auction.endAt, "Ended");
        require(msg.value > auction.price, "Value < price");

        // 如果之前有出价者，记录其待退款金额
        if (auction.highestBidder != address(0)) {
            bids[auction.highestBidder] += auction.price;
        }

        auction.highestBidder = msg.sender;
        auction.price = msg.value;

        emit Bid(msg.sender, msg.value);
    }

    // 用户手动提取被超越的出价
    function withdraw() external nonReentrant {
        uint256 bal = bids[msg.sender];
        require(bal > 0, "No funds");
        
        bids[msg.sender] = 0;
        payable(msg.sender).transfer(bal);
        
        emit Withdraw(msg.sender, bal);
    }
```

### 1.3 结算逻辑

```solidity
    function end() external nonReentrant {
        require(auction.started, "Not started");
        require(block.timestamp >= auction.endAt, "Not ended");
        require(!auction.ended, "Ended");

        auction.ended = true;
        
        if (auction.highestBidder != address(0)) {
            nft.transferFrom(address(this), auction.highestBidder, nftId);
            payable(auction.seller).transfer(auction.price);
        } else {
            nft.transferFrom(address(this), auction.seller, nftId);
        }

        emit End(auction.highestBidder, auction.price);
    }
```

---

## Part 2: 荷兰式拍卖 (Dutch Auction) (1.5小时)

荷兰式拍卖：起拍价高，随着时间推移价格逐渐降低，直到有人购买或降到底价。

### 2.1 价格计算公式

$$ P(t) = P_{start} - \frac{(P_{start} - P_{end}) \times (t - t_{start})}{Duration} $$

### 2.2 合约实现

```solidity
// DutchAuction.sol
contract DutchAuction {
    uint256 private constant DURATION = 7 days;

    IERC721 public immutable nft;
    uint256 public immutable nftId;

    address public immutable seller;
    uint256 public immutable startingPrice;
    uint256 public immutable startAt;
    uint256 public immutable discountRate;

    constructor(
        uint256 _startingPrice,
        uint256 _discountRate,
        address _nft,
        uint256 _nftId
    ) {
        seller = payable(msg.sender);
        startingPrice = _startingPrice;
        discountRate = _discountRate;
        startAt = block.timestamp;
        
        nft = IERC721(_nft);
        nftId = _nftId;
        
        require(_startingPrice >= _discountRate * DURATION, "Starting price < min");
    }

    function getPrice() public view returns (uint256) {
        uint256 timeElapsed = block.timestamp - startAt;
        if (timeElapsed >= DURATION) {
            return startingPrice - (discountRate * DURATION);
        }
        return startingPrice - (discountRate * timeElapsed);
    }

    function buy() external payable {
        require(block.timestamp < startAt + DURATION, "Auction expired");

        uint256 price = getPrice();
        require(msg.value >= price, "ETH < price");

        nft.transferFrom(seller, msg.sender, nftId);
        
        // 退还多余的ETH
        uint256 refund = msg.value - price;
        if (refund > 0) {
            payable(msg.sender).transfer(refund);
        }
        
        // 卖家收款
        payable(seller).transfer(price);
    }
}
```

---

## Part 3: 拍卖优化与安全 (1.5小时)

### 3.1 延时机制 (Time Extension)

如果在拍卖结束前最后时刻出价，自动延长时间，防止"狙击"。

```solidity
uint256 public constant TIME_BUFFER = 15 minutes;

function bid() external payable {
    // ... basic checks ...
    
    // 如果在最后15分钟内出价，延长15分钟
    if (auction.endAt - block.timestamp < TIME_BUFFER) {
        auction.endAt = block.timestamp + TIME_BUFFER;
    }
    
    // ... update state ...
}
```

### 3.2 最小加价幅度

防止恶意用户以极小的加价（如 1 wei）无限延长时间。

```solidity
uint256 public constant MIN_INCREMENT = 5; // 5%

function bid() external payable {
    uint256 minBid = auction.price + ((auction.price * MIN_INCREMENT) / 100);
    require(msg.value >= minBid, "Bid too low");
    
    // ...
}
```

---

## Part 4: 集成到Marketplace (1小时)

将拍卖功能集成到我们在Day 3创建的Marketplace合约中。

### 4.1 统一存储

```solidity
contract Marketplace {
    enum ListingType { FixedPrice, Auction }

    struct Listing {
        ListingType listingType;
        uint256 price;      // 固定价格或起拍价
        uint256 endAt;      // 拍卖结束时间
        // ... other fields
    }
    
    function createAuction(
        address nft, 
        uint256 id, 
        uint256 startPrice, 
        uint256 duration
    ) external {
        // ...
    }
    
    function bid(address nft, uint256 id) external payable {
        // ...
    }
}
```

---

## 📝 今日作业

### 作业1: 实现英式拍卖合约

编写一个完整的英式拍卖合约：
1. 支持设置起拍价和保留价(Reserve Price)
2. 实现"最后时刻出价自动延时"功能
3. 实现流拍逻辑（未达到保留价）
4. 编写测试用例覆盖所有场景

### 作业2: 实现荷兰式拍卖

1. 编写合约支持荷兰式拍卖
2. 前端实现价格倒计时和动态价格显示
3. 测试价格随时间下降的准确性

### 作业3: 拍卖工厂模式

为了节省Gas，实现一个Factory合约：
1. 用户调用Factory部署自己的拍卖合约
2. 使用 `Clones` 库（EIP-1167）部署最小代理合约
3. Factory统一管理所有拍卖的状态

---

## ✅ 检查清单

- [ ] 理解英式 vs 荷兰式拍卖的区别
- [ ] 掌握 Pull Pattern 处理退款
- [ ] 理解拍卖延时机制的重要性
- [ ] 能够计算荷兰式拍卖的实时价格
- [ ] 了解如何防止拍卖攻击

---

## 📅 明日预告

明天学习版税与二级市场：
- EIP-2981 标准详解
- 平台费分发逻辑
- 链上与链下元数据同步
- 市场数据索引 (The Graph)

**🎉 完成Day 4！你现在是拍卖大师了！**
