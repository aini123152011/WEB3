# Week 2 - Day 5: ERC标准学习

**学习日期**: ___________
**预计用时**: 6-7小时  
**难度等级**: ⭐⭐⭐⭐ (进阶)

## 📚 今日学习目标

- ✅ 深入理解ERC20代币标准
- ✅ 掌握ERC721 NFT标准
- ✅ 学习ERC1155多代币标准
- ✅ 了解其他重要ERC标准
- ✅ 实现标准代币合约

---

## 💰 Part 1: ERC20代币标准 (2小时)

### 1.1 ERC20标准接口

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ERC20标准接口
 */
interface IERC20 {
    // 事件
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    // 查询函数
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function allowance(address owner, address spender) external view returns (uint256);
    
    // 交易函数
    function transfer(address to, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}
```

### 1.2 完整ERC20实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 标准ERC20代币实现
 */
contract MyToken is IERC20 {
    // 代币信息
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 private _totalSupply;
    
    // 余额和授权
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    
    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals,
        uint256 initialSupply
    ) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        
        _mint(msg.sender, initialSupply);
    }
    
    // 总供应量
    function totalSupply() public view override returns (uint256) {
        return _totalSupply;
    }
    
    // 查询余额
    function balanceOf(address account) public view override returns (uint256) {
        return _balances[account];
    }
    
    // 转账
    function transfer(address to, uint256 amount) public override returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }
    
    // 查询授权额度
    function allowance(address owner, address spender) 
        public 
        view 
        override 
        returns (uint256) 
    {
        return _allowances[owner][spender];
    }
    
    // 授权
    function approve(address spender, uint256 amount) public override returns (bool) {
        _approve(msg.sender, spender, amount);
        return true;
    }
    
    // 授权转账
    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public override returns (bool) {
        _spendAllowance(from, msg.sender, amount);
        _transfer(from, to, amount);
        return true;
    }
    
    // 内部转账函数
    function _transfer(
        address from,
        address to,
        uint256 amount
    ) internal {
        require(from != address(0), "Transfer from zero address");
        require(to != address(0), "Transfer to zero address");
        
        uint256 fromBalance = _balances[from];
        require(fromBalance >= amount, "Insufficient balance");
        
        unchecked {
            _balances[from] = fromBalance - amount;
            _balances[to] += amount;
        }
        
        emit Transfer(from, to, amount);
    }
    
    // 铸造
    function _mint(address account, uint256 amount) internal {
        require(account != address(0), "Mint to zero address");
        
        _totalSupply += amount;
        unchecked {
            _balances[account] += amount;
        }
        
        emit Transfer(address(0), account, amount);
    }
    
    // 销毁
    function _burn(address account, uint256 amount) internal {
        require(account != address(0), "Burn from zero address");
        
        uint256 accountBalance = _balances[account];
        require(accountBalance >= amount, "Burn amount exceeds balance");
        
        unchecked {
            _balances[account] = accountBalance - amount;
            _totalSupply -= amount;
        }
        
        emit Transfer(account, address(0), amount);
    }
    
    // 授权
    function _approve(
        address owner,
        address spender,
        uint256 amount
    ) internal {
        require(owner != address(0), "Approve from zero address");
        require(spender != address(0), "Approve to zero address");
        
        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }
    
    // 使用授权额度
    function _spendAllowance(
        address owner,
        address spender,
        uint256 amount
    ) internal {
        uint256 currentAllowance = allowance(owner, spender);
        if (currentAllowance != type(uint256).max) {
            require(currentAllowance >= amount, "Insufficient allowance");
            unchecked {
                _approve(owner, spender, currentAllowance - amount);
            }
        }
    }
}
```

### 1.3 扩展功能

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 带扩展功能的ERC20
 */
contract AdvancedToken is MyToken {
    address public owner;
    bool public paused;
    
    // 黑名单
    mapping(address => bool) public blacklist;
    
    // 交易税
    uint256 public taxRate = 100; // 1%
    address public taxReceiver;
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    modifier whenNotPaused() {
        require(!paused, "Contract is paused");
        _;
    }
    
    modifier notBlacklisted(address account) {
        require(!blacklist[account], "Address is blacklisted");
        _;
    }
    
    constructor(
        string memory name,
        string memory symbol,
        uint8 decimals,
        uint256 initialSupply
    ) MyToken(name, symbol, decimals, initialSupply) {
        owner = msg.sender;
        taxReceiver = msg.sender;
    }
    
    // 暂停/恢复
    function pause() external onlyOwner {
        paused = true;
    }
    
    function unpause() external onlyOwner {
        paused = false;
    }
    
    // 黑名单管理
    function addToBlacklist(address account) external onlyOwner {
        blacklist[account] = true;
    }
    
    function removeFromBlacklist(address account) external onlyOwner {
        blacklist[account] = false;
    }
    
    // 设置税率
    function setTaxRate(uint256 rate) external onlyOwner {
        require(rate <= 1000, "Tax rate too high"); // 最高10%
        taxRate = rate;
    }
    
    // 重写转账（添加税收）
    function _transfer(
        address from,
        address to,
        uint256 amount
    ) internal override whenNotPaused notBlacklisted(from) notBlacklisted(to) {
        if (taxRate > 0 && from != owner && to != owner) {
            uint256 tax = (amount * taxRate) / 10000;
            uint256 amountAfterTax = amount - tax;
            
            super._transfer(from, taxReceiver, tax);
            super._transfer(from, to, amountAfterTax);
        } else {
            super._transfer(from, to, amount);
        }
    }
    
    // 铸造
    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }
    
