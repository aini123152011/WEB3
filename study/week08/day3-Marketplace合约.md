# Week 8 - Day 3: Marketplace合约

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 学习目标

- ✅ 设计NFT市场架构
- ✅ 实现上架(Listing)与购买
- ✅ 实现报价(Offer)与接受
- ✅ 处理版税与平台费
- ✅ 掌握EIP-712离线签名

---

## Part 1: 市场合约架构设计 (1.5小时)

### 1.1 核心数据结构

```solidity
// NFTMarketplace.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract NFTMarketplace is Ownable, ReentrancyGuard {
    // 挂单结构体
    struct Listing {
        address seller;
        address nftContract;
        uint256 tokenId;
        uint256 price;
        bool isActive;
    }

    // 报价结构体
    struct Offer {
        address buyer;
        uint256 price;
        uint256 expiration;
    }

    // 状态变量
    mapping(address => mapping(uint256 => Listing)) public listings;
    mapping(address => mapping(uint256 => Offer[])) public offers;
    
    // 平台费率 (2.5% = 250)
    uint256 public platformFee = 250;
    address public feeRecipient;

    // 事件
    event ItemListed(address indexed seller, address indexed nftAddress, uint256 indexed tokenId, uint256 price);
    event ItemCanceled(address indexed seller, address indexed nftAddress, uint256 indexed tokenId);
    event ItemBought(address indexed buyer, address indexed nftAddress, uint256 indexed tokenId, uint256 price);
}
```

### 1.2 资金处理逻辑

- **Pull Pattern**: 尽量让用户提款，而不是自动转账
- **WETH**: 统一使用WETH作为支付代币（处理Offer时更方便）
- **Checks-Effects-Interactions**: 防止重入攻击

---

## Part 2: 核心交易功能 (2小时)

### 2.1 上架 (List Item)

```solidity
function listItem(
    address nftAddress,
    uint256 tokenId,
    uint256 price
) external nonReentrant {
    require(price > 0, "Price must be > 0");
    
    // 检查所有权和授权
    IERC721 nft = IERC721(nftAddress);
    require(nft.ownerOf(tokenId) == msg.sender, "Not the owner");
    require(
        nft.getApproved(tokenId) == address(this) || 
        nft.isApprovedForAll(msg.sender, address(this)),
        "Not approved"
    );

    // 创建挂单
    listings[nftAddress][tokenId] = Listing({
        seller: msg.sender,
        nftContract: nftAddress,
        tokenId: tokenId,
        price: price,
        isActive: true
    });

    emit ItemListed(msg.sender, nftAddress, tokenId, price);
}
```

### 2.2 购买 (Buy Item)

```solidity
function buyItem(address nftAddress, uint256 tokenId) 
    external 
    payable 
    nonReentrant 
{
    Listing storage listing = listings[nftAddress][tokenId];
    require(listing.isActive, "Item not listed");
    require(msg.value >= listing.price, "Insufficient funds");

    // 计算费用
    uint256 feeAmount = (listing.price * platformFee) / 10000;
    uint256 sellerAmount = listing.price - feeAmount;

    // 处理版税 (EIP-2981)
    (address royaltyReceiver, uint256 royaltyAmount) = _getRoyalty(nftAddress, tokenId, listing.price);
    if (royaltyAmount > 0) {
        sellerAmount -= royaltyAmount;
        payable(royaltyReceiver).transfer(royaltyAmount);
    }

    // 转移资金
    payable(feeRecipient).transfer(feeAmount);
    payable(listing.seller).transfer(sellerAmount);

    // 转移NFT
    IERC721(nftAddress).safeTransferFrom(listing.seller, msg.sender, tokenId);

    // 更新状态
    delete listings[nftAddress][tokenId];

    emit ItemBought(msg.sender, nftAddress, tokenId, listing.price);
}

function _getRoyalty(address token, uint256 tokenId, uint256 salePrice) 
    internal 
    view 
    returns (address receiver, uint256 amount) 
{
    // 检查是否支持IERC2981接口
    if (IERC165(token).supportsInterface(type(IERC2981).interfaceId)) {
        return IERC2981(token).royaltyInfo(tokenId, salePrice);
    }
    return (address(0), 0);
}
```

