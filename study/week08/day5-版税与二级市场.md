# Week 8 - Day 5: 版税与二级市场

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 深入理解EIP-2981版税标准
- ✅ 实现分级版税分发
- ✅ 学习市场数据索引
- ✅ 掌握The Graph子图开发
- ✅ 处理链上链下数据同步

---

## Part 1: 复杂版税逻辑 (1.5小时)

### 1.1 EIP-2981 进阶

不仅是单一接收者，我们可能需要将版税分发给多个创作者。

```solidity
// SplitRoyalty.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/common/ERC2981.sol";
import "@openzeppelin/contracts/finance/PaymentSplitter.sol";

contract SplitRoyalty is ERC2981, PaymentSplitter {
    constructor(
        address[] memory payees,
        uint256[] memory shares
    ) PaymentSplitter(payees, shares) {
        // 设置版税接收者为合约本身（PaymentSplitter）
        // 版税比例 10%
        _setDefaultRoyalty(address(this), 1000);
    }

    // 允许合约接收ETH
    receive() external payable virtual override {
        super.receive();
    }
}
```

### 1.2 动态版税

有些项目希望根据销量或时间调整版税。

```solidity
contract DynamicRoyalty is ERC2981, Ownable {
    uint256 public totalSupply;
    
    function royaltyInfo(uint256 tokenId, uint256 salePrice)
        public
        view
        override
        returns (address receiver, uint256 royaltyAmount)
    {
        uint96 feeNumerator = 500; // Default 5%
        
        // 如果销量超过1000，版税降为2.5%
        if (totalSupply > 1000) {
            feeNumerator = 250;
        }
        
        uint256 amount = (salePrice * feeNumerator) / _feeDenominator();
        return (owner(), amount);
    }
}
```

---

## Part 2: 市场数据索引 (The Graph) (2.5小时)

在前端直接查询合约事件非常慢，我们需要使用The Graph来索引数据。

### 2.1 创建Subgraph

```bash
# 安装 Graph CLI
npm install -g @graphprotocol/graph-cli

# 初始化项目
graph init --studio my-nft-marketplace
```

### 2.2 定义Schema (schema.graphql)

```graphql
type Listing @entity {
  id: ID!
  seller: Bytes!
  nftContract: Bytes!
  tokenId: BigInt!
  price: BigInt!
  status: ListingStatus!
  createdAt: BigInt!
  buyer: Bytes
}

enum ListingStatus {
  Active
  Sold
  Canceled
}

type User @entity {
  id: ID!
  listings: [Listing!]! @derivedFrom(field: "seller")
  purchases: [Listing!]! @derivedFrom(field: "buyer")
}
```

### 2.3 编写映射 (mapping.ts)

```typescript
import { BigInt } from "@graphprotocol/graph-ts"
import {
  ItemListed,
  ItemBought,
  ItemCanceled
} from "../generated/Marketplace/Marketplace"
import { Listing, User } from "../generated/schema"

export function handleItemListed(event: ItemListed): void {
  let listing = new Listing(event.params.listingId.toString())
  
  listing.seller = event.params.seller
  listing.nftContract = event.params.nftAddress
  listing.tokenId = event.params.tokenId
  listing.price = event.params.price
  listing.status = "Active"
  listing.createdAt = event.block.timestamp
  
  listing.save()
  
  let user = User.load(event.params.seller.toHex())
  if (!user) {
    user = new User(event.params.seller.toHex())
    user.save()
  }
}

export function handleItemBought(event: ItemBought): void {
  let listing = Listing.load(event.params.listingId.toString())
  if (listing) {
    listing.status = "Sold"
    listing.buyer = event.params.buyer
    listing.save()
  }
}
```

### 2.4 部署Subgraph

```bash
# 认证
graph auth --studio <DEPLOY_KEY>

# 编译
graph codegen && graph build

# 部署
graph deploy --studio my-nft-marketplace
```

---

## Part 3: 前端集成The Graph (1.5小时)

### 3.1 Apollo Client配置

```javascript
// config/apollo.js
import { ApolloClient, InMemoryCache } from '@apollo/client';

export const client = new ApolloClient({
  uri: 'https://api.studio.thegraph.com/query/YOUR_ID/my-nft-marketplace/v0.0.1',
  cache: new InMemoryCache(),
});
```

### 3.2 查询数据

```javascript
// hooks/useListings.js
import { useQuery, gql } from '@apollo/client';

const GET_ACTIVE_LISTINGS = gql`
  query GetActiveListings {
    listings(where: { status: Active }, orderBy: createdAt, orderDirection: desc) {
      id
      seller
      nftContract
      tokenId
      price
    }
  }
`;

export function useActiveListings() {
  const { loading, error, data } = useQuery(GET_ACTIVE_LISTINGS);
  return { loading, error, listings: data?.listings || [] };
}
```

### 3.3 用户个人中心数据

```javascript
const GET_USER_DATA = gql`
  query GetUserData($user: Bytes!) {
    user(id: $user) {
      listings {
        id
        price
        status
      }
      purchases {
        id
        price
        nftContract
        tokenId
      }
    }
  }
`;
```

---

## Part 4: 链下元数据同步 (1小时)

如何高效地获取NFT的图片和属性？

### 4.1 元数据聚合器

创建一个后端服务或使用Alchemy/Moralis API来聚合元数据。

```javascript
// 使用Alchemy NFT API
import { Alchemy, Network } from "alchemy-sdk";

const config = {
  apiKey: "YOUR_API_KEY",
  network: Network.ETH_MAINNET,
};
const alchemy = new Alchemy(config);

export const getNFTMetadata = async (address, tokenId) => {
  const response = await alchemy.nft.getNftMetadata(address, tokenId);
  return {
    title: response.title,
    description: response.description,
    image: response.media[0].gateway,
    attributes: response.rawMetadata.attributes
  };
};
```

---

## 📝 今日作业

### 作业1: 实现PaymentSplitter版税

编写一个合约：
1. 继承自 `ERC2981` 和 `PaymentSplitter`
2. 在构造函数中设置多个收款人（如：艺术家、开发者、慈善机构）
3. 编写测试验证版税收入是否能正确分发

### 作业2: 开发并部署Subgraph

1. 为你的Marketplace合约创建一个Subgraph
2. 索引所有 `Listed`, `Bought`, `Canceled` 事件
3. 添加 `Volume` 统计实体，记录每日交易量
4. 部署到 The Graph Studio

### 作业3: 前端展示层

1. 使用 Apollo Client 获取 Subgraph 数据
2. 展示"最新上架"和"最近成交"列表
3. 实现按价格排序和按属性筛选功能

---

## ✅ 检查清单

- [ ] 理解PaymentSplitter的工作原理
- [ ] 掌握Subgraph Schema的设计
- [ ] 能够编写Subgraph Mapping逻辑
- [ ] 熟练使用GraphQL查询数据
- [ ] 理解链上链下数据的整合方式

---

## 📅 周末预告

周末进行NFT Marketplace综合项目：
- 整合合约、前端和Subgraph
- 实现完整的NFT铸造、上架、交易流程
- 部署到测试网并开源

**🎉 完成Day 5！你已经掌握了构建高性能DApp的关键技术！**
