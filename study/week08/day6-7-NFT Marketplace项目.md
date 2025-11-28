# Week 8 - Day 6-7: NFT Marketplace项目

**学习日期**: ___________
**预计用时**: 12-14小时  
**难度等级**: ⭐⭐⭐⭐⭐

## 📚 项目目标

构建一个完整的NFT市场 (OpenSea-lite)：
- ✅ ERC721A合约开发 (Gas优化)
- ✅ 市场交易合约 (List, Buy, Offer)
- ✅ 前端实现 (Next.js + Tailwind)
- ✅ 数据索引 (The Graph)
- ✅ IPFS存储集成
- ✅ 完整部署流程

---

## Part 1: 智能合约开发 (3小时)

### 1.1 NFT合约

```solidity
// contracts/CoolNFT.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "erc721a/contracts/ERC721A.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/common/ERC2981.sol";

contract CoolNFT is ERC721A, ERC2981, Ownable {
    string public baseURI;
    uint256 public constant MAX_SUPPLY = 10000;
    uint256 public constant PRICE = 0.05 ether;

    constructor() ERC721A("CoolNFT", "COOL") Ownable(msg.sender) {
        _setDefaultRoyalty(msg.sender, 500); // 5% royalty
    }

    function mint(uint256 quantity) external payable {
        require(totalSupply() + quantity <= MAX_SUPPLY, "Max supply exceeded");
        require(msg.value >= PRICE * quantity, "Insufficient funds");
        _mint(msg.sender, quantity);
    }

    function _startTokenId() internal view virtual override returns (uint256) {
        return 1;
    }

    function tokenURI(uint256 tokenId) public view virtual override returns (string memory) {
        require(_exists(tokenId), "URI query for nonexistent token");
        return string(abi.encodePacked(baseURI, _toString(tokenId), ".json"));
    }

    function setBaseURI(string calldata _newBaseURI) external onlyOwner {
        baseURI = _newBaseURI;
    }

    function withdraw() external onlyOwner {
        payable(msg.sender).transfer(address(this).balance);
    }

    function supportsInterface(bytes4 interfaceId)
        public
        view
        virtual
        override(ERC721A, ERC2981)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
```

### 1.2 市场合约

```solidity
// contracts/Marketplace.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/interfaces/IERC2981.sol";

contract Marketplace is ReentrancyGuard {
    struct Listing {
        address seller;
        uint256 price;
    }

    mapping(address => mapping(uint256 => Listing)) public listings;
    
    event ItemListed(address indexed seller, address indexed nftAddress, uint256 indexed tokenId, uint256 price);
    event ItemBought(address indexed buyer, address indexed nftAddress, uint256 indexed tokenId, uint256 price);
    event ItemCanceled(address indexed seller, address indexed nftAddress, uint256 indexed tokenId);

    function listItem(address nftAddress, uint256 tokenId, uint256 price) external nonReentrant {
        require(price > 0, "Price must be > 0");
        IERC721 nft = IERC721(nftAddress);
        require(nft.ownerOf(tokenId) == msg.sender, "Not owner");
        require(nft.isApprovedForAll(msg.sender, address(this)), "Not approved");

        listings[nftAddress][tokenId] = Listing(msg.sender, price);
        emit ItemListed(msg.sender, nftAddress, tokenId, price);
    }

    function buyItem(address nftAddress, uint256 tokenId) external payable nonReentrant {
        Listing memory listing = listings[nftAddress][tokenId];
        require(listing.price > 0, "Not listed");
        require(msg.value >= listing.price, "Insufficient funds");

        // 处理版税
        uint256 royaltyAmount = 0;
        address royaltyReceiver = address(0);

        try IERC2981(nftAddress).royaltyInfo(tokenId, listing.price) returns (address receiver, uint256 amount) {
            royaltyAmount = amount;
            royaltyReceiver = receiver;
        } catch {}

        if (royaltyAmount > 0 && royaltyReceiver != address(0)) {
            payable(royaltyReceiver).transfer(royaltyAmount);
        }

        // 支付卖家
        payable(listing.seller).transfer(listing.price - royaltyAmount);

        // 转移NFT
        IERC721(nftAddress).safeTransferFrom(listing.seller, msg.sender, tokenId);
        
        delete listings[nftAddress][tokenId];
        emit ItemBought(msg.sender, nftAddress, tokenId, listing.price);
    }

    function cancelListing(address nftAddress, uint256 tokenId) external nonReentrant {
        Listing memory listing = listings[nftAddress][tokenId];
        require(listing.seller == msg.sender, "Not seller");
        
        delete listings[nftAddress][tokenId];
        emit ItemCanceled(msg.sender, nftAddress, tokenId);
    }
}
```