    // 销毁
    function burn(uint256 amount) external {
        _burn(msg.sender, amount);
    }
}
```

---

## 🖼️ Part 2: ERC721 NFT标准 (2小时)

### 2.1 ERC721标准接口

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * ERC721标准接口
 */
interface IERC721 {
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);
    
    function balanceOf(address owner) external view returns (uint256);
    function ownerOf(uint256 tokenId) external view returns (address);
    function safeTransferFrom(address from, address to, uint256 tokenId, bytes calldata data) external;
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function transferFrom(address from, address to, uint256 tokenId) external;
    function approve(address to, uint256 tokenId) external;
    function setApprovalForAll(address operator, bool approved) external;
    function getApproved(uint256 tokenId) external view returns (address);
    function isApprovedForAll(address owner, address operator) external view returns (bool);
}

interface IERC721Metadata {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function tokenURI(uint256 tokenId) external view returns (string memory);
}

interface IERC721Receiver {
    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4);
}
```

### 2.2 完整ERC721实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MyNFT is IERC721, IERC721Metadata {
    string public name;
    string public symbol;
    
    // Token数据
    uint256 private _nextTokenId;
    mapping(uint256 => address) private _owners;
    mapping(address => uint256) private _balances;
    mapping(uint256 => address) private _tokenApprovals;
    mapping(address => mapping(address => bool)) private _operatorApprovals;
    
    // Metadata
    mapping(uint256 => string) private _tokenURIs;
    string private _baseURI;
    
    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
    }
    
    // 余额查询
    function balanceOf(address owner) public view override returns (uint256) {
        require(owner != address(0), "Zero address");
        return _balances[owner];
    }
    
    // 所有者查询
    function ownerOf(uint256 tokenId) public view override returns (address) {
        address owner = _owners[tokenId];
        require(owner != address(0), "Token doesn't exist");
        return owner;
    }
    
    // 转账
    function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) public override {
        require(_isApprovedOrOwner(msg.sender, tokenId), "Not authorized");
        _transfer(from, to, tokenId);
    }
    
    // 安全转账
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId
    ) public override {
        safeTransferFrom(from, to, tokenId, "");
    }
    
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) public override {
        require(_isApprovedOrOwner(msg.sender, tokenId), "Not authorized");
        _safeTransfer(from, to, tokenId, data);
    }
    
    // 授权单个token
    function approve(address to, uint256 tokenId) public override {
        address owner = ownerOf(tokenId);
        require(to != owner, "Approval to current owner");
        require(
            msg.sender == owner || isApprovedForAll(owner, msg.sender),
            "Not authorized"
        );
        
        _approve(to, tokenId);
    }
    
    // 授权所有token
    function setApprovalForAll(address operator, bool approved) public override {
        require(operator != msg.sender, "Approve to caller");
        _operatorApprovals[msg.sender][operator] = approved;
        emit ApprovalForAll(msg.sender, operator, approved);
    }
    
    // 查询授权
    function getApproved(uint256 tokenId) public view override returns (address) {
        require(_exists(tokenId), "Token doesn't exist");
        return _tokenApprovals[tokenId];
    }
    
    function isApprovedForAll(address owner, address operator) 
        public 
        view 
        override 
        returns (bool) 
    {
        return _operatorApprovals[owner][operator];
    }
    
    // Token URI
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        require(_exists(tokenId), "Token doesn't exist");
        
        string memory _tokenURI = _tokenURIs[tokenId];
        
        if (bytes(_tokenURI).length > 0) {
            return _tokenURI;
        }
        
        return string(abi.encodePacked(_baseURI, _toString(tokenId)));
    }
    
    // 铸造
    function mint(address to, string memory uri) public returns (uint256) {
        uint256 tokenId = _nextTokenId++;
        _safeMint(to, tokenId);
        _setTokenURI(tokenId, uri);
        return tokenId;
    }
    
    // 内部函数
    function _exists(uint256 tokenId) internal view returns (bool) {
        return _owners[tokenId] != address(0);
    }
    
    function _isApprovedOrOwner(address spender, uint256 tokenId) 
        internal 
        view 
        returns (bool) 
    {
        address owner = ownerOf(tokenId);
        return (
            spender == owner || 
            isApprovedForAll(owner, spender) || 
            getApproved(tokenId) == spender
        );
    }
    
    function _safeMint(address to, uint256 tokenId) internal {
        _mint(to, tokenId);
        require(
            _checkOnERC721Received(address(0), to, tokenId, ""),
            "Transfer to non ERC721Receiver"
        );
    }
    
    function _mint(address to, uint256 tokenId) internal {
        require(to != address(0), "Mint to zero address");
        require(!_exists(tokenId), "Token already minted");
        
        _balances[to] += 1;
        _owners[tokenId] = to;
        
        emit Transfer(address(0), to, tokenId);
    }
    
    function _burn(uint256 tokenId) internal {
        address owner = ownerOf(tokenId);
        
        _approve(address(0), tokenId);
        
        _balances[owner] -= 1;
        delete _owners[tokenId];
        
        if (bytes(_tokenURIs[tokenId]).length != 0) {
            delete _tokenURIs[tokenId];
        }
        
        emit Transfer(owner, address(0), tokenId);
    }
    
    function _transfer(
        address from,
        address to,
        uint256 tokenId
    ) internal {
        require(ownerOf(tokenId) == from, "Transfer from incorrect owner");
        require(to != address(0), "Transfer to zero address");
        
        _approve(address(0), tokenId);
        
        _balances[from] -= 1;
        _balances[to] += 1;
        _owners[tokenId] = to;
        
        emit Transfer(from, to, tokenId);
    }
    
    function _safeTransfer(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) internal {
        _transfer(from, to, tokenId);
        require(
            _checkOnERC721Received(from, to, tokenId, data),
            "Transfer to non ERC721Receiver"
        );
    }
    
    function _approve(address to, uint256 tokenId) internal {
        _tokenApprovals[tokenId] = to;
        emit Approval(ownerOf(tokenId), to, tokenId);
    }
    
    function _setTokenURI(uint256 tokenId, string memory uri) internal {
        require(_exists(tokenId), "Token doesn't exist");
        _tokenURIs[tokenId] = uri;
    }
    
    function _checkOnERC721Received(
        address from,
        address to,
        uint256 tokenId,
        bytes memory data
    ) private returns (bool) {
        if (to.code.length > 0) {
            try IERC721Receiver(to).onERC721Received(msg.sender, from, tokenId, data) 
                returns (bytes4 retval) {
                return retval == IERC721Receiver.onERC721Received.selector;
            } catch {
                return false;
            }
        }
        return true;
    }
    
    function _toString(uint256 value) internal pure returns (string memory) {
        if (value == 0) return "0";
        
        uint256 temp = value;
        uint256 digits;
        while (temp != 0) {
            digits++;
            temp /= 10;
        }
        
        bytes memory buffer = new bytes(digits);
        while (value != 0) {
            digits -= 1;
            buffer[digits] = bytes1(uint8(48 + uint256(value % 10)));
            value /= 10;
        }
        
        return string(buffer);
    }
}
```

---

## 🎯 Part 3: ERC1155多代币标准 (1.5小时)

### 3.1 ERC1155标准接口

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IERC1155 {
    event TransferSingle(
        address indexed operator,
        address indexed from,
        address indexed to,
        uint256 id,
        uint256 value
    );
    
    event TransferBatch(
        address indexed operator,
        address indexed from,
        address indexed to,
        uint256[] ids,
        uint256[] values
    );
    
    event ApprovalForAll(address indexed account, address indexed operator, bool approved);
    event URI(string value, uint256 indexed id);
    
    function balanceOf(address account, uint256 id) external view returns (uint256);
    function balanceOfBatch(
        address[] calldata accounts,
        uint256[] calldata ids
    ) external view returns (uint256[] memory);
    
    function setApprovalForAll(address operator, bool approved) external;
    function isApprovedForAll(address account, address operator) external view returns (bool);
    
    function safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes calldata data
    ) external;
    
    function safeBatchTransferFrom(
        address from,
        address to,
        uint256[] calldata ids,
        uint256[] calldata amounts,
        bytes calldata data
    ) external;
}
```

