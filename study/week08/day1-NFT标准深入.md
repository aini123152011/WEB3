# Week 8 - Day 1: NFT标准深入

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐

## 📚 学习目标

- ✅ 深入理解ERC721标准
- ✅ 掌握ERC1155标准
- ✅ 学习NFT元数据规范
- ✅ 实现NFT白名单铸造
- ✅ 理解版税标准 (EIP-2981)

---

## Part 1: ERC721标准深入 (1.5小时)

### 1.1 核心接口回顾

ERC721是不可替代代币(Non-Fungible Token)的标准接口。

```solidity
// IERC721.sol
interface IERC721 {
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

    function balanceOf(address owner) external view returns (uint256 balance);
    function ownerOf(uint256 tokenId) external view returns (address owner);
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function transferFrom(address from, address to, uint256 tokenId) external;
    function approve(address to, uint256 tokenId) external;
    function getApproved(uint256 tokenId) external view returns (address operator);
    function setApprovalForAll(address operator, bool _approved) external;
    function isApprovedForAll(address owner, address operator) external view returns (bool);
    function safeTransferFrom(address from, address to, uint256 tokenId, bytes calldata data) external;
}
```

### 1.2 ERC721A优化

ERC721A是由Azuki团队开发的一个gas优化版本，特别适合批量铸造。

**主要优化点**：
1. 移除重复的存储写入
2. 批量铸造时更新所有者逻辑优化
3. 只有在转账时才初始化所有权数据

```solidity
// ERC721A.sol (简化版)
contract ERC721A is IERC721A {
    // 拥有者数据结构
    struct TokenOwnership {
        address addr;
        uint64 startTimestamp;
        bool burned;
    }

    // 批量铸造
    function _mint(address to, uint256 quantity) internal virtual {
        uint256 startTokenId = _currentIndex;
        
        // 只需要写入一次所有权数据
        _ownerships[startTokenId] = TokenOwnership(to, uint64(block.timestamp), false);
        _currentIndex += quantity;
        
        // 发送Transfer事件
        for (uint256 i = 0; i < quantity; i++) {
            emit Transfer(address(0), to, startTokenId + i);
        }
    }
}
```

### 1.3 实践：使用ERC721A

```solidity
// MyNFT.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "erc721a/contracts/ERC721A.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

contract MyNFT is ERC721A, Ownable {
    using Strings for uint256;

    uint256 public constant MAX_SUPPLY = 10000;
    uint256 public constant MINT_PRICE = 0.05 ether;
    uint256 public constant MAX_PER_TX = 5;

    string private baseURI;

    constructor() ERC721A("MyNFT", "MNFT") Ownable(msg.sender) {}

    function mint(uint256 quantity) external payable {
        require(totalSupply() + quantity <= MAX_SUPPLY, "Max supply exceeded");
        require(quantity <= MAX_PER_TX, "Max per tx exceeded");
        require(msg.value >= MINT_PRICE * quantity, "Insufficient funds");

        _mint(msg.sender, quantity);
    }

    function _startTokenId() internal view virtual override returns (uint256) {
        return 1;
    }

    function tokenURI(uint256 tokenId) public view virtual override returns (string memory) {
        require(_exists(tokenId), "URI query for nonexistent token");
        return string(abi.encodePacked(baseURI, tokenId.toString(), ".json"));
    }

    function setBaseURI(string calldata _newBaseURI) external onlyOwner {
        baseURI = _newBaseURI;
    }

    function withdraw() external onlyOwner {
        payable(msg.sender).transfer(address(this).balance);
    }
}
```

---

## Part 2: ERC1155标准 (1.5小时)

ERC1155是一个多代币标准，允许在一个合约中管理多种类型的代币（同质化、非同质化、半同质化）。

### 2.1 核心差异

- **批量操作**：支持 `safeBatchTransferFrom`
- **ID管理**：每个Token ID代表一种代币类型
- **Gas效率**：大大降低了部署和交易成本

### 2.2 游戏道具合约示例

```solidity
// GameItems.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract GameItems is ERC1155, Ownable {
    uint256 public constant GOLD = 0;
    uint256 public constant SILVER = 1;
    uint256 public constant SWORD = 2;
    uint256 public constant SHIELD = 3;

    constructor() ERC1155("https://game.example/api/item/{id}.json") Ownable(msg.sender) {
        _mint(msg.sender, GOLD, 10**18, "");
        _mint(msg.sender, SILVER, 10**18, "");
        _mint(msg.sender, SWORD, 1000, "");
        _mint(msg.sender, SHIELD, 1000, "");
    }

    function mint(address account, uint256 id, uint256 amount, bytes memory data)
        public
        onlyOwner
    {
        _mint(account, id, amount, data);
    }

    function mintBatch(address to, uint256[] memory ids, uint256[] memory amounts, bytes memory data)
        public
        onlyOwner
    {
        _mintBatch(to, ids, amounts, data);
    }
    
    // 锻造系统：消耗资源生成新物品
    function forgeSword(address account) public {
        _burn(account, GOLD, 100);
        _burn(account, SILVER, 50);
        _mint(account, SWORD, 1, "");
    }
}
```

---

## Part 3: 白名单机制 (Merkle Tree) (1.5小时)

为了节省Gas，我们使用Merkle Tree来实现白名单验证，而不是将所有地址存储在合约中。

### 3.1 生成Merkle Tree (JavaScript)