---

## Part 3: 报价系统 (Offer) (1.5小时)

报价系统允许买家对任何NFT出价，通常需要使用WETH（Wrapped ETH）。

### 3.1 创建报价

```solidity
// 需要引入WETH接口
interface IWETH {
    function deposit() external payable;
    function transfer(address to, uint256 value) external returns (bool);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
}

function makeOffer(
    address nftAddress,
    uint256 tokenId,
    uint256 price,
    uint256 duration
) external payable nonReentrant {
    require(price > 0, "Price must be > 0");
    require(msg.value == price, "Must send ETH for WETH");

    // 将ETH转换为WETH
    IWETH(WETH).deposit{value: msg.value}();
    
    // 存储报价
    offers[nftAddress][tokenId].push(Offer({
        buyer: msg.sender,
        price: price,
        expiration: block.timestamp + duration
    }));
}
```

### 3.2 接受报价

```solidity
function acceptOffer(
    address nftAddress,
    uint256 tokenId,
    uint256 offerIndex
) external nonReentrant {
    Offer memory offer = offers[nftAddress][tokenId][offerIndex];
    require(block.timestamp <= offer.expiration, "Offer expired");
    
    IERC721 nft = IERC721(nftAddress);
    require(nft.ownerOf(tokenId) == msg.sender, "Not the owner");

    // 转移NFT给买家
    nft.safeTransferFrom(msg.sender, offer.buyer, tokenId);

    // 转移WETH给卖家（扣除费用）
    uint256 fee = (offer.price * platformFee) / 10000;
    IWETH(WETH).transfer(feeRecipient, fee);
    IWETH(WETH).transfer(msg.sender, offer.price - fee);

    // 清理报价
    _removeOffer(nftAddress, tokenId, offerIndex);
}
```

---

## Part 4: 离线签名 (EIP-712) (1小时)

为了节省Gas，我们可以实现离线签名挂单（Seaport模式）。

### 4.1 EIP-712 结构

```solidity
bytes32 private constant LISTING_TYPEHASH = keccak256("Listing(address seller,address nftContract,uint256 tokenId,uint256 price,uint256 deadline)");

function buyItemWithSignature(
    Listing calldata listing,
    uint256 deadline,
    uint8 v, bytes32 r, bytes32 s
) external payable {
    require(block.timestamp <= deadline, "Expired");
    require(msg.value >= listing.price, "Insufficient funds");

    // 验证签名
    bytes32 digest = _hashTypedDataV4(keccak256(abi.encode(
        LISTING_TYPEHASH,
        listing.seller,
        listing.nftContract,
        listing.tokenId,
        listing.price,
        deadline
    )));
    
    address signer = ECDSA.recover(digest, v, r, s);
    require(signer == listing.seller, "Invalid signature");

    // 执行交易逻辑...
}
```

---

## 📝 今日作业

### 作业1: 完善Marketplace合约

实现以下功能：
1. `cancelListing`: 取消挂单
2. `updateListing`: 更新挂单价格
3. `cancelOffer`: 取消报价
4. 批量购买功能 (`buyBatch`)

### 作业2: 集成ERC2981

在Marketplace合约中：
1. 每次交易时检查NFT是否支持ERC2981
2. 自动计算并分发版税
3. 处理不支持ERC2981的旧NFT（可选：设置全局默认版税）

### 作业3: 编写测试脚本

使用Hardhat编写测试：
1. 模拟上架、购买流程
2. 测试版税分发是否准确
3. 测试平台费是否正确扣除
4. 测试报价过期逻辑

---

## ✅ 检查清单

- [ ] 理解Pull vs Push支付模式
- [ ] 掌握ReentrancyGuard的使用
- [ ] 理解WETH在报价中的作用
- [ ] 能够实现EIP-2981版税逻辑
- [ ] 理解离线签名的优势

---

## 📅 明日预告

明天学习拍卖系统：
- 英式拍卖 (English Auction)
- 荷兰式拍卖 (Dutch Auction)
- 竞价逻辑与时间延长
- 自动化结算是

**🎉 完成Day 3！你已经构建了DEX的核心引擎！**