### 3.2 ERC1155实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MyMultiToken is IERC1155 {
    // 余额: account => id => amount
    mapping(address => mapping(uint256 => uint256)) private _balances;
    
    // 授权
    mapping(address => mapping(address => bool)) private _operatorApprovals;
    
    // URI
    mapping(uint256 => string) private _uris;
    
    // 查询余额
    function balanceOf(address account, uint256 id) 
        public 
        view 
        override 
        returns (uint256) 
    {
        require(account != address(0), "Zero address");
        return _balances[account][id];
    }
    
    // 批量查询余额
    function balanceOfBatch(
        address[] memory accounts,
        uint256[] memory ids
    ) public view override returns (uint256[] memory) {
        require(accounts.length == ids.length, "Length mismatch");
        
        uint256[] memory batchBalances = new uint256[](accounts.length);
        
        for (uint256 i = 0; i < accounts.length; i++) {
            batchBalances[i] = balanceOf(accounts[i], ids[i]);
        }
        
        return batchBalances;
    }
    
    // 授权
    function setApprovalForAll(address operator, bool approved) public override {
        require(msg.sender != operator, "Setting approval for self");
        _operatorApprovals[msg.sender][operator] = approved;
        emit ApprovalForAll(msg.sender, operator, approved);
    }
    
    function isApprovedForAll(address account, address operator) 
        public 
        view 
        override 
        returns (bool) 
    {
        return _operatorApprovals[account][operator];
    }
    
    // 转账
    function safeTransferFrom(
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) public override {
        require(
            from == msg.sender || isApprovedForAll(from, msg.sender),
            "Not authorized"
        );
        require(to != address(0), "Transfer to zero address");
        
        uint256 fromBalance = _balances[from][id];
        require(fromBalance >= amount, "Insufficient balance");
        
        unchecked {
            _balances[from][id] = fromBalance - amount;
        }
        _balances[to][id] += amount;
        
        emit TransferSingle(msg.sender, from, to, id, amount);
        
        _doSafeTransferAcceptanceCheck(msg.sender, from, to, id, amount, data);
    }
    
    // 批量转账
    function safeBatchTransferFrom(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) public override {
        require(
            from == msg.sender || isApprovedForAll(from, msg.sender),
            "Not authorized"
        );
        require(ids.length == amounts.length, "Length mismatch");
        require(to != address(0), "Transfer to zero address");
        
        for (uint256 i = 0; i < ids.length; i++) {
            uint256 id = ids[i];
            uint256 amount = amounts[i];
            
            uint256 fromBalance = _balances[from][id];
            require(fromBalance >= amount, "Insufficient balance");
            
            unchecked {
                _balances[from][id] = fromBalance - amount;
            }
            _balances[to][id] += amount;
        }
        
        emit TransferBatch(msg.sender, from, to, ids, amounts);
        
        _doSafeBatchTransferAcceptanceCheck(msg.sender, from, to, ids, amounts, data);
    }
    
    // 铸造
    function mint(
        address to,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) public {
        require(to != address(0), "Mint to zero address");
        
        _balances[to][id] += amount;
        emit TransferSingle(msg.sender, address(0), to, id, amount);
        
        _doSafeTransferAcceptanceCheck(msg.sender, address(0), to, id, amount, data);
    }
    
    // 批量铸造
    function mintBatch(
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) public {
        require(to != address(0), "Mint to zero address");
        require(ids.length == amounts.length, "Length mismatch");
        
        for (uint256 i = 0; i < ids.length; i++) {
            _balances[to][ids[i]] += amounts[i];
        }
        
        emit TransferBatch(msg.sender, address(0), to, ids, amounts);
        
        _doSafeBatchTransferAcceptanceCheck(msg.sender, address(0), to, ids, amounts, data);
    }
    
    // URI
    function uri(uint256 id) public view returns (string memory) {
        return _uris[id];
    }
    
    function _setURI(uint256 id, string memory newuri) internal {
        _uris[id] = newuri;
        emit URI(newuri, id);
    }
    
    // 安全检查（简化版）
    function _doSafeTransferAcceptanceCheck(
        address operator,
        address from,
        address to,
        uint256 id,
        uint256 amount,
        bytes memory data
    ) private {
        if (to.code.length > 0) {
            // 如果是合约，需要实现接收接口
            // 这里简化处理
        }
    }
    
    function _doSafeBatchTransferAcceptanceCheck(
        address operator,
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) private {
        if (to.code.length > 0) {
            // 如果是合约，需要实现接收接口
            // 这里简化处理
        }
    }
}
```

---

## 📋 Part 4: 其他重要ERC标准 (0.5小时)

### 4.1 ERC165 - 接口检测

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IERC165 {
    function supportsInterface(bytes4 interfaceId) external view returns (bool);
}

contract ERC165 is IERC165 {
    mapping(bytes4 => bool) private _supportedInterfaces;
    
    constructor() {
        _registerInterface(type(IERC165).interfaceId);
    }
    
    function supportsInterface(bytes4 interfaceId) 
        public 
        view 
        override 
        returns (bool) 
    {
        return _supportedInterfaces[interfaceId];
    }
    
    function _registerInterface(bytes4 interfaceId) internal {
        require(interfaceId != 0xffffffff, "Invalid interface id");
        _supportedInterfaces[interfaceId] = true;
    }
}
```