```javascript
const { MerkleTree } = require('merkletreejs');
const keccak256 = require('keccak256');

// 白名单地址列表
const whitelistAddresses = [
  "0x5B38Da6a701c568545dCfcB03FcB875f56beddC4",
  "0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2",
  // ...
];

// 1. 哈希化地址
const leafNodes = whitelistAddresses.map(addr => keccak256(addr));

// 2. 创建树
const merkleTree = new MerkleTree(leafNodes, keccak256, { sortPairs: true });

// 3. 获取根哈希
const rootHash = merkleTree.getHexRoot();
console.log('Root Hash:', rootHash);

// 4. 生成证明（用于前端调用）
const claimingAddress = leafNodes[0];
const hexProof = merkleTree.getHexProof(claimingAddress);
console.log('Proof:', hexProof);
```

### 3.2 合约验证

```solidity
// WhitelistNFT.sol
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";

contract WhitelistNFT is ERC721A, Ownable {
    bytes32 public merkleRoot;

    function setMerkleRoot(bytes32 _merkleRoot) external onlyOwner {
        merkleRoot = _merkleRoot;
    }

    function whitelistMint(bytes32[] calldata _merkleProof, uint256 quantity) external payable {
        // 1. 验证不在白名单中的用户无法铸造
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender));
        require(MerkleProof.verify(_merkleProof, merkleRoot, leaf), "Invalid proof");
        
        // 2. 执行铸造逻辑
        _mint(msg.sender, quantity);
    }
}
```

---

## Part 4: 版税标准 (EIP-2981) (1小时)

NFT版税标准允许创作者在二级市场销售中获得持续收益。

### 4.1 实现ERC2981

```solidity
// RoyaltyNFT.sol
import "@openzeppelin/contracts/token/common/ERC2981.sol";

contract RoyaltyNFT is ERC721A, ERC2981, Ownable {
    constructor() ERC721A("RoyaltyNFT", "RNFT") Ownable(msg.sender) {
        // 设置默认版税：接收者为部署者，比例为5% (500/10000)
        _setDefaultRoyalty(msg.sender, 500);
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

    // 为特定Token设置特殊版税
    function setTokenRoyalty(
        uint256 tokenId,
        address receiver,
        uint96 feeNumerator
    ) external onlyOwner {
        _setTokenRoyalty(tokenId, receiver, feeNumerator);
    }
}
```

---

## Part 5: NFT元数据规范 (1小时)

### 5.1 JSON Schema

符合OpenSea标准的元数据结构：

```json
{
  "name": "Cool NFT #1",
  "description": "This is a very cool NFT.",
  "image": "ipfs://Qm...",
  "external_url": "https://myproject.com/nft/1",
  "attributes": [
    {
      "trait_type": "Background",
      "value": "Blue"
    },
    {
      "trait_type": "Eyes",
      "value": "Big"
    },
    {
      "trait_type": "Level",
      "value": 5
    },
    {
      "display_type": "boost_number",
      "trait_type": "Stamina",
      "value": 40
    },
    {
      "display_type": "date",
      "trait_type": "Birthday",
      "value": 1546360800
    }
  ]
}
```

### 5.2 动态元数据

链上SVG生成示例：

```solidity
import "@openzeppelin/contracts/utils/Base64.sol";

function tokenURI(uint256 tokenId) public view override returns (string memory) {
    string memory svg = string(abi.encodePacked(
        '<svg xmlns="http://www.w3.org/2000/svg" preserveAspectRatio="xMinYMin meet" viewBox="0 0 350 350">',
        '<style>.base { fill: white; font-family: serif; font-size: 14px; }</style>',
        '<rect width="100%" height="100%" fill="black" />',
        '<text x="50%" y="50%" class="base" dominant-baseline="middle" text-anchor="middle">',
        'Token #', tokenId.toString(),
        '</text>',
        '</svg>'
    ));

    string memory json = Base64.encode(bytes(string(abi.encodePacked(
        '{"name": "On-chain #', tokenId.toString(), '",',
        '"description": "Fully on-chain SVG NFT",',
        '"image": "data:image/svg+xml;base64,', Base64.encode(bytes(svg)), '"}'
    ))));

    return string(abi.encodePacked('data:application/json;base64,', json));
}
```

---

## 📝 今日作业

### 作业1: 开发ERC721A合约

开发一个具备以下功能的NFT合约：
1. 采用ERC721A标准
2. 支持Merkle Tree白名单
3. 包含公开铸造和白名单铸造阶段
4. 实现提款功能

### 作业2: 开发游戏道具系统

使用ERC1155开发：
1. 定义至少3种道具（金币、武器、药水）
2. 实现道具合成逻辑（如：100金币 + 1个材料 = 1把武器）
3. 实现批量转账功能

### 作业3: 生成元数据

编写脚本：
1. 生成100个NFT的图片和元数据
2. 上传到IPFS（使用Pinata或其他服务）
3. 生成Merkle Tree并导出Proof文件

---

## ✅ 检查清单

- [ ] 掌握ERC721 vs ERC1155的区别
- [ ] 理解ERC721A的省Gas原理
- [ ] 能够实现Merkle Tree白名单
- [ ] 理解EIP-2981版税标准
- [ ] 掌握NFT元数据结构

---

## 📅 明日预告

明天学习IPFS与去中心化存储：
- IPFS原理与架构
- Pinata API使用
- Arweave集成
- 前端上传与展示

**🎉 完成Day 1！NFT世界的大门已经打开！**