---

## Part 2: Subgraph开发 (2小时)

### 2.1 Schema Definition

```graphql
# schema.graphql
type Listing @entity {
  id: ID!
  seller: Bytes!
  nftAddress: Bytes!
  tokenId: BigInt!
  price: BigInt!
  buyer: Bytes
  status: String! # Active, Sold, Canceled
  blockTimestamp: BigInt!
}

type User @entity {
  id: ID!
  listings: [Listing!]! @derivedFrom(field: "seller")
  purchases: [Listing!]! @derivedFrom(field: "buyer")
}
```

### 2.2 Mapping Logic

```typescript
// src/mapping.ts
import { ItemListed, ItemBought, ItemCanceled } from "../generated/Marketplace/Marketplace"
import { Listing, User } from "../generated/schema"

export function handleItemListed(event: ItemListed): void {
  let listing = new Listing(event.transaction.hash.toHex() + "-" + event.logIndex.toString())
  listing.seller = event.params.seller
  listing.nftAddress = event.params.nftAddress
  listing.tokenId = event.params.tokenId
  listing.price = event.params.price
  listing.status = "Active"
  listing.blockTimestamp = event.block.timestamp
  listing.save()

  let user = User.load(event.params.seller.toHex())
  if (!user) {
    user = new User(event.params.seller.toHex())
    user.save()
  }
}

export function handleItemBought(event: ItemBought): void {
  // Find active listing
  // Note: In production, you need a better ID strategy to find the specific listing
  // Here we simplify assuming we query active listings
}
```

---

## Part 3: 前端开发 (4小时)

### 3.1 项目设置

```bash
npx create-next-app@latest nft-marketplace
npm install ethers @web3modal/ethers @tanstack/react-query axios
```

### 3.2 首页：展示NFT列表

```javascript
// pages/index.js
import { useQuery } from '@apollo/client';
import { GET_ACTIVE_LISTINGS } from '../queries';
import NFTCard from '../components/NFTCard';

export default function Home() {
  const { loading, error, data } = useQuery(GET_ACTIVE_LISTINGS);

  if (loading) return <div className="text-center p-10">Loading...</div>;
  if (error) return <div className="text-center p-10 text-red-500">Error loading listings</div>;

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-8">Discover NFTs</h1>
      
      <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-6">
        {data.listings.map((listing) => (
          <NFTCard key={listing.id} listing={listing} />
        ))}
      </div>
    </div>
  );
}
```

### 3.3 NFT卡片组件

```javascript
// components/NFTCard.js
import { ethers } from 'ethers';
import { useNFTMetadata } from '../hooks/useNFTMetadata';

export default function NFTCard({ listing }) {
  const { metadata } = useNFTMetadata(listing.nftAddress, listing.tokenId);

  return (
    <div className="border rounded-xl overflow-hidden hover:shadow-lg transition-shadow">
      <div className="aspect-square bg-gray-100 relative">
        {metadata?.image ? (
          <img 
            src={metadata.image} 
            alt={metadata.name}
            className="object-cover w-full h-full"
          />
        ) : (
          <div className="flex items-center justify-center h-full text-gray-400">
            No Image
          </div>
        )}
      </div>
      
      <div className="p-4">
        <h3 className="font-bold text-lg truncate">{metadata?.name || `#${listing.tokenId}`}</h3>
        <p className="text-gray-500 text-sm mb-2">
          Price: {ethers.formatEther(listing.price)} ETH
        </p>
        
        <button 
          className="w-full bg-blue-600 text-white py-2 rounded-lg hover:bg-blue-700"
          onClick={() => {/* Handle Buy */}}
        >
          Buy Now
        </button>
      </div>
    </div>
  );
}
```

### 3.4 购买功能实现

```javascript
// hooks/useBuyNFT.js
import { useWeb3ModalProvider, useWeb3ModalAccount } from '@web3modal/ethers/react';
import { ethers } from 'ethers';
import MARKETPLACE_ABI from '../abis/Marketplace.json';