### 4.2 ERC2981 - NFT版税标准

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IERC2981 {
    function royaltyInfo(uint256 tokenId, uint256 salePrice)
        external
        view
        returns (address receiver, uint256 royaltyAmount);
}

contract NFTWithRoyalty is MyNFT, IERC2981 {
    struct RoyaltyInfo {
        address receiver;
        uint96 royaltyFraction; // 使用基点（1/10000）
    }
    
    RoyaltyInfo private _defaultRoyaltyInfo;
    mapping(uint256 => RoyaltyInfo) private _tokenRoyaltyInfo;
    
    constructor(
        string memory name,
        string memory symbol
    ) MyNFT(name, symbol) {}
    
    // 设置默认版税
    function setDefaultRoyalty(address receiver, uint96 feeNumerator) public {
        require(feeNumerator <= 10000, "Royalty too high");
        _defaultRoyaltyInfo = RoyaltyInfo(receiver, feeNumerator);
    }
    
    // 设置特定token版税
    function setTokenRoyalty(
        uint256 tokenId,
        address receiver,
        uint96 feeNumerator
    ) public {
        require(feeNumerator <= 10000, "Royalty too high");
        _tokenRoyaltyInfo[tokenId] = RoyaltyInfo(receiver, feeNumerator);
    }
    
    // 查询版税信息
    function royaltyInfo(uint256 tokenId, uint256 salePrice)
        public
        view
        override
        returns (address, uint256)
    {
        RoyaltyInfo memory royalty = _tokenRoyaltyInfo[tokenId];
        
        if (royalty.receiver == address(0)) {
            royalty = _defaultRoyaltyInfo;
        }
        
        uint256 royaltyAmount = (salePrice * royalty.royaltyFraction) / 10000;
        
        return (royalty.receiver, royaltyAmount);
    }
}
```

---

## 🧪 Part 5: 实战练习 (1小时)

### 5.1 实现游戏道具系统

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * 游戏道具系统
 * 使用ERC1155实现不同类型的道具
 */
contract GameItems is MyMultiToken {
    uint256 public constant SWORD = 0;
    uint256 public constant SHIELD = 1;
    uint256 public constant POTION = 2;
    
    address public gameContract;
    
    constructor() {
        gameContract = msg.sender;
    }
    
    modifier onlyGame() {
        require(msg.sender == gameContract, "Only game contract");
        _;
    }
    
    // 游戏奖励道具
    function reward(address player, uint256 itemId, uint256 amount) 
        external 
        onlyGame 
    {
        mint(player, itemId, amount, "");
    }
    
    // 使用道具
    function use(uint256 itemId, uint256 amount) external {
        require(balanceOf(msg.sender, itemId) >= amount, "Insufficient items");
        _burn(msg.sender, itemId, amount);
    }
    
    function _burn(address account, uint256 id, uint256 amount) private {
        uint256 accountBalance = _balances[account][id];
        require(accountBalance >= amount, "Burn amount exceeds balance");
        
        unchecked {
            _balances[account][id] = accountBalance - amount;
        }
        
        emit TransferSingle(msg.sender, account, address(0), id, amount);
    }
}
```

---

## 📝 今日作业

### 作业1: 完整的代币系统

实现一个包含以下功能的ERC20代币：
1. 基础ERC20功能
2. 暂停/恢复交易
3. 黑名单管理
4. 交易手续费（可配置）
5. 分红功能（持币者按比例分红）
6. 时间锁定（锁定期内不能转账）
7. 批量转账

### 作业2: NFT市场

实现一个NFT交易市场：
1. 基于ERC721的NFT合约
2. 支持固定价格出售
3. 支持拍卖（英式拍卖）
4. 版税支持（ERC2981）
5. 交易历史记录
6. 批量操作

### 作业3: 游戏资产系统

使用ERC1155实现游戏资产系统：
1. 多种类型的游戏道具
2. 道具合成（3个低级→1个高级）
3. 道具交易市场
4. 道具租赁系统
5. 成就NFT（不可交易）

---

## ✅ 今日检查清单

- [ ] 理解ERC20标准和实现
- [ ] 掌握ERC721 NFT标准
- [ ] 学习ERC1155多代币标准
- [ ] 了解ERC165和ERC2981
- [ ] 完成所有作业

---

## 🆘 常见问题FAQ

### Q1: ERC20的approve有什么安全问题？

A: approve存在前后端竞态问题。应该使用increaseAllowance和decreaseAllowance：