export function useBuyNFT() {
  const { walletProvider } = useWeb3ModalProvider();
  const { address } = useWeb3ModalAccount();

  const buyNFT = async (nftAddress, tokenId, price) => {
    if (!walletProvider) throw new Error("Wallet not connected");

    const ethersProvider = new ethers.BrowserProvider(walletProvider);
    const signer = await ethersProvider.getSigner();
    
    const contract = new ethers.Contract(
      process.env.NEXT_PUBLIC_MARKETPLACE_ADDRESS,
      MARKETPLACE_ABI,
      signer
    );

    const tx = await contract.buyItem(nftAddress, tokenId, {
      value: price
    });
    
    await tx.wait();
    return tx;
  };

  return { buyNFT };
}
```

---

## Part 4: 上架功能 (2小时)

### 4.1 上架模态框

```javascript
// components/ListModal.js
import { useState } from 'react';
import { ethers } from 'ethers';

export default function ListModal({ nftAddress, tokenId, isOpen, onClose }) {
  const [price, setPrice] = useState('');
  const [loading, setLoading] = useState(false);
  
  const handleList = async () => {
    setLoading(true);
    try {
      // 1. Approve
      const nftContract = new ethers.Contract(nftAddress, ERC721_ABI, signer);
      const tx1 = await nftContract.setApprovalForAll(MARKETPLACE_ADDRESS, true);
      await tx1.wait();
      
      // 2. List
      const marketContract = new ethers.Contract(MARKETPLACE_ADDRESS, MARKETPLACE_ABI, signer);
      const tx2 = await marketContract.listItem(
        nftAddress, 
        tokenId, 
        ethers.parseEther(price)
      );
      await tx2.wait();
      
      onClose();
    } catch (err) {
      console.error(err);
    } finally {
      setLoading(false);
    }
  };

  if (!isOpen) return null;

  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center">
      <div className="bg-white p-6 rounded-xl w-96">
        <h2 className="text-xl font-bold mb-4">List NFT</h2>
        <input
          type="number"
          placeholder="Price in ETH"
          className="w-full border p-2 rounded mb-4"
          value={price}
          onChange={e => setPrice(e.target.value)}
        />
        <button 
          onClick={handleList}
          disabled={loading}
          className="w-full bg-blue-600 text-white py-2 rounded"
        >
          {loading ? 'Processing...' : 'List Item'}
        </button>
      </div>
    </div>
  );
}
```

---

## Part 5: 部署与测试 (1.5小时)

### 5.1 部署脚本

```javascript
// scripts/deploy.js
const hre = require("hardhat");

async function main() {
  // 1. Deploy NFT
  const CoolNFT = await hre.ethers.getContractFactory("CoolNFT");
  const nft = await CoolNFT.deploy();
  await nft.waitForDeployment();
  console.log("NFT deployed to:", await nft.getAddress());

  // 2. Deploy Marketplace
  const Marketplace = await hre.ethers.getContractFactory("Marketplace");
  const market = await Marketplace.deploy();
  await market.waitForDeployment();
  console.log("Marketplace deployed to:", await market.getAddress());
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

---

## 📝 项目总结

### 完成功能
1. **合约层**: ERC721A优化、市场交易、版税分发
2. **数据层**: The Graph索引、IPFS存储
3. **前端层**: 浏览、购买、上架、钱包连接

### 扩展优化
- 实现拍卖功能
- 添加购物车批量购买
- 支持ERC1155多重代币
- 优化图片加载性能 (Next/Image)

---

## ✅ 检查清单

- [ ] 合约测试覆盖率 > 90%
- [ ] Subgraph正确索引事件
- [ ] 前端能够正确显示NFT元数据
- [ ] 购买流程顺畅，包含Approval处理
- [ ] 移动端适配良好

---

## 📅 下周预告

下周进入Week 9：跨链桥开发
- 跨链消息传递原理
- LayerZero集成
- 资产跨链锁定与铸造
- 跨链安全模型

**🎉 恭喜完成Week 8！你已经可以构建自己的OpenSea了！**