```solidity
function increaseAllowance(address spender, uint256 addedValue) public returns (bool) {
    _approve(msg.sender, spender, allowance(msg.sender, spender) + addedValue);
    return true;
}

function decreaseAllowance(address spender, uint256 subtractedValue) public returns (bool) {
    uint256 currentAllowance = allowance(msg.sender, spender);
    require(currentAllowance >= subtractedValue, "Decreased below zero");
    _approve(msg.sender, spender, currentAllowance - subtractedValue);
    return true;
}
```

### Q2: safeTransferFrom和transferFrom的区别？

A: safeTransferFrom会检查接收方是否实现了接收接口，防止NFT被锁死在合约中。

### Q3: ERC1155相比ERC721的优势？

A: 
- 单个合约管理多种代币
- 批量操作更省gas
- 适合游戏道具等场景
- 同时支持同质化和非同质化代币

---

## 📚 扩展阅读

- [EIP-20: Token Standard](https://eips.ethereum.org/EIPS/eip-20)
- [EIP-721: Non-Fungible Token Standard](https://eips.ethereum.org/EIPS/eip-721)
- [EIP-1155: Multi Token Standard](https://eips.ethereum.org/EIPS/eip-1155)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)

---

## 📅 学习记录

**今日完成情况**:
- [ ] 学习了ERC20标准
- [ ] 学习了ERC721标准
- [ ] 学习了ERC1155标准
- [ ] 完成了作业

**遇到的问题**:
___________________________________________

**解决方案**:
___________________________________________

**明日计划**:
完成周末综合项目 - 去中心化众筹平台

**🎉 完成Day 5！明天开始周末项目！**
